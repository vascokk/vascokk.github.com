---
layout: post
title: "Machine Learning in Erlang and CUDA"
date: 2013-03-23 18:49
comments: true
categories: Еrlang Machine Learning CUDA GPU
---

Since I finished the "Machine Learning" course on Coursera, I wanted to try it in Erlang, Unfortunately Erlang is notorious with its inability to do intensive math calculations. In short - no matrix calculations - no Machine learning. Solution - use C and call C-functions from Erlang. Or better - use the GPU (in C) and call the functions from Erlang. I guess, this approach is no more qualified as "pure Erlang" one, but that is the best we can get. In fact - most of the Python ML libraries, so famous in the ML society, are using NumPy library, which guess what - is written in C (at least - the important part of it).

So, first thing to do was - google "erlang matrix operations cuda gpu"....Almost nothing. Actually, I found two projects - OpenCL bindings for erlang [https://github.com/tonyrog/cl](https://github.com/tonyrog/cl) and Kevin Smith's [Pteracuda](https://github.com/kevsmith/pteracuda). I liked Kevin's approach, as using the Thrust library would hide the pure C boilerplate CUDA code. Also some algorithms are one-liners implemented with Thrust. 

I forked Pteracuda and added matrix operations (CUBLAS-based), but since it had some parts that I didn't need (like: strings, search, min/max), I decided to create another project and so - the [NumEr](https://github.com/vascokk/NumEr) was born :)

Basically, it is a bunch of Erlang NIF functions for BLAS operations on vectors and matrices. Both are natively implemented as Thrust host/device vectors and special "buffer" classes are used to transfer them from Erlang to CUDA and back. 

Here is an example:

``` erlang
% this is a row-major matrix:
A = [[4.0,6.0,8.0,2.0],[5.0,7.0,9.0,3.0]].

%this is a vector:
X = [2.0,5.0,1.0,7.0].

% create a CUDA context and transfer to "buffers"
{ok, Ctx} = numer_nifs:new_context().
{ok, Buf_A} = numer_nifs:new_matrix_float_buffer(Ctx, A, ?ROW_MAJOR).
{ok, Buf_X} = numer_nifs:new_float_buffer(Ctx).
numer_nifs:write_buffer(Buf_X, X).
```

As you see one of the parameters in the matrix buffer is "?ROW_MAJOR". It is kinda borrowed from Boost library, but not yet fully implemented in NumEr. I mean - currently only row-major matrices are supported. However, under the hood in the Thrust vectors the numbers are stored in column-major format. I chose to do it in this way, because the CUBLAS library is using column-major storage - being a derivative of the FORTRAN BLAS library.

There are several modules, which are wrappers for the NIF functions, like: numer\_blas.erl - for BLAS operations, numer\_buffer.erl - for operations with buffers (new, delete, read, write), etc.

Using numer\_buffer module, the above example will look like:

``` erlang
 {ok, Ctx} = numer_context:new().
 {ok, Buf_A} = numer_buffer:new(Ctx, matrix, float, row_major, A).
 {ok, Buf_X} = numer_buffer:new(Ctx, float).
 numer_buffer:write(Buf_X, X).
``` 

Now, the more interesting part - calculations with vectors and matrices. Here is the BLAS GEMV example:

``` erlang
%  GEMV: y <- α op ( A ) x + β y
gemv_test()->
    {ok, Ctx} = numer_context:new(),
    A = [[4.0,6.0,8.0,2.0],[5.0,7.0,9.0,3.0]],
    _m = 2, %rows A
    _n = 4, %columns A
    _alpha = 1.0,
    _beta = 0.0,
    X = [2.0,5.0,1.0,7.0],
    Y = [0.0, 0.0], 
    {ok, Buf_A} = numer_buffer:new(Ctx, matrix, float, row_major, A),
    {ok, Buf_X} = numer_buffer:new(Ctx, float),
    numer_buffer:write(Buf_X, X),
    {ok, Buf_Y} = numer_buffer:new(Ctx, float),
    numer_buffer:write(Buf_Y, Y),
    ok = numer_blas:gemv(Ctx, no_transpose , _m, _n, _alpha, Buf_A, Buf_X, _beta, Buf_Y),
    {ok, [60.0,75.0]} = numer_buffer:read(Buf_Y),
    ok = numer_buffer:destroy(Buf_A),
    ok = numer_buffer:destroy(Buf_X),
    ok = numer_buffer:destroy(Buf_Y),
    ok = numer_context:destroy(Ctx).
```

As for the ML part of the whole exercise - there is an implementation of the Logistic Regression (without regularization) algorithm. Take a look at the numer\_logreg.erl module.

The numer\_ml.erl module contains an all-native implementation of Logistic Regression, while the numer\_logreg.erl is using buffers and NIFs. The first one I used to compare the speed between native and "Erlang" implementations.

Since using buffer operations can make the code awkward to read, there is also a helper module - numer\_helpers.erl, wich can be used for prototyping the algorithms. WARNING - it is extremely slow and suitable ONLY for prototyping. Here is how you can use it:

``` erlang
gemv_2_test()->
    A = [[4.0,6.0,8.0,2.0],[5.0,7.0,9.0,3.0]],
    _alpha = 1.0,
    _beta = 0.0,
    X = [2.0,5.0,1.0,7.0],
    Res = numer_helpers:gemv(no_transpose , _alpha, A, X, _beta, []),
    ?assertEqual([60.0,75.0], Res).
```

It is much more readable and useful for one-time calculations, but in the ML "training" stage (with hundreds of iterations) it will be unusable, due to the multiple buffer transfers. 

The project is still a work in progress and needs a lot of polishing and if anyone is willing to give a hand I'll be more than happy. Any suggestions to improve the framework are also very welcome.

That's all folks! Happy hacking! :-)

Link to the GitHub repository: [https://github.com/vascokk/NumEr](https://github.com/vascokk/NumEr)