## Numpy

- **Vector dot product**

**a · b** = **aᵀ b** = Σ<sub>d=1</sub><sup>D</sup> (a<sub>d</sub> b<sub>d</sub>)

Where **D** is the dimension of the vectors, and **a<sub>d</sub>** and **b<sub>d</sub>** are the components of **a** and **b** at dimension **d**.

- **Matrix Multiplication (C = AB)**

C<sub>i,j</sub> = A<sub>i,1</sub>B<sub>1,j</sub> + A<sub>i,2</sub>B<sub>2,j</sub> + A<sub>i,3</sub>B<sub>3,j</sub> + ... + A<sub>i,K</sub>B<sub>K,j</sub>

Where:

- **C<sub>i,j</sub>** represents the element in the i-th row and j-th column of matrix **C**,
- **A<sub>i,k</sub>** represents the element in the i-th row and k-th column of matrix **A**,
- **B<sub>k,j</sub>** represents the element in the k-th row and j-th column of matrix **B**, and
- **K** is the number of columns in matrix **A** (or equivalently, the number of rows in matrix **B**).
