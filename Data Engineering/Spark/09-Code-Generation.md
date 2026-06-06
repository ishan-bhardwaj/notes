# Code Generation

## Volcano Iterator Model vs Codegen

- __Volcano Iterator Model__ - the classical database execution model; every operator implements a `next()` method that pulls one row at a time from its child operator; used by Spark before Tungsten and still used as fallback
- __Problem with Volcano__ - virtual dispatch on every `next()` call; one row processed per call across entire operator tree; no JIT optimization across operator boundaries; boxing/unboxing of primitive types

### Volcano Model Cost

- For a pipeline `Scan → Filter → Project → Aggregate` processing $100M$ rows -
    - $100M$ calls to `Aggregate.next()` → $100M$ calls to `Project.next()` → $100M$ calls to `Filter.next()` → $100M$ calls to `Scan.next()`
    - Each call is a virtual dispatch (JVM `invokeinterface`) - not inlined by JIT across operator boundaries
    - Each row crosses operator boundaries as `InternalRow` object - heap allocation, GC pressure
    - CPU branch mispredictions from type checking at every operator boundary

### Codegen Model

- __Whole-Stage CodeGen (WSCG)__ - fuses a pipeline of operators into a single generated Java class with a tight inner loop
- Instead of each operator pulling rows from its child, the generated code processes one row through the entire pipeline in a single function call
- Generated code -
    - No virtual dispatch - all operator logic inlined into one method
    - Primitives stay in CPU registers - no boxing; `int` stays `int` not `Integer`
    - JIT can optimize the entire pipeline as one compilation unit - loop unrolling, SIMD, dead code elimination
    - No intermediate row objects between operators - values passed as local variables

### Performance Difference

- Volcano model - ~$10-50$ ns per row per operator (virtual dispatch + memory access)
- WSCG generated code - ~$1-5$ ns per row for the entire pipeline (tight loop, CPU cache-friendly)
- Real-world - $2-10×$ speedup on CPU-bound operations (aggregation, sorting, expression evaluation)

### Conceptual Comparison

- Volcano -
```java
    // Interpreted - virtual dispatch chain per row
    while (aggregate.hasNext()) {
        Row row = aggregate.next();         // calls project.next() which calls filter.next()...
        result.add(row);
    }
```

- Generated code -
```java
    // Generated - single tight loop, all operators fused
    while (scan.hasNext()) {
        long value_0 = scan.nextLong();     // primitive, no boxing
        if (value_0 > 100L) {               // filter inlined
            long projected = value_0 * 2L;  // project inlined
            agg_sum += projected;           // aggregate inlined
        }
    }
```

---

## Whole-Stage CodeGen Overview

- __Whole-Stage CodeGen (WSCG)__ - Spark's code generation framework; fuses chains of physical operators that support codegen into a single generated Java class per "codegen stage"
- Enabled by `spark.sql.codegen.wholeStage=true` (default `true`)
- Introduced in Spark 2.0 as part of Project Tungsten second phase

### CodeGen Stage Boundaries

- A WSCG stage is a maximal chain of operators that all implement `CodegenSupport`
- Stage boundaries occur at -
    - Operators that don't implement `CodegenSupport` (UDFs, some aggregate types)
    - `InputAdapter` - wraps operators that don't support codegen; bridges into the codegen stage
    - Exchange operators (`ShuffleExchangeExec`, `BroadcastExchangeExec`) - always stage boundaries
    - `SortExec` - implements codegen but produces a stage boundary for the sort itself
- Each WSCG stage becomes one `WholeStageCodegenExec` node in the physical plan
- Each `WholeStageCodegenExec` generates one Java class compiled by Janino

### Push-Based Execution Model

- WSCG uses push-based (producer-driven) model instead of Volcano's pull-based model -
    - Leaf operator (scan) drives the loop
    - Each row is "pushed" up through the operator chain via `consume` calls
    - Parent operators add their logic to the `consume` method of the child
