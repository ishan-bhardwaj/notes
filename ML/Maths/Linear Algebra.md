# Linear Algebra

- When formalizing intuitive concepts, a common approach is to construct a set of objects (symbols) and a set of rules to manipulate these objects. This is known as an **algebra**.
- **Linear algebra** is the study of _vectors_ and certain algebra _rules_ to manipulate vectors.
- Any object that satisfies these two properties can be considered a _vector_ (written in bold letters) - 
    - can be added together.
    - can be multiplied by scalars to produce another object of the same kind.

- Example -
    - Geometric vectors are directed segments -
        - Two geometric vectors **x** & **y** can be added to create another geometric vector - **x** + **y** = **z**
        - Multiplication by scalar also creates another geometric vector - λ **x**, λ ∈ ℝ

    - Polynomials -
        - Two polynomials can be added together, which results in another polynomial.
        - Can be multiplied by a scalar λ ∈ ℝ, and the result is a polynomial.

    - Audio signals -
        - Audio signals are represented as a series of numbers. We can add audio signals together, and their sum is a new audio signal. 
        - If we scale an audio signal, we also obtain an audio signal.

    - Elements of ℝ<sup>n</sup> (tuples of `n` real numbers) -
        - Adding two vectors **a**, **b** ∈ ℝ<sup>n</sup> component-wise results in another vector - **a** + **b** = **c** ∈ ℝ<sup>n</sup>
        - Multiplying **a** ∈ ℝ<sup>n</sup> by λ ∈ ℝ results in a scaled vector λ**a** ∈ ℝ<sup>n</sup>

## Systems of Linear Equations

- Analogy -
    - A company produces products N<sub>1</sub>, ... , N<sub>n</sub> for which resources R<sub>1</sub>, ... , R<sub>m</sub> are required.

    - a<sub>ij</sub> units of resource R<sub>i</sub> are needed To produce a unit of product N<sub>j</sub>.

    - The objective is to find an optimal production plan, i.e., a plan of how many units x<sub>j</sub> of product N<sub>j</sub> should be produced if a total of b<sub>i</sub> units of resource R<sub>i</sub> are available and (ideally) no resources are left over.

    - If we produce x<sub>1</sub>, ... , x<sub>n</sub> units of the corresponding products, we need a total of - a<sub>i1</sub>x<sub>1</sub> + ... + a<sub>in</sub>x<sub>n</sub> many units of resource R<sub>i<sub>

    - An optimal production plan (x<sub>1</sub>, ... , x<sub>n</sub>) ∈ ℝ<sup>n</sup>, therefore, has to satisfy the following system of equations -

        a<sub>11</sub>x<sub>1</sub> + ... + a<sub>1n</sub>x<sub>n</sub> = b<sub>1</sub>
    
        ...
    
        a<sub>m1</sub>x<sub>1</sub> + ... + a<sub>mn</sub>x<sub>n</sub> = b<sub>m</sub>

        where a<sub>ij</sub> ∈ ℝ and b<sub>i</sub> ∈ ℝ (1.1)

- Equation (1.1) is the general form of a _system of linear equations_, and x<sub>1</sub>, ... , x<sub>n</sub> are the _unknowns_ of this system.

- Every n-tuple (x<sub>1</sub>, ... , x<sub>n</sub>) ∈ ℝ<sup>n</sup> that satisfies (1.1) is a solution of the linear equation system.


    