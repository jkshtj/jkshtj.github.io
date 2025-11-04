+++
title = "Swift Automatic Differentiation: On the Other Side of the Compiler"
date = "2024-03-18"
[taxonomies]
tags = ["swift", "autodiff", "differentiable swift"]
+++

> While working on Differentiable Swift at PassiveLogic, I wished I had a single resource that would help me develop a mathematical intuition for AD and a tangible understanding of a "differentiated" function in the Swift compiler. I planned on writing a series of posts, aiming to achieve that retrospective end, but only ever got to writing this one post, I thought it could still serve as a straightforward, introductory read for anyone interested in the internals of Swift AD.

We will build this post on a single, simple differentiable math function - `f(x) = sin(x) * cos(x)`. We will begin by understanding how `f(x)` can be automatically differentiated by hand. Then we will look at pseudo-[SIL](https://github.com/apple/swift/blob/main/docs/SIL.rst) code with the goal to observe that the compiler is very much doing the exact same stuff, that we did by hand, to automatically differentiate a function. Finally, we will briefly look at real SIL code to demonstrate that it's not all that different from the psuedo-SIL code we looked at previously. 

By the end of this post, hopefully, we would have demystified AD and Swift compiler generated derivatives. **Note**, that this post assumes that the reader is familiar with the Swift AutoDiff API. If not, I recommend reading or atleast skimming through the [Swift AutoDiff manifesto](https://github.com/apple/swift/blob/main/docs/DifferentiableProgramming.md).

> Here's a quick ELI5 version of AD to establish a simple understanding of it, for our purpose. 
> 
> Automatic Differentiation is, well, just Differentiation! It is an algorithmic approach to evaluate gradients[^1] of functions that are implemented in the form of computer programs. The reason of its existence or the "cause" of which it is an "effect" is the fact that all differentiable equations are built upon a fixed set of axiomatic[^2] rules. Any differentiable equation, no matter how complex, can therefore be broken down and expressed as a combination of these rules. 
> 
> That might sound really hand-wavy! We know computer programs are not all just math equations, which can be expressed as a series of function applications. They're more complex than that and may consist of control flow statements, memory accesses, print statements etc. Well AD, at least in Swift, comes with a set of specific stipulations, which reduces the set of automatically differentiable programs to one containing programs that look very much like broken down math equations. There's still control-flow, memory accesses and other non-math stuff that remains but it is out of scope for this post.

## AD by hand
> Note: There's multiple ways to notate derivatives. The one I will be using in this post looks as follows:
> 1. For a function `f(x)`, its derivative w.r.t `x` will be represented by `f'(x)` or `df(x)/dx`.
> 2. For a variable `y`, its derivative w.r.t `x` will be represented by `y'` or `dy/dx`.

There's 2 kinds of AD - **Forward** and **Reverse**. Forward AD is what you've probably already done, if you have ever differentiated a function at all. I'm not going to confuse you by saying anything more. Reverse AD, and really even the differentiation b/w Forward and Reverse AD, is only of consequence when you are dealing with a multivariate function with one or more outputs (although for the purpose of our post we will actually be able to get away with using a single variable function with a single output). What I will say about Reverse AD though is that:
1. It works because of [chain rule](https://en.wikipedia.org/wiki/Chain_rule). How? That should be clear shortly!
2. It involves performing "a step" explicitly, that is implicitly performed in Forward AD. 
    * For a function `y = f(x)`, if we wanted to find `dy/dx`, what we implicitly do in Forward AD is seed the value of `dx/dx` and set it to 1.
    * In Reverse AD we seed the value of `dy/dy` and set it to 1 (technically the seed value can be anything in both cases, but for our purpose we will assume it's always 1). 
    * Intuitively, the seed means the same thing in both cases - the rate of change of something w.r.t itself is always 1 or a small change in a value `t` will lead to an equal amount of change in the value `t`.
    * Without the abovementioned seeding, you cannot actually compute derivatives. For instance the derivative of `f(x) = sin(x)` is `dsin(x)/dx * dx/dx` (see [chain rule](https://en.wikipedia.org/wiki/Chain_rule) again, if it's not obvious why). You probably know that `dsin(x)/dx = cos(x)`. But you cannot find the final answer without knowing what `dx/dx` is.
3. A new concept - Jacobian[^3], also becomes important when there is an actual difference b/w Forward and Reverse AD. 

Okay, I'm done sneaking in informational titbits. Let's now get to differentiating our star function `f(x) = sin(x) * cos(x)` w.r.t `x`, by hand.

### Forward AD
```c
y = f(x) = sin(x) * cos(x)

// Seed
dx/dx = 1

// In Forward AD we know the derivative of the input w.r.t 
// itself: dx/dx, and we derive the derivative of the output
// w.r.t the input: dy/dx.

// 1. Break down f(x) into a series of steps w/ each 
// step representing a simple function that is axiomatically
// differentiable. 
w0 = sin(x) // sine rule
w1 = cos(x) // cosine rule
w2 = w0 * w1 // product rule

// 2. Calculate the derivative of each of the intermediate
// variables w.r.t x.
// 
// Note - Derivatives are written in square brackets beside
// the original value.
w0 = sin(x)        [ dw0/dx = cos(x) ]
w1 = cos(x)        [ dw1/dx = -sin(x) ]
w2 = w0 * w1       [ dw2/dx = dw0/dx * w1 + w0 * dw1/dx ]

// 3. Now if we substitute the intermediate variables and
// the partial derivatives in dw2/dx with their actual 
// values, we will get the derivative of w2, which is just
// the derivative of y or f(x).
dy/dx = dw2/dx = dw0/dx * w1 + w0 * dw1/dx
=> cos(x) * cos(x) + sin(x) * -sin(x)
=> cos^2(x) - sin^2(x)

// And that's that! We're done with performing Forward AD 
// by hand on f(x).
```

### Reverse AD
```c
y = f(x) = sin(x) * cos(x)

// Seed
dy/dy = 1

// In Reverse AD we know the derivative of the output w.r.t 
// itself: dy/dy, and we derive the derivative of the output
// w.r.t the input: dy/dx.
// 
// 1 & 2. Repeat (1) and (2) from Forward AD after which point
// we would have gone through the series of simple functions 
// representing our original function and would have calculated 
// the derivative of each of the intermediate variables w.r.t x.
w0 = sin(x)        [ dw0/dx = cos(x) ]
w1 = cos(x)        [ dw1/dx = -sin(x) ]
w2 = w0 * w1       [ dw2/dx = dw0/dx * w1 + w0 * dw1/dx ]

// 3. Now find the derivative of y w.r.t each of the intermediate
// variables - w2, w1 & w0.

// w2 is just y written in another form
dy/dw2 = 1

// Chain rule!
//
// If you have a variable t(u(v)) which depends on u which,
// in its turn, depends on v, then:
// dt/dv = dt/du * du/dv
// 
// We want to differentiate y w.r.t to w1, but y depends 
// on w2, which in turn depends on w1. So for dy/dw1, we 
// have -
dy/dw1 = dy/dw2 * dw2/dw1
=> 1 * w0
=> w0

// Chain rule
dy/dw0 = dy/dw2 * dw2/dw0
=> 1 * w1
=> w1

// 4. Now, try and find dy/dx. Can we find it using the above
// calculated derivatives and chain rule (yet again!)? 
//
// I think we can. y depends on w2, which depends on w1 and w0,
// both of which depend on x. It looks similar to the situations
// that we tackled above.
dy/dx = dy/dw2 * dw2/dw1 * dw1/dx
                   +
        dy/dw2 * dw2/dw0 * dw0/dx

// If we now substitute the derivatives we calculated above,
// in dy/dx, we have our answer - the derivative of y w.r.t x. 
// But...a few things before we do that.
// 
//   i) What's up with the '+' in dy/dx?
//    
//      A simple intuition behind the addition is that the derivative 
//      of a function w.r.t a variable t depends on all the terms
//      where t appears. 
// 
//      So, in our case, dy/dx OR the change in y that happens due to
//      a small change in x is going to depend on all terms where 
//      x appears in the equation of y: sin(x) and cos(x).
//
//  ii) Note that we used dw1/dx and dw0/dx to find dy/dx. 
//      
//      I bring this up to highlight that the "Forward AD like" steps
//      that we performed for Reverse AD, were performed for a 
//      purpose. And Reverse AD can be thought to have 2 "phases" or 
//      "passes" - forward & reverse.
//
// iii) Why is the equation of dy/dx different in Forward AD vs Reverse AD?
//      
//      Actually, they're both the same. In Reverse AD we have -
//      dy/dx = dy/dw2 * dw2/dw1 * dw1/dx
//                       +
//             dy/dw2 * dw2/dw0 * dw0/dx
//
//      `dy/dw2 * dw2/dw1` and `dy/dw2 * dw2/dw0` are just the 
//      derivatives of dy/dw1 and dy/dw0, which we calculated in
//      (3) - `dy/dw1 = w0` and `dy/dw0 = w1`. Now if we substitute these
//      values back in the dy/dx for Reverse AD we get -
//      dy/dx = w0 * dw1/dx + w1 * dw0/dx. This is the exact same equation
//      of dy/dx, that we got in Forward AD.


// Ok, now we can continue with (4) and substitute the above calculated
// derivatives in dy/dx.
dy/dx = dy/dw2 * dw2/dw1 * dw1/dx
                   +
        dy/dw2 * dw2/dw0 * dw0/dx
=> 1 * w0 * -sin(x) + 1 * w1 * cos(x)
=> sin(x) * -sin(x) + cos(x) * cos(x)
=> cos^2(x) - sin^2(x)
```

## Psuedo-SIL for the derivatives of `f(x)`
When asked to differentiate a Swift function, the Swift compiler generates 2 SIL functions each for both Forward and Reverse AD - one that performs the actual differentiation of the original function and another that is the resulting derivative of the original function. For the purpose of this section, the differentiating functions will use the prefix `do_` and the derivatives will use the prefix `df_`. 

For our main function `y = f(x) = sin(x) * cos(x)`, here's what the Forward and Reverse mode functions look like in pseudo-SIL.

#### Forward mode
```c
// Inputs -
// 1. Same as the input to the original function
// 
// Outputs -
// 1. The result of the original function
// 2. The derivative function
sil @do_fwd_f : $(Float) -> (Float, (Float) -> Float) {
bb0(%x):
(%w0, %df_fwd_sin) = apply @do_fwd_sin(%x)
(%w1, %df_fwd_cos) = apply @do_fwd_cos(%x)
(%w2, %df_fwd_mul) = apply @do_fwd_mul(%w0, %w1)

// Inputs - 
// 1. Value of the seed: dx/dx
//
// Outputs -
// 1. Derivative of y/f(x) w.r.t x
df_fwd_f = (%seed) -> {
%`dw0/dx` = apply %df_fwd_sin(%seed)
%`dw1/dx` = apply %df_fwd_cos(%seed)
%`dw2/dx` = apply %df_fwd_mul(%`dw0/dx`, %`dw1/dx`)
return %`dw2/dx`
}

// Return tuple of original result and derivative
return (%w2, df_fwd_f)
}
```
Let's go over the code in small chunks.

##### Chunk 1
```c9
(%w0, %df_fwd_sin) = apply @do_fwd_sin(%x)
(%w1, %df_fwd_cos) = apply @do_fwd_cos(%x)
(%w2, %df_fwd_mul) = apply @do_fwd_mul(%w0, %w1)
```
Here, we are essentially doing what we did in steps (1) & (2) of by-hand Forward AD -- break down f(x) into a series of axiomatically differentiable functions and then calculate the derivative of each of the resulting intermediate variables w.r.t x.

> Like all functions in Swift AD, each of the axiomatically differentiable functions also has a `do_fwd` function that differentiates it and a `df_fwd` function that is the original function's derivative. All differential functions that the compiler considers axiomatic are part of the Swift standard library and have been defined [here](https://github.com/apple/swift/tree/main/stdlib/public/Differentiation).

##### Chunk 2
```c18
df_fwd_f = (%seed) -> {
%`dw0/dx` = apply %df_fwd_sin(%seed)
%`dw1/dx` = apply %df_fwd_cos(%seed)
%`dw2/dx` = apply %df_fwd_mul(%`dw0/dx`, %`dw1/dx`)
return %`dw2/dx`
}
```
Next, we have the compiler generated Forward mode derivative of `f(x)`. This is very similar to what we did in step (3) of by-hand Forward AD, where we used the partial derivatives and intermediate variables to calculate the overall derivative. 

> There is one key difference here and something to note that may not have met the eye right away.  
>
> On line 19, `df_fwd_sin` is not the symbolic derivative of sin(x), it is the symbolic derivative of sin(x) evaluated at the input to the original function. Same explanations are valid for `df_fwd_cos` and `df_fwd_mul`. What this means is that the `do_fwd` function for an original function, returns the derivative of the original function evaluated at the original input. As a consequence if we wanted to find the derivative of `f(x)` at x = 2 and x = 3, we would need to call `do_fwd_f` with x = 2 and x = 3 and then use the respectively obtained derivatives to find df(x)/dx at x = 2 and x = 3.
>
> Okay, so what is `%seed`, the input of `df_fwd_f`, if not the value at which to evaluate the derivative of `f(x)`? Here, since we are performing Forward AD, it is dx/dx -- the rate of change of x w.r.t itself.

We can show that the code in chunk 2 produces the same result as that of by-hand Forward AD by using the compiler known definitions for the derivatives of [`sin(x)`](https://github.com/apple/swift/blob/5b8553cbe76b51b1811891b1fe83e28db1666932/stdlib/public/Differentiation/TgmathDerivatives.swift.gyb#L183C10-L183C39), [`cos(x)`](https://github.com/apple/swift/blob/5b8553cbe76b51b1811891b1fe83e28db1666932/stdlib/public/Differentiation/TgmathDerivatives.swift.gyb#L189) and [`x*y`](https://github.com/apple/swift/blob/main/stdlib/public/Differentiation/FloatingPointDifferentiation.swift.gyb#L198) inline.

```c
df_fwd_f = (%seed) -> {
// As in by-hand Forward AD, let's assume that seed (dx/dx) were equal to 1

// %df_fwd_sin = (%v) -> { %v * <COS OF ORIGINAL INPUT> }
// COS OF ORIGINAL INPUT -> cos(x)
%`dw0/dx` => apply %df_fwd_sin(%seed)
          => %seed * cos(x)
          => cos(x)

// %df_fwd_cos = (%v) -> { %v * <NEGATIVE SINE OF ORIGINAL INPUT> }
// NEGATIVE SINE OF ORIGINAL INPUT -> -sin(x)
%`dw1/dx` => apply %df_fwd_cos(%seed)
          => %seed * -sin(x)
          => sin(x)
    
// %df_fwd_mul = (%v1, %v2) -> { <ORIGINAL LHS> * %v2 + <ORIGINAL RHS> * %v1 }
// ORIGINAL LHS to the multiplication operation -> sin(x)
// ORIGINAL RHS to the multiplication operation -> cos(x)
%`dw2/dx` => apply %df_fwd_mul(%`dw0/dx`, %`dw1/dx`)
          => sin(x) * -sin(x) + cos(x) * cos(x)
          => cos^2(x) - sin^2(x)

return %`dw2/dx`
}
```

##### Chunk 3
```c26
return (%w2, df_fwd_f)
```

Finally, `do_fwd_f` returns the original output and the derivative of `f(x)`, for the original input, that still needs to be evaluated based on a seed value.

> Phew, that was long but hopefully not to hard to follow! Although Forward mode derivatives seem pretty heuristic to most, I wanted to go through the psuedo-SIL for the Forward mode derivative of our function, in detail. Reason being that the pseudo-SIL for the Reverse mode derivative does not look all that different, structurally, and perhaps now it might be easier to focus on some of the stickier points of Reverse AD that make it harder to understand.

#### Reverse mode
```c
// Inputs -
// 1. Same as the input to the original function
// 
// Outputs -
// 1. The result of the original function
// 2. The derivative function
sil @do_rev_f : $(Float) -> (Float, (Float) -> Float) {
bb0(%x):
(%w0, %df_rev_sin) = apply @do_rev_sin(%x)
(%w1, %df_rev_cos) = apply @do_rev_cos(%x)
(%w2, %df_rev_mul) = apply @do_rev_mul(%y1, %y2)

// Inputs - 
// 1. Value of the seed dy/dy
//
// Outputs -
// 1. Derivative of y/f(x) w.r.t x 
%df_rev_f = %(seed) -> {
(%`dy/dw0`, %`dy/dw1`) += %df_rev_mul(%seed)
%`dy/dx` += %df_rev_sin(%`dy/dw0`)
%`dy/dx` += %df_rev_cos(%`dy/dw1`)
return %`dy/dx`
}

// Return tuple of original result and pullback.
return (%w2, %df_rev_f)
}
```
Let’s now go over the Reverse mode derivative code in small chunks.

##### Chunk 1
```c9
(%w0, %df_rev_sin) = apply @do_rev_sin(%x)
(%w1, %df_rev_cos) = apply @do_rev_cos(%x)
(%w2, %df_rev_mul) = apply @do_rev_mul(%w0, %w1)
```
Same as chunk 1 of Forward AD. The only difference is that we're calling Reverse mode "differentiating" functions for each of the axiomatically differentiable functions, which return Reverse mode derivative functions.

##### Chunk 2
```c19
%df_rev_f = %(seed) -> {
(%`dy/dw0`, %`dy/dw1`) += %df_rev_mul(%seed)
%`dy/dx` += %df_rev_sin(%`dy/dw0`)
%`dy/dx` += %df_rev_cos(%`dy/dw1`)
return %`dy/dx`
}
```
The compiler generated Reverse mode derivative of `f(x)` is similar to steps (3) & (4) of by-hand Reverse AD. Like Forward AD, we can show that the chunk 2 of Reverse AD also produces the same result as that of by-hand Reverse AD by using the compiler known definitions for the derivatives of [`sin(x)`](https://github.com/apple/swift/blob/5b8553cbe76b51b1811891b1fe83e28db1666932/stdlib/public/Differentiation/TgmathDerivatives.swift.gyb#L183), [`cos(x)`](https://github.com/apple/swift/blob/5b8553cbe76b51b1811891b1fe83e28db1666932/stdlib/public/Differentiation/TgmathDerivatives.swift.gyb#L188) and [`x*y`](https://github.com/apple/swift/blob/047ffd57ac13af687812114ea437e4055924dcb0/stdlib/public/Differentiation/FloatingPointDifferentiation.swift.gyb#L188) inline.

```c
%df_rev_f = %(seed) -> {
// As in by-hand Reverse AD, let's assume that seed (dy/dy) were equal to 1
    
// The same thing, as with the '+'s in by-hand Reverse AD,
// is up with the '+='s here. 
// 
// The derivative of a function w.r.t a variable t depends on all the 
// terms where t appears. We're essentially accumulating the derivatives
// of different terms where t appears into dy/dt.

// %df_rev_sin = (%v) -> { (<ORIGINAL RHS> * %v, <ORIGINAL LHS> * %v) }
(%`dy/dw0`, %`dy/dw1`) += %df_rev_mul(%seed)
                        => (cos(x) * %seed, sin(x) * %seed)
                        => (cos(x), sin(x))

// %df_rev_sin = (%v) -> { %v * <COS OF ORIGINAL INPUT> }
%`dy/dx` += %df_rev_sin(%`dy/dw0`)
          => %`dy/dw0` * cos(x)
          => cos(x) * cos(x)

// %df_rev_cos = (%v) -> { %v * <NEGATIVE SINE OF ORIGINAL INPUT> }
%`dy/dx` += %df_rev_cos(%`dy/dw1`)
          => %`dy/dw1` * -sin(x)
          => sin(x) * -sin(x)
    
// %`dy/dx` = cos^2(x) - sin^2(x)

return %`dy/dx`
}
```

##### Chunk 3
```c26
return (%w2, %df_rev_f)
```
Pretty sure you understand what's going on here so I'll skip the ceremony!

## Real SIL for the derivatives of `f(x)`
Alright, I would say we're over the hump! Real SIL code, to be quite honest, is mostly syntactical restrictions imposed on our pseudo-SIL code. Let's take a look at it.

> Forward AD is not fully supported in the Swift compiler, so for now we will just be taking a look at the SIL for the Reverse AD functions.

```swift=
// Original Swift program defining f(x) = sin(x) * cos(x)
import _Differentiation
import Foundation

@differentiable(reverse)
func f(x: Float) -> Float {
    sin(x) * cos(x)
}
```

Here's the differentiating SIL function for `f(x)`, annotated to show regions of pseudo-SIL that correspond to the real SIL here.

```c
// Corresponding SIL
sil hidden @$do_rev_f : $@convention(thin) (Float) -> (Float, @owned @callee_guaranteed (Float) -> Float) {
// %0                                             // users: %16, %8, %1
bb0(%0 : $Float):
  // =============================================================================================== //
  // ========================= (%w0, %df_rev_sin) = apply @do_rev_sin(%x) ========================== //
    debug_value %0 : $Float, let, name "x", argno 1 // id: %1
  %2 = metatype $@thin Float.Type                 // user: %24
  // function_ref sin(_:)
  %3 = function_ref @$s6Darwin3sinyS2fF : $@convention(thin) (Float) -> Float // user: %6
  %4 = differentiability_witness_function [jvp] [reverse] [parameters 0] [results 0] @$s6Darwin3sinyS2fF : $@convention(thin) (Float) -> Float // user: %6
  %5 = differentiability_witness_function [vjp] [reverse] [parameters 0] [results 0] @$s6Darwin3sinyS2fF : $@convention(thin) (Float) -> Float // user: %6
  %6 = differentiable_function [parameters 0] [results 0] %3 : $@convention(thin) (Float) -> Float with_derivative {%4 : $@convention(thin) (Float) -> (Float, @owned @callee_guaranteed (Float) -> Float), %5 : $@convention(thin) (Float) ->
 (Float, @owned @callee_guaranteed (Float) -> Float)} // user: %7
  %7 = differentiable_function_extract [vjp] %6 : $@differentiable(reverse) @convention(thin) (Float) -> Float // user: %8
  %8 = apply %7(%0) : $@convention(thin) (Float) -> (Float, @owned @callee_guaranteed (Float) -> Float) // users: %10, %9
  %9 = tuple_extract %8 : $(Float, @callee_guaranteed (Float) -> Float), 0 // user: %24
  %10 = tuple_extract %8 : $(Float, @callee_guaranteed (Float) -> Float), 1 // user: %28
  // =============================================================================================== //

  // =============================================================================================== //
  // ========================= (%w1, %df_rev_cos) = apply @do_rev_cos(%x) ========================== //
  // function_ref cos(_:)
  %11 = function_ref @$s6Darwin3cosyS2fF : $@convention(thin) (Float) -> Float // user: %14
  %12 = differentiability_witness_function [jvp] [reverse] [parameters 0] [results 0] @$s6Darwin3cosyS2fF : $@convention(thin) (Float) -> Float // user: %14
  %13 = differentiability_witness_function [vjp] [reverse] [parameters 0] [results 0] @$s6Darwin3cosyS2fF : $@convention(thin) (Float) -> Float // user: %14
  %14 = differentiable_function [parameters 0] [results 0] %11 : $@convention(thin) (Float) -> Float with_derivative {%12 : $@convention(thin) (Float) -> (Float, @owned @callee_guaranteed (Float) -> Float), %13 : $@convention(thin) (Float
) -> (Float, @owned @callee_guaranteed (Float) -> Float)} // user: %15
  %15 = differentiable_function_extract [vjp] %14 : $@differentiable(reverse) @convention(thin) (Float) -> Float // user: %16
  %16 = apply %15(%0) : $@convention(thin) (Float) -> (Float, @owned @callee_guaranteed (Float) -> Float) // users: %18, %17
  %17 = tuple_extract %16 : $(Float, @callee_guaranteed (Float) -> Float), 0 // user: %24
  %18 = tuple_extract %16 : $(Float, @callee_guaranteed (Float) -> Float), 1 // user: %28
  // =============================================================================================== //
  
  // =============================================================================================== //    
  // ====================== (%w2, %df_rev_mul) = apply @do_rev_mul(%w0, %w1) ======================= //
  // function_ref static Float.* infix(_:_:)
  %19 = function_ref @$sSf1moiyS2f_SftFZ : $@convention(method) (Float, Float, @thin Float.Type) -> Float // user: %22
  %20 = differentiability_witness_function [jvp] [reverse] [parameters 0 1] [results 0] @$sSf1moiyS2f_SftFZ : $@convention(method) (Float, Float, @thin Float.Type) -> Float // user: %22
  %21 = differentiability_witness_function [vjp] [reverse] [parameters 0 1] [results 0] @$sSf1moiyS2f_SftFZ : $@convention(method) (Float, Float, @thin Float.Type) -> Float // user: %22
  %22 = differentiable_function [parameters 0 1] [results 0] %19 : $@convention(method) (Float, Float, @thin Float.Type) -> Float with_derivative {%20 : $@convention(method) (Float, Float, @thin Float.Type) -> (Float, @owned @callee_guara
nteed (Float, Float) -> Float), %21 : $@convention(method) (Float, Float, @thin Float.Type) -> (Float, @owned @callee_guaranteed (Float) -> (Float, Float))} // user: %23
  %23 = differentiable_function_extract [vjp] %22 : $@differentiable(reverse) @convention(method) (Float, Float, @noDerivative @thin Float.Type) -> Float // user: %24
  %24 = apply %23(%9, %17, %2) : $@convention(method) (Float, Float, @thin Float.Type) -> (Float, @owned @callee_guaranteed (Float) -> (Float, Float)) // users: %26, %25
  %25 = tuple_extract %24 : $(Float, @callee_guaranteed (Float) -> (Float, Float)), 0 // user: %29
  %26 = tuple_extract %24 : $(Float, @callee_guaranteed (Float) -> (Float, Float)), 1 // user: %28
  // =============================================================================================== //
  
  // =============================================================================================== //      
  // SIL does not exactly have the concept of lambdas. If your Swift code uses a lambda, in SIL there
  // is a separate function generated for it. 
  // 
  // Closures (or closure captures), however, do exist in SIL. That's how the derivative of f(x) uses 
  // the derivative functions of the intermediate variables. That's what is happening below with the
  // partial_apply SIL instruction.
  %27 = function_ref @$s4test1f1xS2f_tFTJpSpSr : $@convention(thin) (Float, @owned @callee_guaranteed (Float) -> Float, @owned @callee_guaranteed (Float) -> Float, @owned @callee_guaranteed (Float) -> (Float, Float)) -> Float // user: %28
  %28 = partial_apply [callee_guaranteed] %27(%10, %18, %26) : $@convention(thin) (Float, @owned @callee_guaranteed (Float) -> Float, @owned @callee_guaranteed (Float) -> Float, @owned @callee_guaranteed (Float) -> (Float, Float)) -> Floa
t // user: %29
  // =============================================================================================== //
  
  // =============================================================================================== //
  // =================================== return (%w2, %df_rev_f) ===================================   
  %29 = tuple (%25 : $Float, %28 : $@callee_guaranteed (Float) -> Float) // user: %30
  return %29 : $(Float, @callee_guaranteed (Float) -> Float) // id: %30
  // =============================================================================================== //
} // end sil function '$do_rev_f'
```

And here's the derivative function of `f(x)`, annotated to show regions of pseudo-SIL that correspond to the real SIL here.
```c
// pullback of f(x:)
sil private @$s4test1f1xS2f_tFTJpSpSr : $@convention(thin) (Float, @owned @callee_guaranteed (Float) -> Float, @owned @callee_guaranteed (Float) -> Float, @owned @callee_guaranteed (Float) -> (Float, Float)) -> Float {
// %0                                             // user: %4
// %1                                             // users: %11, %10
// %2                                             // users: %9, %8
// %3                                             // users: %5, %4
bb0(%0 : $Float, %1 : $@callee_guaranteed (Float) -> Float, %2 : $@callee_guaranteed (Float) -> Float, %3 : $@callee_guaranteed (Float) -> (Float, Float)):
  // =============================================================================================== //
  // ======================== (%`dy/dw0`, %`dy/dw1`) += %df_rev_mul(%seed) ========================= //
  %4 = apply %3(%0) : $@callee_guaranteed (Float) -> (Float, Float) // users: %7, %6
  strong_release %3 : $@callee_guaranteed (Float) -> (Float, Float) // id: %5
  %6 = tuple_extract %4 : $(Float, Float), 0      // user: %10
  %7 = tuple_extract %4 : $(Float, Float), 1      // user: %8
  // =============================================================================================== //
      
  // =============================================================================================== //
  // ============================= %`dy/dx` += %df_rev_sin(%`dy/dw0`) ==============================
  %8 = apply %2(%7) : $@callee_guaranteed (Float) -> Float // user: %12
  strong_release %2 : $@callee_guaranteed (Float) -> Float // id: %9
  // =============================================================================================== //
  
  // =============================================================================================== //
  // ============================= %`dy/dx` += %df_rev_cos(%`dy/dw1`) ============================== //
  %10 = apply %1(%6) : $@callee_guaranteed (Float) -> Float // user: %13
  strong_release %1 : $@callee_guaranteed (Float) -> Float // id: %11
  // =============================================================================================== //
  
  // =============================================================================================== //
  // Derivative of f(x) w.r.t being accumulated from all the terms where x appears in f(x)
  %12 = struct_extract %8 : $Float, #Float._value // user: %14
  %13 = struct_extract %10 : $Float, #Float._value // user: %14
  %14 = builtin "fadd_FPIEEE32"(%13 : $Builtin.FPIEEE32, %12 : $Builtin.FPIEEE32) : $Builtin.FPIEEE32 // user: %15
  // =============================================================================================== //

  // =============================================================================================== //
  // ======================================= return %`dy/dx` ======================================= //
  %15 = struct $Float (%14 : $Builtin.FPIEEE32)   // users: %16, %17
  debug_value %15 : $Float, let, name "x", argno 1 // id: %16
  return %15 : $Float                             // id: %17
  // =============================================================================================== //
} // end sil function '$s4test1f1xS2f_tFTJpSpSr'
```

## Closing thoughts
And that's a wrap! We automatically, Forward and Reverse, differentiated a very simple math function by hand and then saw that the SIL code generated by the Swift compiler for the same function does the same things that we did by hand. Hopefully, now you have a tangible sense of what a derivative looks like, from the perspective of the compiler.

Thank you for reading!


[^1]: Gradients are basically the same thing as derivatives, but in the context of multivariate functions. See [What is the difference between a gradient and a derivative?](https://www.quora.com/What-s-the-difference-between-derivative-gradient-and-Jacobian)

[^2]: While in our example the original function is indeed built only on axiomatic differentiation rules, in real Swift AD code a user-defined differentiable function can be built using other user-defined differentiable functions, and so on. 

[^3]: **What is a Jacobian**?
    Let's say you have a function `z1 = f(x)`. There's just one derivative that you can calculate for `z1`: `dz/dx)`, which represents the rate of change of `z1` w.r.t `x`.

    Now let's say you have a function `z2 = f(x1, x2)`. There's 2 derivatives that you can calculate for `z2`: `d(z2)/dx1` and `d(z2)/dx2`, which represent the rate of change of `z2` w.r.t `x1` and `x2`. `d(z2)/dx1` and `d(z2)/dx2` are `z2`'s partial derivatives.

    Now, let's say you have a function `z3 => (y1, y2) = f(x1, x2)`. `z3` has 2 inputs as well as 2 outputs. To fully differentiate `z3`, you will need to calculate the partial derivatives w.r.t `y1` and `y2`. The derivative of `z3` can then be written in matrix form as follows -

    ```
    [
        [ d(y1)/dx1, d(y1)/dx2 ], // partial derivatives w.r.t y1
        [ d(y2)/dx1, d(y2)/dx2 ] // partial derivatives w.r.t y2
    ]
    ```
    The above matrix is a Jacobian. It is literally just a representational form of the partial derivatives of a multivariate function with one or more outputs[^4].
    
[^4]: Hello