- This enables the entire pipeline to be one inlined loop in the generated class

### Example Generated Class Structure

```java
// WholeStageCodegenExec generates something like this
public class GeneratedIteratorForCodegenStage1 extends BufferedRowIterator {
    private Object[] references;        // constants, partition data
    private long agg_sum;               // aggregation state
    private scala.collection.Iterator scan_input;

    public void init(int index, scala.collection.Iterator[] inputs) {
        scan_input = inputs[0];
        agg_sum = 0L;
    }

    protected void processNext() throws java.io.IOException {
        while (scan_input.hasNext()) {
            InternalRow scan_row = (InternalRow) scan_input.next();
            long scan_value = scan_row.getLong(0);      // column access
            if (!(scan_value > 100L)) continue;         // filter inlined
            long project_result = scan_value * 2L;      // project inlined
            agg_sum += project_result;                  // aggregate inlined
        }
        // emit result row
        rowWriter.write(0, agg_sum);
        append(resultRow);
    }
}
```

---

## CodegenSupport Interface

- __`CodegenSupport`__ - the trait that physical operators implement to participate in WSCG; defines the code generation contract between operators

### Core Methods

- `produce(ctx: CodegenContext, parent: CodegenSupport): String` -
    - Called on a child operator by its parent to get the code that produces rows
    - Calls `doProduce(ctx)` internally
    - Returns Java code string that when executed, calls `parent.consume(ctx, ...)` for each row
- `consume(ctx: CodegenContext, input: Seq[ExprCode], row: ExprCode): String` -
    - Called by a child to get the code that the parent wants executed for each row
    - Calls `doConsume(ctx, input, row)` internally
    - Returns Java code string for what the parent does with each row
- `doProduce(ctx: CodegenContext): String` - operator-specific production code; must be overridden
- `doConsume(ctx: CodegenContext, input: Seq[ExprCode], row: ExprCode): String` - operator-specific consumption code; must be overridden

### ExprCode

- __`ExprCode`__ - holds the generated code for one expression value -
    - `code: Block` - Java code that must be executed before reading the value
    - `value: ExprValue` - Java variable name holding the computed value
    - `isNull: ExprValue` - Java variable name holding the null flag (`boolean`)
- Example for `col("price") * lit(1.1)` -
    ```
    ExprCode(
    code    = "double price_val = row.getDouble(2);",
    value   = "price_result",     // variable holding price * 1.1
    isNull  = "price_isNull"      // boolean variable
    )
    ```

### CodegenContext

- __`CodegenContext`__ - accumulator for generated code; shared across all operators in a WSCG stage -
    - `addMutableState(javaType, variableName, initCode)` - declares class-level mutable state (eg - aggregation accumulators, loop counters)
    - `addImmutableStateIfNotExists(javaType, name, initCode)` - shared constants
    - `freshName(prefix)` - generates unique variable name to avoid collisions between operators
    - `addReferenceObj(name, obj)` - adds Java object to `references` array; accessed in generated code via `references[i]`
    - `addNewFunction(funcName, code)` - adds a helper method to the generated class; used when inlined code would be too large
    - `INPUT_ROW` - variable name of the current input row in generated code

### Producer-Consumer Code Flow

- `WholeStageCodegenExec.doCodeGen()`
    - calls `child.produce(ctx, this)`, `child` = last operator in chain (eg, `HashAggregateExec`)
    - `HashAggregateExec.doProduce(ctx)`
    - calls `child.produce(ctx, this)`, `child` = `FilterExec`
    - `FilterExec.doProduce(ctx)`
    - calls `child.produce(ctx, this)`, `child` = `InputAdapter(FileScan)`
    - `InputAdapter` generates scan loop
    - for each row, calls `FilterExec.consume(ctx, input, row)`
    - `FilterExec.doConsume` adds - `if (condition) { parent.consume(ctx, input, row) }`
    - `HashAggregateExec.doConsume` adds - `agg_sum += value`

