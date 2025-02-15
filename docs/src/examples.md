# Examples

The first step is to create **FD** variables which are then passed to the function you want to differentiate. The return value is a graph structure which **FD** will analyze to generate efficient executables or symbolic expressions.
 
**FD** uses a global cache for common subexpression elimination so the **FD** expression preprocessing step is not thread safe. 

Under ordinary conditions the memory used by the cache won't be an issue. But, if you have a long session where you are creating many complex functions it is possible the cache will use too much memory. If this happens call the function `clear_cache` after you have completely processed your expression.

Set up variables:
```
using FastDifferentiation

@variables x y z

```
 Make a vector of variables
```
julia> X = make_variables(:x,3)
3-element Vector{Node}:
 x1
 x2
 x3
```

Make an executable function
```
julia> xy_exe = make_function([x^2*y^2,sqrt(x*y)],[x,y]) #[x,y] vector specifies the order of the arguments to the exe
...
julia> xy_exe([1.0,2.0])
2-element Vector{Float64}:
 4.0
 1.4142135623730951
```
Compute Hessian:
```
@variables x y z

julia> h_symb = hessian(x^2+y^2+z^2,[x,y,z])
3×3 Matrix{Node}:
 2    0.0  0.0
 0.0  2    0.0
 0.0  0.0  2

julia> h_symb1 = hessian(x^2*y^2*z^2,[x,y,z])
3×3 Matrix{FastDifferentiation.Node}:
 (2 * ((z ^ 2) * (y ^ 2)))        (((2 * x) * (2 * y)) * (z ^ 2))  (((2 * x) * (2 * z)) * (y ^ 2))
 (((2 * y) * (2 * x)) * (z ^ 2))  (2 * ((z ^ 2) * (x ^ 2)))        (((2 * y) * (2 * z)) * (x ^ 2))
 (((2 * z) * (2 * x)) * (y ^ 2))  (((2 * z) * (2 * y)) * (x ^ 2))  (2 * ((x ^ 2) * (y ^ 2)))

julia> hexe_1 = make_function(h_symb1,[x,y,z])
...
julia> hexe_1([1.0,2.0,3.0])
3×3 Matrix{Float64}:
 72.0  72.0  48.0
 72.0  18.0  24.0
 48.0  24.0   8.0
```
Compute `Hv` without forming the full Hessian matrix. This is useful if the Hessian is very large
```
julia> @variables x y
y

julia> f = x^2 * y^2
((x ^ 2) * (y ^ 2))

julia> hv_fast, v_vec2 = hessian_times_v(f, [x, y])
...

julia> hv_fast_exe = make_function(hv_fast, [[x, y]; v_vec2]) #need v_vec2 because hv_fast is a function of x,y,v1,v2 and have to specify the order of all inputs to the executable
...
julia> hv_fast_exe([1.0,2.0,3.0,4.0]) #first two vector elements are x,y last two are v1,v2
2-element Vector{Float64}:
 56.0
 32.0
```
Compute Jacobian:
```
julia> f1 = cos(x) * y
(cos(x) * y)

julia> f2 = sin(y) * x
(sin(y) * x)

julia> symb = jacobian([f1, f2], [x, y]) #non-destructive
2×2 Matrix{Node}:
 (y * -(sin(x)))  cos(x)
 sin(y)           (x * cos(y))
```
Create executable to evaluate Jacobian:
```
julia> jac_exe = make_function(symb,[x,y])
...
julia> jac_exe([1.0,2.0])
2×2 Matrix{Float64}:
 -1.68294    0.540302
  0.909297  -0.416147
```
Executable with in_place matrix evaluation to avoid allocation of a matrix for the Jacobian (in_place option available on all executables including Jᵀv,Jv,Hv):
```
julia> jac_exe = make_function(symb,[x,y], in_place=true)
...
julia> a = Matrix{Float64}(undef,2,2)
2×2 Matrix{Float64}:
 0.0  0.0
 0.0  6.93532e-310

julia> jac_exe([1.0,2.0],a)
2×2 Matrix{Float64}:
 -1.68294    0.540302
  0.909297  -0.416147

julia> a
2×2 Matrix{Float64}:
 -1.68294    0.540302
  0.909297  -0.416147
```

