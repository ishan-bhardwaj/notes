# Core Java Vol. II — Chapter 5: Database Programming

## JDBC Design

- __JDBC__ — Java API for communicating SQL statements to relational databases via pluggable drivers
- Follows the ODBC model: application code talks to driver manager, which delegates to a database-specific driver
- Driver manager (`DriverManager`) iterates registered drivers to find one matching the JDBC URL subprotocol

### Driver Types

- __Type 1__ — translates JDBC to ODBC; requires native ODBC driver; deprecated, no longer in JDK
- __Type 2__ — partly Java, partly native; requires platform-specific code on client
- __Type 3__ — pure Java; database-independent protocol to a server-side translator
- __Type 4__ — pure Java; translates JDBC directly to database-specific protocol; most common today

### Deployment Models

- __Two-tier__ — Java client connects directly to database via JDBC driver on client
- __Three-tier__ — Java client talks HTTP to middleware; middleware uses JDBC to query database; separates presentation, business logic, and data

---

## SQL Basics

### Core Statements

```sql
SELECT ISBN, Price, Title FROM Books WHERE Price <= 29.95
SELECT * FROM Books, Publishers WHERE Books.Publisher_Id = Publishers.Publisher_Id
INSERT INTO Books VALUES ('Title', '0-000-00000-0', '0201', 47.95)
UPDATE Books SET Price = Price - 5.00 WHERE Title LIKE '%C++%'
DELETE FROM Books WHERE Title LIKE '%C++%'
CREATE TABLE Books (Title VARCHAR(60), ISBN CHAR(13), Price DECIMAL(10,2))
DROP TABLE Books
```

- `=` and `<>` for equality/inequality (not `==` / `!=`)
- `LIKE` with `%` (zero or more chars) and `_` (one char) for pattern matching
- Single quotes for strings; `''` for a literal single quote inside a string
- `ORDER BY` — required to guarantee row ordering; result set order is otherwise arbitrary
- `JOIN` via multi-table `FROM` + `WHERE` — avoids data duplication; each publisher URL stored once

### Common SQL Types

| SQL Type | Java Type |
|---|---|
| `INTEGER` / `INT` | `int` |
| `SMALLINT` | `short` |
| `NUMERIC(m,n)` / `DECIMAL(m,n)` | `java.math.BigDecimal` |
| `FLOAT(n)` / `DOUBLE` | `double` |
| `REAL` | `float` |
| `CHAR(n)` / `VARCHAR(n)` | `String` |
| `BOOLEAN` | `boolean` |
| `DATE` | `java.sql.Date` |
| `TIME` | `java.sql.Time` |
| `TIMESTAMP` | `java.sql.Timestamp` |
| `BLOB` | `java.sql.Blob` |
| `CLOB` | `java.sql.Clob` |
| `ARRAY` | `java.sql.Array` |

---

## JDBC Configuration

### Database URLs

jdbc:derby://localhost:1527/COREJAVA;create=true

jdbc:postgresql:COREJAVA

Format: `jdbc:subprotocol:other stuff`

### Setup Checklist

- Include driver JAR on classpath at runtime (not needed at compile time): `java -classpath driverJar:. ProgramName`
- Since JDBC 4: driver JARs include `META-INF/services/java.sql.Driver` — auto-registration; no manual `Class.forName()` needed
- For older drivers: `-Djdbc.drivers=com.example.Driver`

### Connecting

```java
Connection conn = DriverManager.getConnection(url, username, password);
```

- Throws `SQLException` on failure
- Enable tracing: `DriverManager.setLogWriter(new PrintWriter(System.out))`
- Use try-with-resources to ensure connection is always closed

---

## Working with Statements

### Executing SQL

```java
Statement stat = conn.createStatement();

// DDL and DML (INSERT, UPDATE, DELETE, CREATE, DROP)
int rowsAffected = stat.executeUpdate("UPDATE Books SET Price = Price - 5");

// SELECT
ResultSet rs = stat.executeQuery("SELECT * FROM Books");

// Generic (use when statement type is unknown)
boolean isResultSet = stat.execute(command);
```

### Reading ResultSets

```java
while (rs.next()) {
    String isbn = rs.getString(1);       // column index (1-based)
    double price = rs.getDouble("Price"); // column label
}
```

- `next()` must be called before reading the first row
- `hasNext()` does not exist — loop until `next()` returns `false`
- Row order is arbitrary unless `ORDER BY` is specified
- `getXxx` methods perform reasonable type conversions (e.g. `getString` on a numeric column)
- Column indexes are 1-based

### Managing Connections, Statements, ResultSets

- One `Connection` can create multiple `Statement` objects
- One `Statement` has at most one open `ResultSet` at a time — a new query closes the previous result set
- `ResultSet`, `Statement`, `Connection` all have `close()` methods — call when done to free server resources
- `stat.closeOnCompletion()` — auto-closes statement when all its result sets are closed
- Use try-with-resources for `Connection`; inner resources close automatically via statement/connection close

### SQL Exceptions