---

## InputAdapter & WholeStageCodegenExec

### WholeStageCodegenExec

- __`WholeStageCodegenExec`__ - the physical operator that wraps a codegen stage; generates, compiles, and executes the fused Java class
- Added by `CollapseCodegenStages` preparation rule
- Identified by `*` prefix in `df.explain()` output -
    ```
    *(1) HashAggregate(...)
    +- *(1) Filter (price > 100)
    +- *(1) FileScan parquet [...]
    ```

- `*(1)` means codegen stage 1; all operators with same stage number fused into one class

### WholeStageCodegenExec Execution

- `doExecute()` -
    1. Calls `doCodeGen()` → generates Java source code string
    2. Calls `CodeGenerator.compile(code)` → Janino compiles to `GeneratedClass`
    3. For each partition, instantiates the generated class and calls `processNext()`
    4. Returns `RDD[InternalRow]` where each partition processed by generated class instance

### InputAdapter

- __`InputAdapter`__ - wraps a non-codegen operator (or a stage boundary) to bridge it into a WSCG stage
- Implements `CodegenSupport` but its `doProduce` generates a Volcano-style `while (iter.hasNext())` loop rather than fused code
- Appears in explain output without `*` prefix -
    ```
    *(2) SortMergeJoin [id]
    :- *(1) Sort [id ASC]
    :  +- Exchange hashpartitioning(id, 200)    ← no codegen prefix; stage boundary
    :     +- *(1) Filter (age > 18)
    :        +- *(1) FileScan parquet [...]
    +- *(2) Sort [id ASC]
        +- Exchange hashpartitioning(id, 200)
            +- *(2) Filter (salary > 50000)
                +- *(2) FileScan parquet [...]
    ```

- `Exchange` has no `*` - it is a stage boundary; `InputAdapter` wraps it for the upstream codegen stage

### Stage Numbering

- Each `WholeStageCodegenExec` has a `codegenStageId`
- Operators with the same stage ID are fused into one generated class
- Different stage IDs = different generated classes; communication between stages via `RDD` (materialized rows)

---

## Expression CodeGen

- __Expression CodeGen__ - the mechanism by which individual Catalyst `Expression` nodes generate Java code fragments; the building block of operator-level codegen

### Expression.genCode

- `Expression.genCode(ctx: CodegenContext): ExprCode` - generates Java code to evaluate the expression
- Returns `ExprCode(code, value, isNull)` -
    - `code` - setup code that computes the value
    - `value` - variable name holding the result
    - `isNull` - variable name holding the null flag

### Generated Code Examples

- `Literal(42L, LongType)` -
```scala
    ExprCode(
        code  = "",                 // no setup code needed
        value = "42L",              // literal directly inlined
        isNull = "false"
    )
```

- `AttributeReference("price", DoubleType)` at ordinal 2 -
```scala
    ExprCode(
        code  = "double price_value_1 = row.getDouble(2);\nboolean price_isNull_1 = row.isNullAt(2);",
        value = "price_value_1",
        isNull = "price_isNull_1"
    )
```

- `Add(left, right)` -
```scala
    // Generates code for left, code for right, then addition
    val leftCode = left.genCode(ctx)
    val rightCode = right.genCode(ctx)
    val resultVar = ctx.freshName("addResult")
    ExprCode(
        code  = s"${leftCode.code}\n${rightCode.code}\n" +
                s"double $resultVar = ${leftCode.value} + ${rightCode.value};",
        value = resultVar,
        isNull = s"(${leftCode.isNull} || ${rightCode.isNull})"
    )
```

- `Cast(child, LongType)` -
```scala
    ExprCode(
        code  = s"${childCode.code}\nlong cast_result = (long)(${childCode.value});",
        value = "cast_result",
        isNull = childCode.isNull
    )
```

### Null Handling in Generated Code