For out of place evaluation (matrix created and returned by executable function) input vector and return matrix of executable can be any mix of StaticArray and Vector. If the first argument to `make_function` is a subtype of StaticArray then the compiled executable will return a StaticArray value. The compiled executable can be called with either an `SVector` or `Vector` argument. For small input sizes the `SVector` should be faster, essentially the same as passing the input as scalar values.

For functions with low input and output dimensions the fastest executable will be generated by calling `make_function` with first argument a subtype of StaticArray and calling the executable with an SVector argument. The usual cautions of StaticArrays apply, that total length of the return value < 100 or so and total length of the input < 100 or so.

```
julia> @variables x y
y

julia> j = jacobian([x^2 * y^2, cos(x + y), log(x / y)], [x, y]);

julia> j_exe = make_function(j, [x, y]);

julia> @assert typeof(j_exe([1.0, 2.0])) <: Array #return type is Array and input type is Vector

julia> j_exe2 = make_function(SArray{Tuple{3,2}}(j), [x, y]);

julia> @assert typeof(j_exe2(SVector{2}([1.0, 2.0]))) <: StaticArray #return type is StaticArray and input type is SVector. This should be the fastest.
```


Compute any subset of the columns of the Jacobian:
```
julia> symb = jacobian([x*y,y*z,x*z],[x,y,z]) #all columns
3×3 Matrix{Node}:
 y    x    0.0
 0.0  z    y
 z    0.0  x

julia> symb = jacobian([x*y,y*z,x*z],[x,y]) #first two columns
3×2 Matrix{Node}:
 y    x
 0.0  z
 z    0.0

julia> symb = jacobian([x*y,y*z,x*z],[z,y]) #second and third columns, reversed so ∂f/∂z is 1st column of the output, ∂f/∂y the 2nd
3×2 Matrix{Node}:
 0.0  x
 y    z
 x    0.0
```

Symbolic and executable Jᵀv and Jv (see this [paper](https://arxiv.org/abs/1812.01892) for applications of this operation).
```
julia> (f1,f2) = cos(x)*y,sin(y)*x
((cos(x) * y), (sin(y) * x))

julia> jv,vvec = jacobian_times_v([f1,f2],[x,y])
...

julia> jv_exe = make_function(jv,[[x,y];vvec])
...

julia> jv_exe([1.0,2.0,3.0,4.0]) #first 2 arguments are x,y values and last two are v vector values

2×1 Matrix{Float64}:
 -2.8876166853748195
  1.0633049342884753

julia> jTv,rvec = jacobian_transpose_v([f1,f2],[x,y])
...

julia> jtv_exe = make_function(jTv,[[x,y];rvec])
...
julia> jtv_exe([1.0,2.0,3.0,4.0])
2-element Vector{Float64}:
 -1.4116362015446517
 -0.04368042858415033
```

Convert between FastDifferentiation and Symbolics representations (requires [FDConversion](https://github.com/brianguenter/FDConversion/tree/main) package, not released yet[^1]):
```
julia> f = x^2+y^2 #Symbolics expression
x^2 + y^2

julia> Node(f) #convert to FastDifferentiation form
x^2 + y^2

julia> typeof(ans)
Node{SymbolicUtils.BasicSymbolic{Real}, 0}

julia> node_exp = x^3/y^4 #FastDifferentiation expression
((x ^ 3) / (y ^ 4))

julia> to_symbolics(node_exp)
(x^3) / (y^4)

julia> typeof(ans)
Symbolics.Num
```

[^1]: I am working with the SciML team to see if it is possible to integrate **FD** differentiation directly into Symbolics.jl.