```java
for (Throwable t : sqlException) {  // Iterable<Throwable> chains both SQLException chain and cause chain
    t.printStackTrace();
}
sqlException.getSQLState();   // X/Open or SQL:2003 standardised code
sqlException.getErrorCode();  // vendor-specific
```

- `SQLWarning` — non-fatal; chained via `getNextWarning()`; retrieve from connection, statement, or result set
- `DataTruncation extends SQLWarning` — data truncated on read; thrown as exception on update

---

## Prepared Statements

```java
PreparedStatement pstat = conn.prepareStatement(
    "SELECT * FROM Books WHERE Publisher_Id = ? AND Price < ?");
pstat.setString(1, publisherId);
pstat.setDouble(2, maxPrice);
ResultSet rs = pstat.executeQuery();
int rowsAffected = pstat.executeUpdate(); // for INSERT/UPDATE/DELETE
```

- Host variables marked with `?`; indexed 1-based
- Bound values persist across re-executions unless changed or `clearParameters()` called
- Prevents SQL injection
- Database caches query plan — call `prepareStatement` freely, overhead is minimal
- Always prefer prepared statements over string concatenation when variables are involved

---

## LOBs (Large Objects)

### Reading

```java
Blob blob = rs.getBlob("Cover");
InputStream in = blob.getBinaryStream();

Clob clob = rs.getClob("Notes");
Reader reader = clob.getCharacterStream();
String sub = clob.getSubString(1, 100);
```

### Writing

```java
Blob blob = conn.createBlob();
OutputStream out = blob.setBinaryStream(0);
// write to out...
pstat.setBlob(2, blob);
pstat.executeUpdate();
```

- Actual data fetched lazily from the database when individual values are requested

---

## SQL Escapes

Portable syntax the JDBC driver translates to database-specific syntax:

```sql
{d '2024-01-24'}                   -- DATE literal
{t '23:59:59'}                     -- TIME literal
{ts '2024-01-24 23:59:59.999'}     -- TIMESTAMP literal
{fn left(?, 20)}                   -- scalar function call
{fn user()}                        -- scalar function, no args
{oj Books LEFT OUTER JOIN Publishers ON ...}  -- outer join
... WHERE col LIKE %!_% {escape '!'}          -- escape char in LIKE
{call PROC1(?, ?)}                 -- stored procedure call
{call ? = PROC3(?)}                -- stored procedure with return value
```

### Stored Procedures

```java
CallableStatement cstat = conn.prepareCall("{call PROC4(?, ?)}");
cstat.setInt(1, id);
cstat.registerOutParameter(2, java.sql.Types.VARCHAR);
cstat.execute();
String name = cstat.getString(2);
```

---

## Multiple Results

```java
boolean isResult = stat.execute(command);
while (true) {
    if (isResult) {
        ResultSet result = stat.getResultSet();
        // process result
    } else {
        int count = stat.getUpdateCount();
        if (count < 0) break;   // no more results
        // process update count
    }
    isResult = stat.getMoreResults();
}
```

---

## Autogenerated Keys

```java
stat.executeUpdate(insertStatement, Statement.RETURN_GENERATED_KEYS);
ResultSet rs = stat.getGeneratedKeys();
if (rs.next()) {
    int key = rs.getInt(1);
}
```

---

## Scrollable and Updatable Result Sets

```java
Statement stat = conn.createStatement(
    ResultSet.TYPE_SCROLL_INSENSITIVE,
    ResultSet.CONCUR_UPDATABLE);
ResultSet rs = stat.executeQuery(query);
```

### ResultSet Types

| Constant | Behaviour |
|---|---|
| `TYPE_FORWARD_ONLY` | Default; next() only |
| `TYPE_SCROLL_INSENSITIVE` | Scrollable; does not reflect database changes after query |
| `TYPE_SCROLL_SENSITIVE` | Scrollable; reflects database changes |

### Concurrency

| Constant | Behaviour |
|---|---|
| `CONCUR_READ_ONLY` | Default; cannot update database |
| `CONCUR_UPDATABLE` | Can update database through result set |

### Scrolling Methods

```java
rs.previous()           // move backward; returns false if before first row
rs.relative(n)          // n > 0 forward, n < 0 backward
rs.absolute(n)          // jump to row n (1-based)
rs.first() / rs.last()  // jump to first/last row
rs.beforeFirst() / rs.afterLast()  // move before/after all rows
rs.getRow()             // current row number; 0 if not on a row
rs.isFirst() / rs.isLast() / rs.isBeforeFirst() / rs.isAfterLast()
```

### Updating via Result Set

```java
rs.updateDouble("Price", price + increase);
rs.updateRow();           // must call after field updates to write to DB
rs.cancelRowUpdates();    // cancel pending updates for current row
```

### Inserting via Result Set

```java
rs.moveToInsertRow();
rs.updateString("Title", title);
rs.updateString("ISBN", isbn);
rs.insertRow();
rs.moveToCurrentRow();
```

### Deleting via Result Set

```java
rs.deleteRow();  // removes from both result set and database immediately
```

> [!NOTE]
> Updatable result sets work best with single-table queries or joins on primary keys. SQL `UPDATE` is far more efficient for programmatic bulk changes; use updatable result sets for interactive editing only.