- Every expression tracks nullability
- Null propagation - `null + expr → null` implemented as short-circuit in generated code
- `Expression.nullable` flag used to elide null checks when compiler can prove non-null

### Non-Foldable vs Foldable

- `foldable = true` - expression can be evaluated at planning time (only `Literal`s and deterministic functions of literals)
- Foldable expressions fully inlined as constants in generated code
- Non-foldable expressions (column references, UDFs) generate runtime evaluation code

---

## Project & Filter CodeGen

### FilterExec CodeGen

- `FilterExec.doProduce` - just calls `child.produce(ctx, this)`
- `FilterExec.doConsume` - generates an `if` statement wrapping the parent's `consume` call -

- Scala (internal) -
```scala
    override def doConsume(ctx: CodegenContext, input: Seq[ExprCode], row: ExprCode): String = {
        val conditionCode = condition.genCode(ctx)
        s"""
           |${conditionCode.code}
           |if (${conditionCode.isNull} || !${conditionCode.value}) continue;
           |${consume(ctx, input)}
         """.stripMargin
    }
```

- Generated Java (example for `price > 100`) -
```java
    // Inside the scan loop - FilterExec inlined
    double price_value = row.getDouble(2);
    boolean price_isNull = row.isNullAt(2);
    if (price_isNull || !(price_value > 100.0)) continue;
    // parent consume code follows
```

### ProjectExec CodeGen

- `ProjectExec.doProduce` - just calls `child.produce(ctx, this)`
- `ProjectExec.doConsume` - generates code for each projection expression; passes results to parent -

- Scala (internal) -
```scala
    override def doConsume(ctx: CodegenContext, input: Seq[ExprCode], row: ExprCode): String = {
        val projectionCodes = projectList.map(_.genCode(ctx))
        s"""
           |${projectionCodes.map(_.code).mkString("\n")}
           |${consume(ctx, projectionCodes)}
         """.stripMargin
    }
```

- Generated Java (example for `SELECT price * 1.1 AS adjusted`) -
```java
    // After filter check - ProjectExec inlined
    double adjusted_value = price_value * 1.1;
    boolean adjusted_isNull = price_isNull;
    // parent consume called with adjusted_value, adjusted_isNull
```

### HashAggregateExec CodeGen

- More complex - two phases, each generating separate code -
- Partial aggregate (map side) -
```java
    // Inside scan loop
    long hashCode = computeHash(key_value);
    int bucketIdx = findOrInsert(hashMap, hashCode, key_value);
    if (bucketIdx >= 0) {
        aggBuffer[bucketIdx] += value;    // update in-place, no allocation
    } else {
        spill();    // fall back to sort-based aggregate
    }
```
- Final aggregate (reduce side) - iterates hash map, emits one row per group

---

## Generated Java Bytecode

- WSCG generates Java source code as a `String`; compiled to JVM bytecode by Janino; loaded as a class

### Generated Class Structure

```java
public class GeneratedIteratorForCodegenStage1
        extends org.apache.spark.sql.execution.BufferedRowIterator {

    private Object[] references;            // constants and external objects
    private scala.collection.Iterator[] inputs;

    // Mutable state - aggregation accumulators, loop variables
    private long agg_sum_0;
    private boolean agg_sum_isNull_0;

    // UnsafeRow writers for output
    private org.apache.spark.sql.catalyst.expressions.codegen.UnsafeRowWriter rowWriter;

    public GeneratedIteratorForCodegenStage1(Object[] references) {
        this.references = references;
    }

    public void init(int index, scala.collection.Iterator[] inputs) {
        partitionIndex = index;
        this.inputs = inputs;
        agg_sum_0 = 0L;
        agg_sum_isNull_0 = true;
        rowWriter = new UnsafeRowWriter(1);
    }

    protected void processNext() throws java.io.IOException {
        // Fused scan + filter + project + aggregate
        scala.collection.Iterator scan_input = inputs[0];
        while (scan_input.hasNext()) {
            InternalRow scan_row = (InternalRow) scan_input.next();

            // Filter inlined
            long price_value = scan_row.getLong(2);
            if (price_value <= 100L) continue;

            // Project inlined
            long adjusted = price_value * 2L;

            // Aggregate inlined
            if (agg_sum_isNull_0) {
                agg_sum_0 = adjusted;
                agg_sum_isNull_0 = false;
            } else {
                agg_sum_0 += adjusted;
            }
        }

        // Emit result
        rowWriter.reset();
        if (!agg_sum_isNull_0) rowWriter.write(0, agg_sum_0);
        append(rowWriter.getRow());
    }
}
```

