```@meta
CurrentModule = StatsModels
DocTestSetup = quote
    using StatsModels
    using Random
    Random.seed!(1)
end
```

# Modeling tabular data

Most statistical models require that data be represented as a `Matrix`-like
collection of a single numeric type.  Much of the data we want to model,
however, is **tabular data**, where data is represented as a collection of
fields with possibly heterogeneous types.  One of the primary goals of
`StatsModels` is to make it simpler to transform tabular data into matrix format
suitable for statistical modeling.

At the moment, "tabular data" means a
[Tables.jl](https://github.com/JuliaData/Tables.jl) table, which will be
materialized as a `Tables.ColumnTable` (a `NamedTuple` of column vectors).  Work
on first-class support for streaming/row-oriented tables is ongoing.

## The `@formula` language

StatsModels implements the [`@formula`](@ref) domain-specific language for
describing table-to-matrix transformations.  This language is designed to be
familiar to users of other statistical software, while also taking advantage of
Julia's unique strengths to be fast and flexible.

A basic formula is composed of individual *terms*—symbols which refer to data
columns, or literal numbers `0` or `1`—combined by `+`, `&`, `*`, and (at the
top level) `~`.

!!! note 

    The `@formula` macro **must** be called with parentheses to ensure that
    the formula is parsed properly.

Here is an example of the `@formula` in action:

```jldoctest 1
julia> using StatsModels, DataFrames

julia> f = @formula(y ~ 1 + a + b + c + b&c)
FormulaTerm
Response:
  y(unknown)
Predictors:
  1
  a(unknown)
  b(unknown)
  c(unknown)
  b(unknown) & c(unknown)

julia> df = DataFrame(y = rand(9), a = 1:9, b = rand(9), c = repeat(["d","e","f"], 3))
9×4 DataFrame
│ Row │ y          │ a     │ b         │ c      │
│     │ Float64    │ Int64 │ Float64   │ String │
├─────┼────────────┼───────┼───────────┼────────┤
│ 1   │ 0.236033   │ 1     │ 0.986666  │ d      │
│ 2   │ 0.346517   │ 2     │ 0.555751  │ e      │
│ 3   │ 0.312707   │ 3     │ 0.437108  │ f      │
│ 4   │ 0.00790928 │ 4     │ 0.424718  │ d      │
│ 5   │ 0.488613   │ 5     │ 0.773223  │ e      │
│ 6   │ 0.210968   │ 6     │ 0.28119   │ f      │
│ 7   │ 0.951916   │ 7     │ 0.209472  │ d      │
│ 8   │ 0.999905   │ 8     │ 0.251379  │ e      │
│ 9   │ 0.251662   │ 9     │ 0.0203749 │ f      │

julia> f = apply_schema(f, schema(f, df))
FormulaTerm
Response:
  y(continuous)
Predictors:
  1
  a(continuous)
  b(continuous)
  c(DummyCoding:3→2)
  b(continuous) & c(DummyCoding:3→2)

julia> resp, pred = modelcols(f, df);

julia> pred
9×7 Array{Float64,2}:
 1.0  1.0  0.986666   0.0  0.0  0.0       0.0
 1.0  2.0  0.555751   1.0  0.0  0.555751  0.0
 1.0  3.0  0.437108   0.0  1.0  0.0       0.437108
 1.0  4.0  0.424718   0.0  0.0  0.0       0.0
 1.0  5.0  0.773223   1.0  0.0  0.773223  0.0
 1.0  6.0  0.28119    0.0  1.0  0.0       0.28119
 1.0  7.0  0.209472   0.0  0.0  0.0       0.0
 1.0  8.0  0.251379   1.0  0.0  0.251379  0.0
 1.0  9.0  0.0203749  0.0  1.0  0.0       0.0203749

julia> coefnames(f)
("y", ["(Intercept)", "a", "b", "c: e", "c: f", "b & c: e", "b & c: f"])

```

Let's break down the formula expression ` y ~ 1 + a + b + c + b&c`:

At the top level is the **formula separator** `~`, which separates the left-hand
(or response) variable `y` from the right-hand size (or predictor) variables on
the right `1 + a + b + c + b&c`.

The left-hand side has one term `y` which means that the response variable is
the column from the data named `:y`.  The response can be accessed with the
analogous `response(f, df)` function.

The right hand side is made up of a number of different **terms**, separated by
`+`: `1 + a + b + c + b&c`.  Each term corresponds to one or more columns in the
generated model matrix: 

* The first term `1` generates a constant or "intercept" column full of `1.0`s.
* The next two terms `a` and `b` correspond to columns from the data table
  called `:a`, `:b`, which both hold numeric data (`Float64` and `Int`
  respectively).  By default, numerical columns are assumed to correspond to
  **continuous terms**, and are converted to `Float64` and copied to the model
  matrix.
* The term `c` corresponds to the `:c` column in the table, which is _not_
  numeric, so it has been [contrast coded](@ref Modeling-categorical-data):
  there are three unique values or levels, and the default coding scheme
  ([`DummyCoding`](@ref)) generates an indicator variable for each level after
  the first (e.g., `df[:c] .== "b"` and `df[:c] .== "a"`).
* The last term `b&c` is an **interaction term**, and generates model matrix
  columns for each _pair_ of columns generated by the `b` and `c` terms.
  Columns are combined with element-wise multiplication.  Since `b` generates
  only a single column and `c` two, `b&c` generates two columns, equivalent to
  `df[:b] .* (df[:c] .== "b")` and `df[:b] .* (df[:c] .== "c")`.

Because we often want to include both "main effects" (`b` and `c`) and
interactions (`b&c`) of multiple variables, within a `@formula` the `*`
operator denotes this "main effects and interactions" operation:

```jldoctest 1
julia> @formula(y ~ 1 + a + b*c)
FormulaTerm
Response:
  y(unknown)
Predictors:
  1
  a(unknown)
  b(unknown)
  c(unknown)
  b(unknown) & c(unknown)
```

Also note that the interaction operators `&` and `*` are distributive with the
term separator `+`:

```jldoctest 1
julia> @formula(y ~ 1 + (a + b) & c)
FormulaTerm
Response:
  y(unknown)
Predictors:
  1
  a(unknown) & c(unknown)
  b(unknown) & c(unknown)
```

## Julia functions in a `@formula`

Any calls to Julia functions that don't have special meaning (or are part of an
[extension](@ref Internals-and-extending-the-formula-DSL) provided by a modeling
package) are treated like normal Julia code, and evaluated elementwise:

```jldoctest 1
julia> modelmatrix(@formula(y ~ 1 + a + log(1+a)), df)
9×3 Array{Float64,2}:
 1.0  1.0  0.693147
 1.0  2.0  1.09861
 1.0  3.0  1.38629
 1.0  4.0  1.60944
 1.0  5.0  1.79176
 1.0  6.0  1.94591
 1.0  7.0  2.07944
 1.0  8.0  2.19722
 1.0  9.0  2.30259
```

Note that the expression `1 + a` is treated differently as part of the formula
than in the call to `log`, where it's interpreted as normal addition.

This even applies to custom functions.  For instance, if for some reason you
wanted to include a regressor based on a `String` column that encoded whether
any character in a string was after `'e'` in the alphabet, you could do

```jldoctest 1
julia> gt_e(s) = any(c > 'e' for c in s)
gt_e (generic function with 1 method)

julia> modelmatrix(@formula(y ~ 1 + gt_e(c)), df)
9×2 Array{Float64,2}:
 1.0  0.0
 1.0  0.0
 1.0  1.0
 1.0  0.0
 1.0  0.0
 1.0  1.0
 1.0  0.0
 1.0  0.0
 1.0  1.0

```

Julia functions like this are evaluated elementwise when the numeric arrays are
created for the response and model matrix.  This makes it easy to fit models to
transformed data _lazily_, without creating temporary columns in your table.
For instance, to fit a linear regression to a log-transformed response:

```jldoctest 1
julia> using GLM

julia> lm(@formula(log(y) ~ 1 + a + b), df)
StatsModels.TableRegressionModel{LinearModel{LmResp{Array{Float64,1}},DensePredChol{Float64,LinearAlgebra.Cholesky{Float64,Array{Float64,2}}}},Array{Float64,2}}

:(log(y)) ~ 1 + a + b

Coefficients:
             Estimate Std.Error  t value Pr(>|t|)
(Intercept)  -4.16168   2.98788 -1.39285   0.2131
a            0.357482  0.342126  1.04489   0.3363
b             2.32528   3.13735 0.741159   0.4866

julia> df[:log_y] = log.(df[:y]);

julia> lm(@formula(log_y ~ 1 + a + b), df)            # equivalent
StatsModels.TableRegressionModel{LinearModel{LmResp{Array{Float64,1}},DensePredChol{Float64,LinearAlgebra.Cholesky{Float64,Array{Float64,2}}}},Array{Float64,2}}

log_y ~ 1 + a + b

Coefficients:
             Estimate Std.Error  t value Pr(>|t|)
(Intercept)  -4.16168   2.98788 -1.39285   0.2131
a            0.357482  0.342126  1.04489   0.3363
b             2.32528   3.13735 0.741159   0.4866

```

The no-op function `identity` can be used to block the normal formula-specific
interpretation of `+`, `*`, and `&`:

```jldoctest 1
julia> modelmatrix(@formula(y ~ 1 + b + identity(1+b)), df)
9×3 Array{Float64,2}:
 1.0  0.986666   1.98667
 1.0  0.555751   1.55575
 1.0  0.437108   1.43711
 1.0  0.424718   1.42472
 1.0  0.773223   1.77322
 1.0  0.28119    1.28119
 1.0  0.209472   1.20947
 1.0  0.251379   1.25138
 1.0  0.0203749  1.02037
```

## Constructing a formula programatically

A formula can be constructed at run-time by creating `Term`s and combining them
with the formula operators `+`, `&`, and `~`:

```jldoctest 1
julia> Term(:y) ~ ConstantTerm(1) + Term(:a) + Term(:b) + Term(:a) & Term(:b)
FormulaTerm
Response:
  y(unknown)
Predictors:
  1
  a(unknown)
  b(unknown)
  a(unknown) & b(unknown)
```

The [`term`](@ref) function constructs a term of the appropriate type from
symbols and numbers, which makes it easy to work with collections of mixed type:

```jldoctest 1
julia> ts = term.((1, :a, :b))
1
a(unknown)
b(unknown)
```

These can then be combined with standard reduction techniques:

```jldoctest 1
julia> f1 = term(:y) ~ foldl(+, ts)
FormulaTerm
Response:
  y(unknown)
Predictors:
  1
  a(unknown)
  b(unknown)

julia> f2 = term(:y) ~ sum(ts)
FormulaTerm
Response:
  y(unknown)
Predictors:
  1
  a(unknown)
  b(unknown)

julia> f1 == f2 == @formula(y ~ 1 + a + b)
true

```

## Fitting a model from a formula

The main use of `@formula` is to streamline specifying and fitting statistical
models based on tabular data.  From the user's perspective, this is done by
`fit` methods that take a `FormulaTerm` and a table instead of numeric
matrices.

As an example, we'll simulate some data from a linear regression model with an
interaction term, a continuous predictor, a categorical predictor, and the
interaction of the two, and then fit a `GLM.LinearModel` to recover the
simulated coefficients.

```jldoctest
julia> using GLM, DataFrames, StatsModels

julia> data = DataFrame(a = rand(100), b = repeat(["d", "e", "f", "g"], 25));

julia> X = StatsModels.modelmatrix(@formula(y ~ 1 + a*b).rhs, data);

julia> β_true = 1:8;

julia> ϵ = randn(100)*0.1;

julia> data[:y] = X*β_true .+ ϵ;

julia> mod = fit(LinearModel, @formula(y ~ 1 + a*b), data)
StatsModels.TableRegressionModel{LinearModel{LmResp{Array{Float64,1}},DensePredChol{Float64,LinearAlgebra.Cholesky{Float64,Array{Float64,2}}}},Array{Float64,2}}

y ~ 1 + a + b + a & b

Coefficients:
             Estimate Std.Error t value Pr(>|t|)
(Intercept)   0.98878 0.0384341 25.7266   <1e-43
a             2.00843 0.0779388 25.7694   <1e-43
b: e          3.03726 0.0616371 49.2764   <1e-67
b: f          4.03909 0.0572857 70.5078   <1e-81
b: g          5.02948 0.0587224 85.6484   <1e-88
a & b: e       5.9385   0.10753 55.2264   <1e-71
a & b: f       6.9073  0.112483 61.4075   <1e-75
a & b: g      7.93918  0.111285 71.3407   <1e-81

```

Internally, this is accomplished in three steps:

1. The expression passed to `@formula` is lowered to term constructors combined
   by `~`, `+`, and `&`, which evaluate to create terms for the whole formula
   and any interaction terms.
2. A schema is extracted from the data, which determines whether each variable
   is continuous or categorical and extracts the summary statistics of each
   variable (mean/variance/min/max or unique levels respectively).  This schema
   is then _applied_ to the formula with `apply_schema(term, schema,
   ::Type{Model})`, which returns a new formula with each placeholder `Term`
   replaced with a concrete `ContinuousTerm` or `CategoricalTerm` as
   appropriate.  This is also the stage where any custom syntax is applied (see
   [the section on extending the `@formula` language](@ref
   Internals-and-extending-the-formula-DSL) for more details).
3. Numeric arrays are generated for the response and predictors from the full
   table using `modelcols(term, data)`.

The `ModelFrame` and `ModelMatrix` types can still be used to do this
transformation, but this is only to preserve some backwards compatibility.
Package authors who would like to include support for fitting models from a
`@formula` are **strongly** encouraged to directly use `schema`, `apply_schema`,
and `modelcols` to handle the table-to-matrix transformations they need.