---

## Row Sets

- `RowSet` extends `ResultSet` but can operate disconnected from the database
- Suitable for passing query results between application tiers or devices

### Types (from `javax.sql.rowset`)

- `CachedRowSet` — disconnected; stores all data in memory
- `WebRowSet extends CachedRowSet` — can be saved to/loaded from XML
- `FilteredRowSet` — lightweight SQL `WHERE`-like filtering on cached data
- `JoinRowSet` — lightweight SQL `JOIN`-like operations on cached data
- `JdbcRowSet` — thin wrapper around `ResultSet`; adds `RowSet` interface methods

### Using `CachedRowSet`

```java
RowSetFactory factory = RowSetProvider.newFactory();
CachedRowSet crs = factory.createCachedRowSet();

// Populate from existing ResultSet
crs.populate(result);
conn.close();  // can close connection now

// Or populate directly (auto-connects)
crs.setURL("jdbc:derby://localhost:1527/COREJAVA");
crs.setUsername("dbuser");
crs.setPassword("secret");
crs.setCommand("SELECT * FROM Books WHERE Publisher_Id = ?");
crs.setString(1, publisherId);
crs.execute();  // connects, queries, disconnects

// Pagination
crs.setPageSize(20);
crs.execute();
crs.nextPage();   // fetch next 20 rows
crs.previousPage();

// Write changes back
crs.setTableName("Books");   // required if populated from ResultSet
crs.acceptChanges(conn);     // reconnects and issues SQL for all edits
```

- `acceptChanges` checks that original values still match current database values before writing
- Throws `SyncProviderException` if data changed in database since row set was populated

---

## Metadata

### Database Metadata

```java
DatabaseMetaData meta = conn.getMetaData();
ResultSet tables = meta.getTables(null, null, null, new String[]{"TABLE"});
// Column 3 = TABLE_NAME
while (tables.next()) System.out.println(tables.getString(3));

meta.supportsResultSetType(ResultSet.TYPE_SCROLL_INSENSITIVE)
meta.supportsResultSetConcurrency(type, concurrency)
meta.getMaxStatements()   // max concurrent open statements per connection
meta.getMaxConnections()
meta.getJDBCMajorVersion() / getJDBCMinorVersion()
```

### ResultSet Metadata

```java
ResultSetMetaData rsmd = rs.getMetaData();
int colCount = rsmd.getColumnCount();
String label = rsmd.getColumnLabel(i);    // AS alias or column name; 1-based
int width = rsmd.getColumnDisplaySize(i);
String className = rsmd.getColumnClassName(i);
```

### Parameter Metadata

```java
ParameterMetaData pmd = pstat.getParameterMetaData();
for (int i = 1; i <= pmd.getParameterCount(); i++) {
    pmd.getParameterTypeName(i);
    pmd.getParameterMode(i);  // IN, OUT, INOUT, UNKNOWN
}
```

---

## Transactions

### Basic Transaction Control

```java
conn.setAutoCommit(false);   // turn off per-statement commits
try {
    stat.executeUpdate(command1);
    stat.executeUpdate(command2);
    conn.commit();
} catch (SQLException e) {
    conn.rollback();
}
```

- Default is `autocommit = true` — each statement committed immediately
- After `commit()`, changes are permanent and cannot be rolled back
- `rollback()` reverts all changes since the last `commit()`

### Savepoints

```java
Savepoint svpt = conn.setSavepoint();       // unnamed
Savepoint svpt = conn.setSavepoint("name"); // named
conn.rollback(svpt);                        // roll back to savepoint only
conn.releaseSavepoint(svpt);                // release when no longer needed
```

### Batch Updates

```java
boolean prev = conn.getAutoCommit();
conn.setAutoCommit(false);
Statement stat = conn.createStatement();
stat.addBatch("CREATE TABLE ...");
stat.addBatch("INSERT INTO ...");
int[] counts = stat.executeBatch();  // returns row count per statement
conn.commit();
conn.setAutoCommit(prev);
```

- Only DDL and DML in batches — `SELECT` throws exception
- `SUCCESS_NO_INFO` (-2) — statement succeeded but no row count available
- `EXECUTE_FAILED` (-3) — statement failed
- Check support: `conn.getMetaData().supportsBatchUpdates()`
- Treat entire batch as one transaction — rollback on failure

---

## Connection Management in Production

- `DriverManager.getConnection` is suitable for small programs only
- In web/enterprise environments: use JNDI to look up a `DataSource`

```java
var ctx = new InitialContext();
DataSource source = (DataSource) ctx.lookup("java:comp/env/jdbc/corejava");
Connection conn = source.getConnection();
```

- In Java EE containers: inject via `@Resource(name="jdbc/corejava")`
- __Connection pooling__ — connections are reused rather than physically closed
    - Configured by the web container or application server (e.g. Tomcat, GlassFish)
    - Transparent to the programmer: `getConnection()` and `close()` work the same way
    - `close()` returns the connection to the pool rather than closing it physically
    - Pools typically cache prepared statements as well