### Method Size Limit

- JVM enforces $64 KB$ limit on compiled method bytecode size
- Complex queries with many expressions can exceed this limit in `processNext()`
- Catalyst detects this - splits logic into helper methods via `ctx.addNewFunction(name, code)`
- Helper methods called from `processNext()` - slight overhead vs fully inlined but still far better than Volcano

### Class Loading

- Each generated class loaded via `GeneratedClass` - a custom `ClassLoader`
- Class eviction - Spark maintains an LRU cache of compiled classes; `spark.sql.codegen.cache.maxEntries` (default $100$)
- Cache hit - same query plan shape reuses previously compiled class; avoids Janino compilation overhead

---

## Janino Compiler Integration

- __Janino__ - an in-process Java compiler that compiles Java source to JVM bytecode without `javac` subprocess
- Used by Spark exclusively for WSCG - compiles generated Java source strings to `Class` objects at runtime
- `org.codehaus.janino.ClassBodyEvaluator` / `SimpleCompiler` - the Janino API Spark uses

### Why Janino Not javac

- __In-process__ - no subprocess spawn; no filesystem temp files; compiles in milliseconds not seconds
- __Lightweight__ - Janino is a small library (~$1 MB$); includes only what's needed for runtime compilation
- __No full JDK required__ - Janino works with JRE only; `javac` requires full JDK installation
- __Fast__ - typical WSCG class compiles in $<10 ms$; `javac` overhead for same class would be $>100 ms$

### Compilation Flow

- `CodeGenerator.compile(code: CodeAndComment): (GeneratedClass, Int)` -
    1. Check `CodeGenerator.cache` (LRU cache) - if same code string seen before, return cached class
    2. Call `CodeGenerator.doCompile(code)` -
        - Create `SimpleCompiler` instance
        - Set parent `ClassLoader` to Spark's class loader (so generated code can reference Spark classes)
        - Call `compiler.cook(generatedSourceCode)` - compiles String → bytecode
        - Returns `Class[GeneratedIterator]`
    3. Cache compiled class; return

### Compilation Errors

- `org.codehaus.janino.CompileException` thrown if generated code has syntax errors
- Should not happen in production - Catalyst generates correct code
- Can happen with custom extensions injecting malformed expression codegen
- `spark.sql.codegen.comments=true` - include original SQL expressions as comments in generated code; helps debug compilation errors

### Janino Limitations

- Does not support all Java language features - notably -
    - No generics in generated code (type erasure used instead)
    - No lambda expressions in generated code (anonymous classes used instead)
    - Limited `java.lang.invoke` support
- Spark's code generator is designed around these limitations

---

## CodeGen Fallbacks

- WSCG is not universally applicable - certain conditions cause fallback to interpreted (Volcano) mode

### Fallback Triggers

- __Schema too wide__ -
    - `spark.sql.codegen.maxFields` (default $100$) - if input or output schema has more fields than this threshold, WSCG disabled for that stage
    - Wide schemas generate code with too many variable declarations → method size limit exceeded → JVM rejects class
    - Fix - reduce selected columns; increase `spark.sql.codegen.maxFields` cautiously

