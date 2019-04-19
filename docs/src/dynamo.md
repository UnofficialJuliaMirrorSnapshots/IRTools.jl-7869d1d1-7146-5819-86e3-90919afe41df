# Dynamo

IRTools can be used with metaprogramming tools like [Cassette](https://github.com/jrevels/Cassette.jl), but it also provides a few of its own utilities. The main one is named the "dynamo" after the idea of a "dynamically-scoped macro".

Let me explain. If you write down

```julia
@foo begin
  bar(baz())
end
```

then the `@foo` macro has access to the expression `bar(baz())` and can modify this however it pleases. However, the code of the functions `bar` and `baz` are completely invisible; in more technical terms the macro has lexical extent.

In contrast, a dynamo looks like this:

```julia
foo() do
  bar(baz())
end
```

This can *also* freely modify the `bar(baz())` expression (though it sees it as an `IR` object rather than `Expr`). But more importantly, it can *recurse*, viewing and manipulating the source code of `bar` and `baz` and even any functions *they* call. In other words, it has dynamic extent.

For example, imagine a macro for replacing `*` with `+`:

```julia
julia> using MacroTools

julia> macro foo(ex)
         MacroTools.prewalk(ex) do x
           x == :* ? :+ : x
         end |> esc
       end
@foo (macro with 1 method)

julia> @foo 10*5
15

julia> @foo prod([5, 10])
50
```

The explicit `*` that appears to the macro gets changed, but the implicit one inside `prod` does not. This guide shows you how to do one better.

## A Simple Dynamo

The simplest possible dynamo is a no-op, analagous to the macro

```julia
macro roundtrip(ex)
  esc(ex)
end
```

Here it is:

```julia
julia> using IRTools: IR, @dynamo

julia> @dynamo roundtrip(meta) = IR(meta)

julia> mul(a, b) = a*b
mul (generic function with 1 method)

julia> roundtrip(mul, 2, 3)
6
```

Here's how it works: our dynamo gets passed a *metadata* object `meta` representing the method we're working on. Typically, the only thing we'll want to do with this is to get its IR with `IR(meta)`. Then we can transform that IR, return it, and it'll be compiled and run as usual.

In this case, we can easily check that the transformed code produced by `roundtrip` is identical to the original IR for `mul`.

```julia
julia> using IRTools: @code_ir

julia> @code_ir mul(2, 3)
1:
  %1 = _2 * _3
  return %1

julia> @code_ir roundtrip mul(1, 2)
1:
  %1 = _2 * _3
  return %1
```

Now we can recreate our `foo` macro. It's a little more verbose since simple symbols like `*` are resolved to `GlobalRef`s in lowered code, but it's broadly the same as our macro.

```julia
@dynamo function foo(meta)
  ir = IR(meta)
  ir = prewalk(ir) do x
    x isa GlobalRef && x.name == :(*) && return GlobalRef(Base, :+)
    return x
  end
  return ir
end
```

It behaves identically, too.

```julia
# Check it works
julia> @code_ir foo mul(5, 10)
1:
  %1 = _2 + _3
  return %1

julia> foo(mul, 5, 10)
15

julia> foo() do
         10*5
       end
15

julia> foo() do
         prod([10, 5])
       end
50
```

To get different behaviour we need to *go deeper* – and talk about recursion.

## Recursing

A key difference between macros and dynamos is that dynamos get passed *functions* with they look inside, rather than expressions, so we don't need to write out `mul` when calling `foo(mul, 5, 10)`.

So what if `foo` actually inserted calls to itself when modifying a function? In other words, `prod([1, 2, 3])` would become `foo(prod, [1, 2, 3])`, and so on for each call inside a function. This lets us get the "dynamic extent" property that we talked about earlier.

```julia
@dynamo function foo2(meta)
  meta == nothing && return
  ir = IR(meta)
  ir = prewalk(ir) do x
    x isa GlobalRef && x.name == :(*) && return GlobalRef(Base, :+)
    return x
  end
  for (x, st) in ir
    isexpr(st.expr, :call) || continue
    ir[x] = xcall(Main, :foo2, st.expr.args...)
  end
  return ir
end
```

There are two changes here: firstly, walking over all IR statements to look for, and modify, `call` expressions. Secondly we handle the case where `meta == nothing`, which can happen when we hit things like intrinsic functions for which there is no Julia code. If we return `nothing`, the dynamo will just run that function as usual.

Check it does the transform we wanted:

```julia
julia> mul_wrapped(a, b) = mul(a, b)
mul_wrapped (generic function with 1 method)

julia> @code_ir mul_wrapped(5, 10)
1:
  %1 = (Main.mul)(_2, _3)
  return %1

julia> @code_ir foo2 mul_wrapped(5, 10)
1:
  %1 = (Main.foo2)(Main.mul, _2, _3)
  return %1
```

And that it works as expected:

```julia
julia> foo() do # Does not work (since there is no literal `*` here)
         mul(5, 10)
       end
50

julia> foo2() do # Works correctly
         mul(5, 10)
       end
15

julia> foo2() do
         prod([5, 10])
       end
15
```

This, we have rewritten the `prod` function to actually calculate `sum`, by *internally* rewriting all calls to `*` to instead use `+`.

!!! warning

    A current usability issue with the dynamo is that it is _not_ automatically updated when you redefine functions. For example:

    ```julia
    julia> @dynamo roundtrip(m) = IR(m)

    julia> foo(x) = x^2
    foo (generic function with 1 method)

    julia> roundtrip(foo, 5)
    25

    julia> foo(x) = x+1
    foo (generic function with 1 method)

    julia> roundtrip(foo, 5)
    25
    ```

    In order to get the dynamo to see the new definition of `foo`, you can explicitly call `IRTools.refresh()`:

    ```julia
    julia> IRTools.refresh(roundtrip)

    julia> roundtrip(foo, 5)
    6
    ```
