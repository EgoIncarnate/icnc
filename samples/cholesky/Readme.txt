Description
-----------
Given a symmetric positive definite matrix A, the Cholesky decomposition is
a lower triangular matrix L such that A=L.L^(T). 

Cholesky has six steps (k_compute), (kj_compute), (kji_compute), (S1_compute),
(S2_compute) and (S3_compute).  (k_compute), (kj_compute) and (kji_compute)
take in 'p' as input which is n/b, where 'n' is the matrix size, 
'b' is block/tile size. Both 'n' and 'b' are command line arguments.
They produce loop indices <tags> for the execution of the other steps. 

(S1_compute) performs unblocked cholesky factorization of the input tile
[Lkji:k,k,k], which is an input from the environment. It produces output
item [Lkji:k,k,k+1] which is one of the inputs to (S2_compute).
This step performs triangular system solve, also takes in as input
[Lkji:j,k,k] and ouputs [Lkji:j,k,k+1].  (S3_compute) performs a symmetric
rank-k update and takes in as input [Lkji:j,i,k] along with [Lkji:j,k,k+1] 
if its a diagonal tile. If its not a diagonal tile, it also takes in
[Lkji:i,k,k+1] as input. It produces [Lkji:j,i,k+1] as output. 

The constraints on parallelism for this kernel are as follows: a [Lkji]
instance may not be processed by a compute step until the [Lkji] instance
and the associated controlling tags <control_S1>, <control_S2> 
or <control_S3> have been produced. For example, one of the inputs to
the step (S2_compute) is computed by step (S1_compute) and the loop indices
are tags <control_S2> which are generated by step (kj_compute). So unless these
steps execute and produce the ouput, step (S2_compute) cannot execute.

The constraints on the algorithm are as follows: the input matrix should be
a symmetric positive definite (SPD) matrix. Otherwise Cholesky factorization
cannot be performed. 

The version in cholesky_mkl uses sequential BLAS routines for its
matrix-multiplication. These calls in MKL highly optimized and so
significantly improve the performance of the CnC implementation. You
need icc and MKL to be able to build this implementation.


BLAS/MKL
========
For better performance, the implementation can use cblas_dgemm and
cblas_dsyrk instead of a naive nested loop (in S3). It will however not use any
paralleism within MKL (all parallelism there is explicitly switched
off).
To turn this on, you need blas/MKL. Compile with -DUSE_MKL. The
makefile is setup for MKL's BLAS, just do make USE_MKL=1.



DistCnC enabling
================

The cholesky code is distCnC-ready with a tuner which implements a
selection of distribution plans which can be slected at runtime.
Data and computation distribution are synced by using compute_on
within consumed_on. The distribution is changed only in coompute_on,
the data distribution follows automatically and optimally.
Use the preprocessor definition _DIST_ to enable distCnC.
    
See the runtime_api reference for how to run distCnC programs.


Usage
-----

The command line is:

cholesky N BS [-i infile] [-o outfile] [-w mfile] [-dt {0,1,2,3}]
	 N :	input SPD matrix size
	 BS:	block/tile size
	 infile:  input matrix file name, if omitted, a random matrix
	          will be generated
         outfile: if specified: resulting lower triangular matrix
                  to this file
	 mfile:   if specified, the generated matrix will be written
                  to this file
         dt:      distribution type (distCnC)
                  BLOCKED_ROWS = 0,
                  ROW_CYCLIC = 1,
                  COLUMN_CYCLIC = 2,
                  BLOCKED_CYCLIC = 3

e.g.
cholesky 1000 50 -i m1000.in -o /tmp/res
will read in a 1000x1000 matrix from file m1000.in and compute
eigen-values with a block-size of 50 and write the result to file
/tmp/res.