- __UDFs (Scala/Java)__ -
    - `ScalaUDF` and `JavaUDF` do not implement `CodegenSupport`
    - Any operator containing a UDF cannot be fused into WSCG
    - Stage boundary splits at the UDF; operators above and below UDF are separate WSCG stages
    - The UDF itself runs interpreted, row-by-row
    - Fix - replace UDF with built-in `functions.*` equivalent; use `pandas_udf` for Python

- __Python UDFs__ -
    - `PythonUDF` - serializes rows to Python process via Arrow/pickle; full stage boundary
    - Even worse than Scala UDFs - inter-process serialization overhead per row
    - `pandas_udf` - batches rows into Arrow `ColumnarBatch`; much faster than row-level Python UDFs
    - Neither Python UDF type generates JVM bytecode

- __Unsupported expression types__ -
    - Some expressions don't implement `genCode` - fall back to `eval()` interpreted path
    - `CodegenFallback` trait - expressions mix this in to indicate they cannot generate code
    - Any parent expression of a `CodegenFallback` also cannot fully codegen

- __Method size limit__ -
    - Generated `processNext()` method exceeds $64 KB$ bytecode limit
    - `Catalyst` splits into helper methods; if splitting impossible, falls back to interpreted
    - `spark.sql.codegen.fallback=true` (default `true`) - if compilation fails, fall back silently instead of failing the query

- __Fallback detection__ -
```python
    # Explain shows operators WITHOUT * prefix = not in WSCG
    df.explain()
    # *(1) means codegen stage 1 - good
    # No * prefix means interpreted - investigate why

    # Check for UDFs in plan
    df.queryExecution.executedPlan.foreach {
        case p if p.toString.contains("ScalaUDF") => println(s"UDF found: $p")
        case _ =>
    }
```

### Fallback Performance Impact

- Interpreted operators process one row at a time with virtual dispatch
- $10-50×$ slower than generated code for CPU-bound operations
- Shuffle, I/O-bound operations - fallback less impactful (bottleneck is not CPU)
- Identify fallbacks via Spark UI - SQL tab shows codegen stage boundaries; operators outside WSCG stages are interpreted

### pandas_udf as Partial Mitigation

- `pandas_udf` (vectorized UDF) - operates on `pd.Series` (Arrow batch) not individual rows
- Does not participate in WSCG - still a stage boundary
- But - batch processing eliminates per-row serialization overhead; typically $10-100×$ faster than row-level Python UDFs
- Python -
```python
    from pyspark.sql.functions import pandas_udf
    import pandas as pd

    @pandas_udf("double")
    def my_udf(s: pd.Series) -> pd.Series:
        return s * 2.0      # vectorized - no Python loop

    df.select(my_udf(col("price")))
```

### Diagnosing CodeGen Issues

- Python -
```python
    # See generated Java code
    df.explain("codegen")

    # Disable WSCG globally (for debugging - never in production)
    spark.conf.set("spark.sql.codegen.wholeStage", False)

    # Enable compilation error details
    spark.conf.set("spark.sql.codegen.comments", True)

    # Log every rule application
    spark.conf.set("spark.sql.planChangeLog.level", "TRACE")

    # Check codegen stage boundaries
    df.queryExecution.executedPlan
    # Operators wrapped in WholeStageCodegenExec = codegen
    # Operators NOT wrapped = interpreted fallback
```

> [!NOTE]
> The most common WSCG fallback in production is Scala/Python UDFs. Before writing a UDF, always check whether the same logic can be expressed with `functions.*` built-ins. Built-ins fuse into WSCG; UDFs do not. The performance gap is not marginal - it is often $10-50×$.

> [!TIP]
> To verify a specific operator supports codegen -
> ```scala
> // In Spark source or shell
> val plan = df.queryExecution.executedPlan
> plan.foreach {
>     case w: WholeStageCodegenExec =>
>         println(s"CodeGen stage ${w.codegenStageId}: ${w.child.getClass.getSimpleName}")
>     case other =>
>         println(s"Interpreted: ${other.getClass.getSimpleName}")
> }
> ```
