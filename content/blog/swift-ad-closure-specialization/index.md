+++
title = "[Design] Autodiff Pullback Closure Specialization Optimization"
date = "2023-11-03"
[taxonomies]
tags = ["swift", "autodiff", "differentiable swift", "closure specialization"]
+++

> Design for the Swift AD closure specialization optimization that was implemented in [this](https://github.com/swiftlang/swift/pull/71775/) commit.

### What is the closure specialization optimization?
In Swift, the general closure specialization optimization can help alleviate heap allocation costs associated to closures by eliminating usage of closures altogether, in certain call-sites.

Given a function call-site, if the callee takes a closure as an input argument and then calls the closure in its body, we can eliminate the overhead of the closure context's heap allocation by -

1. Cloning the original callee. 
2. Moving the closure creation (the partial_apply instruction) from the body of the caller to the body of the cloned callee (where the corresponding apply of the closure lives).
3. Changing the cloned callee's signature to take the partially applied arguments, instead of the closure.
4. Modifing the apply site in the caller to call the specialized callee.
5. Getting rid of the closure creation (the partial_apply now in the cloned callee) altogether via peephole optimizations.

Using this optimization, SIL code like this:
```
sil [noinline] @takes_closure : $@convention(thin) (Int, @owned @callee_owned (Int) -> Int) -> () {
bb0(%0: $Int, %1: $@callee_owned (Int) -> Int):
  %4 = apply %1(%0) : $@callee_owned (Int) -> Int
  %9999 = tuple()
  return %9999 : $()
}

sil [noinline] @partially_applies : $@convention(thin) (Int) -> () {
bb0(%0 : $Int):
  // `multiplies_two_ints` is a defined function
  %1 = function_ref @multiplies_two_ints : $@convention(thin) (Int, Int) -> Int
  // Create a closure out of `multiplies_two_ints` by partially 
  // applying one of the integer arguments.
  %2 = partial_apply [callee_guaranteed] %1(%0) : $@convention(thin) (Int, Int) -> Int
  %3 = convert_function %2 : $@callee_guaranteed (Int) -> Int to $@callee_owned (Int) -> Int
  // Pass the closure created out of `multiplies_two_ints` to `takes_closure`
  %4 = function_ref @takes_closure : $@convention(thin) (Int, @owned @callee_owned (Int) -> Int) -> ()
  %5 = apply %4(%0, %3) : $@convention(thin) (Int, @owned @callee_owned (Int) -> Int) -> ()
  %9999 = tuple()
  return %9999 : $()
}

```

Will look like this:
```
sil shared [noinline] @specialized_takes_closure : $@convention(thin) (Int, Int) -> () {
// %0                                             // user: %5
// %1                                             // user: %3
bb0(%0 : $Int, %1 : $Int):
  // function_ref multiplies_two_ints
  %2 = function_ref @multiplies_two_ints : $@convention(thin) (Int, Int) -> Int // user: %3
  // This partial apply can be optimized away altogether via peephole optimizations
  // since the corresponding apply is also in the same block at %5.
  %3 = partial_apply [callee_guaranteed] %2(%1) : $@convention(thin) (Int, Int) -> Int // user: %4
  %4 = convert_function %3 : $@callee_guaranteed (Int) -> Int to $@callee_owned (Int) -> Int // user: %5
  %5 = apply %4(%0) : $@callee_owned (Int) -> Int
  %6 = tuple ()                                   // user: %7
  return %6 : $()                                 // id: %7
}

sil [noinline] @partially_applies : $@convention(thin) (Int) -> () {
// %0                                             // users: %6, %6, %3
bb0(%0 : $Int):
  %1 = function_ref @$s13takes_closure19multiplies_two_intsSiTf1nc_n : $@convention(thin) (Int, Int) -> () // user: %6
  // `takes_closure` has been specialized and now takes the originally closed over
  // arguments instead of the closure.
  %2 = apply %1(%0, %0) : $@convention(thin) (Int, Int) -> ()
  %3 = tuple ()                                   // user: %9
  return %3 : $()                                 // id: %9
}
```

See [here](https://github.com/apple/swift/blob/a74c02d84093b2dcb0f51a0849440f3920b1fa52/lib/SILPasses/ClosureSpecializer.cpp#L15) for a high-level description of the general Swift closure specialization optimization.


### Limitations of the general optimization and need for an AD specific optimization

The general closure specialization optimization only works on actual apply-sites and for closures that are directly passed as arguments. That will be enough if both the VJP and the pullback have been inlined into the same function and the pullback takes intermediate closures directly. 

However, pullbacks with control flow (loopy pullbacks are excluded from the discussion) do not receive the benefit of this optimization because they may receive closures opaquely - wrapped inside of branch tracing enums. Additionally, inlining stops working based on the sizes of the function being inlined into and the inlinee. But we want to get rid of the memory allocation overhead associated with closures in Swift AD, most of the time.

With the AD specific closure specialization optimization we would be able to alleviate heap allocation costs by turning the SIL for the below code -
```swift=
// Original Swift code
@differentiable(reverse)
func foo(x: Float) -> Float {
  if (x > 5) {
    return sin(x) * cos(x)
  } else {
    return sin(x) + cos(x)
  }
}

@inline(never)
func bar() -> Float {
  let (_, df) = valueWithPullback(at: Float(4), of: foo)
  let r = df(Float(4))
  return r
}
```

```c
// Generated SIL code

enum _AD__$s4main3foo1xS2f_tF_bb0__Pred__src_0_wrt_0 {
}

enum _AD__$s4main3foo1xS2f_tF_bb1__Pred__src_0_wrt_0 {
  case bb0(())
}

enum _AD__$s4main3foo1xS2f_tF_bb2__Pred__src_0_wrt_0 {
  case bb0(())
}

enum _AD__$s4main3foo1xS2f_tF_bb3__Pred__src_0_wrt_0 {
  case bb2((predecessor: _AD__$s4main3foo1xS2f_tF_bb2__Pred__src_0_wrt_0, (Float) -> Float, (Float) -> Float, (Float) -> (Float, Float)))
  case bb1((predecessor: _AD__$s4main3foo1xS2f_tF_bb1__Pred__src_0_wrt_0, (Float) -> Float, (Float) -> Float, (Float) -> (Float, Float)))
}

// reverse-mode derivative of foo(x:)
sil hidden @$s4main3foo1xS2f_tFTJrSpSr : $@convention(thin) (Float) -> (Float, @owned @callee_guaranteed (Float) -> Float) {
[global: ]
// %0                                             // users: %11, %15, %26, %29, %2, %1
bb0(%0 : $Float):
  debug_value %0 : $Float, let, name "x", argno 1 // id: %1
  %2 = struct_extract %0 : $Float, #Float._value  // users: %27, %24, %12, %8, %4
  %3 = float_literal $Builtin.FPIEEE32, 0x40A00000 // 5 // user: %4
  %4 = builtin "fcmp_olt_FPIEEE32"(%3 : $Builtin.FPIEEE32, %2 : $Builtin.FPIEEE32) : $Builtin.Int1 // user: %6
  %5 = tuple ()                                   // users: %23, %7
  cond_br %4, bb1, bb2                            // id: %6

bb1:                                              // Preds: bb0
  %7 = enum $_AD__$s4main3foo1xS2f_tF_bb1__Pred__src_0_wrt_0, #_AD__$s4main3foo1xS2f_tF_bb1__Pred__src_0_wrt_0.bb0!enumelt, %5 : $() // user: %20
  %8 = builtin "int_sin_FPIEEE32"(%2 : $Builtin.FPIEEE32) : $Builtin.FPIEEE32 // users: %16, %9
  %9 = struct $Float (%8 : $Builtin.FPIEEE32)     // user: %19
  // function_ref closure #1 in _vjpSin(_:)
  %10 = function_ref @$s16_Differentiation7_vjpSinySf5value_S2fc8pullbacktSfFS2fcfU_ : $@convention(thin) (Float, Float) -> Float // user: %11
  %11 = partial_apply [callee_guaranteed] %10(%0) : $@convention(thin) (Float, Float) -> Float // user: %20
  %12 = builtin "int_cos_FPIEEE32"(%2 : $Builtin.FPIEEE32) : $Builtin.FPIEEE32 // users: %16, %13
  %13 = struct $Float (%12 : $Builtin.FPIEEE32)   // user: %19
  // function_ref closure #1 in _vjpCos(_:)
  %14 = function_ref @$s16_Differentiation7_vjpCosySf5value_S2fc8pullbacktSfFS2fcfU_ : $@convention(thin) (Float, Float) -> Float // user: %15
  %15 = partial_apply [callee_guaranteed] %14(%0) : $@convention(thin) (Float, Float) -> Float // user: %20
  %16 = builtin "fmul_FPIEEE32"(%8 : $Builtin.FPIEEE32, %12 : $Builtin.FPIEEE32) : $Builtin.FPIEEE32 // user: %17
  %17 = struct $Float (%16 : $Builtin.FPIEEE32)   // user: %22
  // function_ref closure #1 in static Float._vjpMultiply(lhs:rhs:)
  %18 = function_ref @$sSf16_DifferentiationE12_vjpMultiply3lhs3rhsSf5value_Sf_SftSfc8pullbacktSf_SftFZSf_SftSfcfU_ : $@convention(thin) (Float, Float, Float) -> (Float, Float) // user: %19
  %19 = partial_apply [callee_guaranteed] %18(%13, %9) : $@convention(thin) (Float, Float, Float) -> (Float, Float) // user: %20
  %20 = tuple $(predecessor: _AD__$s4main3foo1xS2f_tF_bb1__Pred__src_0_wrt_0, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> (Float, Float)) (%7, %11, %15, %19) // user: %21
  %21 = enum $_AD__$s4main3foo1xS2f_tF_bb3__Pred__src_0_wrt_0, #_AD__$s4main3foo1xS2f_tF_bb3__Pred__src_0_wrt_0.bb1!enumelt, %20 : $(predecessor: _AD__$s4main3foo1xS2f_tF_bb1__Pred__src_0_wrt_0, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> (Float, Float)) // user: %22
  br bb3(%17 : $Float, %21 : $_AD__$s4main3foo1xS2f_tF_bb3__Pred__src_0_wrt_0) // id: %22

bb2:                                              // Preds: bb0
  %23 = enum $_AD__$s4main3foo1xS2f_tF_bb2__Pred__src_0_wrt_0, #_AD__$s4main3foo1xS2f_tF_bb2__Pred__src_0_wrt_0.bb0!enumelt, %5 : $() // user: %34
  %24 = builtin "int_sin_FPIEEE32"(%2 : $Builtin.FPIEEE32) : $Builtin.FPIEEE32 // user: %30
  // function_ref closure #1 in _vjpSin(_:)
  %25 = function_ref @$s16_Differentiation7_vjpSinySf5value_S2fc8pullbacktSfFS2fcfU_ : $@convention(thin) (Float, Float) -> Float // user: %26
  %26 = partial_apply [callee_guaranteed] %25(%0) : $@convention(thin) (Float, Float) -> Float // user: %34
  %27 = builtin "int_cos_FPIEEE32"(%2 : $Builtin.FPIEEE32) : $Builtin.FPIEEE32 // user: %30
  // function_ref closure #1 in _vjpCos(_:)
  %28 = function_ref @$s16_Differentiation7_vjpCosySf5value_S2fc8pullbacktSfFS2fcfU_ : $@convention(thin) (Float, Float) -> Float // user: %29
  %29 = partial_apply [callee_guaranteed] %28(%0) : $@convention(thin) (Float, Float) -> Float // user: %34
  %30 = builtin "fadd_FPIEEE32"(%24 : $Builtin.FPIEEE32, %27 : $Builtin.FPIEEE32) : $Builtin.FPIEEE32 // user: %31
  %31 = struct $Float (%30 : $Builtin.FPIEEE32)   // user: %36
  // function_ref closure #1 in static Float._vjpAdd(lhs:rhs:)
  %32 = function_ref @$sSf16_DifferentiationE7_vjpAdd3lhs3rhsSf5value_Sf_SftSfc8pullbacktSf_SftFZSf_SftSfcfU_ : $@convention(thin) (Float) -> (Float, Float) // user: %33
  %33 = thin_to_thick_function %32 : $@convention(thin) (Float) -> (Float, Float) to $@callee_guaranteed (Float) -> (Float, Float) // user: %34
  %34 = tuple $(predecessor: _AD__$s4main3foo1xS2f_tF_bb2__Pred__src_0_wrt_0, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> (Float, Float)) (%23, %26, %29, %33) // user: %35
  %35 = enum $_AD__$s4main3foo1xS2f_tF_bb3__Pred__src_0_wrt_0, #_AD__$s4main3foo1xS2f_tF_bb3__Pred__src_0_wrt_0.bb2!enumelt, %34 : $(predecessor: _AD__$s4main3foo1xS2f_tF_bb2__Pred__src_0_wrt_0, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> (Float, Float)) // user: %36
  br bb3(%31 : $Float, %35 : $_AD__$s4main3foo1xS2f_tF_bb3__Pred__src_0_wrt_0) // id: %36

// %37                                            // user: %41
// %38                                            // user: %40
bb3(%37 : $Float, %38 : $_AD__$s4main3foo1xS2f_tF_bb3__Pred__src_0_wrt_0): // Preds: bb2 bb1
  // function_ref pullback of foo(x:)
  %39 = function_ref @$s4main3foo1xS2f_tFTJpSpSr : $@convention(thin) (Float, @owned _AD__$s4main3foo1xS2f_tF_bb3__Pred__src_0_wrt_0) -> Float // user: %40
  %40 = partial_apply [callee_guaranteed] %39(%38) : $@convention(thin) (Float, @owned _AD__$s4main3foo1xS2f_tF_bb3__Pred__src_0_wrt_0) -> Float // user: %41
  %41 = tuple (%37 : $Float, %40 : $@callee_guaranteed (Float) -> Float) // user: %42
  return %41 : $(Float, @callee_guaranteed (Float) -> Float) // id: %42
} // end sil function '$s4main3foo1xS2f_tFTJrSpSr'

// pullback of foo(x:)
sil private @$s4main3foo1xS2f_tFTJpSpSr : $@convention(thin) (Float, @owned _AD__$s4main3foo1xS2f_tF_bb3__Pred__src_0_wrt_0) -> Float {
[%0: read v**.c*.v**, write v**.c*.v**, copy v**.c*.v**, destroy v**.c*.v**]
[%1: read v**.c*.v**, write v**.c*.v**, copy v**.c*.v**, destroy v**.c*.v**]
[global: read,write,copy,destroy,allocate,deinit_barrier]
// %0                                             // users: %35, %10
// %1                                             // user: %5
bb0(%0 : $Float, %1 : $_AD__$s4main3foo1xS2f_tF_bb3__Pred__src_0_wrt_0):
  %2 = integer_literal $Builtin.Int64, 0          // user: %3
  %3 = builtin "sitofp_Int64_FPIEEE32"(%2 : $Builtin.Int64) : $Builtin.FPIEEE32 // users: %43, %40, %18, %15, %4, %48, %23
  debug_value %3 : $Builtin.FPIEEE32, let, name "x", argno 1, type $Float, expr op_fragment:#Float._value // id: %4
  switch_enum %1 : $_AD__$s4main3foo1xS2f_tF_bb3__Pred__src_0_wrt_0, case #_AD__$s4main3foo1xS2f_tF_bb3__Pred__src_0_wrt_0.bb2!enumelt: bb1, case #_AD__$s4main3foo1xS2f_tF_bb3__Pred__src_0_wrt_0.bb1!enumelt: bb2 // id: %5

// %6                                             // users: %9, %8, %7
bb1(%6 : $(predecessor: _AD__$s4main3foo1xS2f_tF_bb2__Pred__src_0_wrt_0, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> (Float, Float))): // Preds: bb0
  %7 = tuple_extract %6 : $(predecessor: _AD__$s4main3foo1xS2f_tF_bb2__Pred__src_0_wrt_0, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> (Float, Float)), 1 // users: %25, %24
  %8 = tuple_extract %6 : $(predecessor: _AD__$s4main3foo1xS2f_tF_bb2__Pred__src_0_wrt_0, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> (Float, Float)), 2 // users: %21, %20
  %9 = tuple_extract %6 : $(predecessor: _AD__$s4main3foo1xS2f_tF_bb2__Pred__src_0_wrt_0, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> (Float, Float)), 3 // users: %11, %10
  %10 = apply %9(%0) : $@callee_guaranteed (Float) -> (Float, Float) // users: %13, %12
  strong_release %9 : $@callee_guaranteed (Float) -> (Float, Float) // id: %11
  %12 = tuple_extract %10 : $(Float, Float), 0    // user: %14
  %13 = tuple_extract %10 : $(Float, Float), 1    // user: %17
  %14 = struct_extract %12 : $Float, #Float._value // user: %15
  %15 = builtin "fadd_FPIEEE32"(%14 : $Builtin.FPIEEE32, %3 : $Builtin.FPIEEE32) : $Builtin.FPIEEE32 // user: %16
  %16 = struct $Float (%15 : $Builtin.FPIEEE32)   // user: %24
  %17 = struct_extract %13 : $Float, #Float._value // user: %18
  %18 = builtin "fadd_FPIEEE32"(%17 : $Builtin.FPIEEE32, %3 : $Builtin.FPIEEE32) : $Builtin.FPIEEE32 // user: %19
  %19 = struct $Float (%18 : $Builtin.FPIEEE32)   // user: %20
  %20 = apply %8(%19) : $@callee_guaranteed (Float) -> Float // user: %22
  strong_release %8 : $@callee_guaranteed (Float) -> Float // id: %21
  %22 = struct_extract %20 : $Float, #Float._value // user: %23
  %23 = builtin "fadd_FPIEEE32"(%22 : $Builtin.FPIEEE32, %3 : $Builtin.FPIEEE32) : $Builtin.FPIEEE32 // user: %27
  %24 = apply %7(%16) : $@callee_guaranteed (Float) -> Float // user: %26
  strong_release %7 : $@callee_guaranteed (Float) -> Float // id: %25
  %26 = struct_extract %24 : $Float, #Float._value // user: %27
  %27 = builtin "fadd_FPIEEE32"(%26 : $Builtin.FPIEEE32, %23 : $Builtin.FPIEEE32) : $Builtin.FPIEEE32 // user: %28
  %28 = struct $Float (%27 : $Builtin.FPIEEE32)   // users: %30, %29
  debug_value %28 : $Float, let, name "x", argno 1 // id: %29
  br bb3(%28 : $Float)                            // id: %30

// %31                                            // users: %34, %33, %32
bb2(%31 : $(predecessor: _AD__$s4main3foo1xS2f_tF_bb1__Pred__src_0_wrt_0, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> (Float, Float))): // Preds: bb0
  %32 = tuple_extract %31 : $(predecessor: _AD__$s4main3foo1xS2f_tF_bb1__Pred__src_0_wrt_0, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> (Float, Float)), 1 // users: %50, %49
  %33 = tuple_extract %31 : $(predecessor: _AD__$s4main3foo1xS2f_tF_bb1__Pred__src_0_wrt_0, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> (Float, Float)), 2 // users: %46, %45
  %34 = tuple_extract %31 : $(predecessor: _AD__$s4main3foo1xS2f_tF_bb1__Pred__src_0_wrt_0, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> (Float, Float)), 3 // users: %36, %35
  %35 = apply %34(%0) : $@callee_guaranteed (Float) -> (Float, Float) // users: %38, %37
  strong_release %34 : $@callee_guaranteed (Float) -> (Float, Float) // id: %36
  %37 = tuple_extract %35 : $(Float, Float), 0    // user: %39
  %38 = tuple_extract %35 : $(Float, Float), 1    // user: %42
  %39 = struct_extract %37 : $Float, #Float._value // user: %40
  %40 = builtin "fadd_FPIEEE32"(%39 : $Builtin.FPIEEE32, %3 : $Builtin.FPIEEE32) : $Builtin.FPIEEE32 // user: %41
  %41 = struct $Float (%40 : $Builtin.FPIEEE32)   // user: %49
  %42 = struct_extract %38 : $Float, #Float._value // user: %43
  %43 = builtin "fadd_FPIEEE32"(%42 : $Builtin.FPIEEE32, %3 : $Builtin.FPIEEE32) : $Builtin.FPIEEE32 // user: %44
  %44 = struct $Float (%43 : $Builtin.FPIEEE32)   // user: %45
  %45 = apply %33(%44) : $@callee_guaranteed (Float) -> Float // user: %47
  strong_release %33 : $@callee_guaranteed (Float) -> Float // id: %46
  %47 = struct_extract %45 : $Float, #Float._value // user: %48
  %48 = builtin "fadd_FPIEEE32"(%47 : $Builtin.FPIEEE32, %3 : $Builtin.FPIEEE32) : $Builtin.FPIEEE32 // user: %52
  %49 = apply %32(%41) : $@callee_guaranteed (Float) -> Float // user: %51
  strong_release %32 : $@callee_guaranteed (Float) -> Float // id: %50
  %51 = struct_extract %49 : $Float, #Float._value // user: %52
  %52 = builtin "fadd_FPIEEE32"(%51 : $Builtin.FPIEEE32, %48 : $Builtin.FPIEEE32) : $Builtin.FPIEEE32 // user: %53
  %53 = struct $Float (%52 : $Builtin.FPIEEE32)   // users: %55, %54
  debug_value %53 : $Float, let, name "x", argno 1 // id: %54
  br bb3(%53 : $Float)                            // id: %55

// %56                                            // users: %58, %57
bb3(%56 : $Float):                                // Preds: bb1 bb2
  debug_value %56 : $Float, let, name "x", argno 1 // id: %57
  return %56 : $Float                             // id: %58
} // end sil function '$s4main3foo1xS2f_tFTJpSpSr'
```

Into something like the following -
```c
enum _AD__$s4main3foo1xS2f_tF_bb0__Pred__src_0_wrt_0 {
}

enum _AD__$s4main3foo1xS2f_tF_bb1__Pred__src_0_wrt_0 {
  case bb0(())
}

enum _AD__$s4main3foo1xS2f_tF_bb2__Pred__src_0_wrt_0 {
  case bb0(())
}

enum _AD__$s4main3foo1xS2f_tF_bb3__Pred__src_0_wrt_0 {
  case bb2((predecessor: _AD__$s4main3foo1xS2f_tF_bb2__Pred__src_0_wrt_0, Float, Float)
  case bb1((predecessor: _AD__$s4main3foo1xS2f_tF_bb1__Pred__src_0_wrt_0, Float, Float, (Float, Float))
}

// bar()
sil hidden @$s4main3foo1xS2f_tFTJrSpSr : $@convention(thin) (Float) -> (Float, @owned @callee_guaranteed (Float) -> Float) {
[global: read,write,copy,destroy,allocate,deinit_barrier]
bb0:
  %0 = float_literal $Builtin.FPIEEE32, 0x40800000 // 4 
  %1 = struct $Float (%0 : $Builtin.FPIEEE32)     
  debug_value %1 : $Float, let, name "x", argno 1 // id: %2
  %3 = float_literal $Builtin.FPIEEE32, 0x40A00000 // 5 
  %4 = builtin "fcmp_olt_FPIEEE32"(%3 : $Builtin.FPIEEE32, %0 : $Builtin.FPIEEE32) : $Builtin.Int1 
  %5 = tuple ()                                   
  cond_br %4, bb1, bb2                            // id: %6

bb1:                                              // Preds: bb0
  %7 = enum $_AD__$s4main3foo1xS2f_tF_bb1__Pred__src_0_wrt_0, #_AD__$s4main3foo1xS2f_tF_bb1__Pred__src_0_wrt_0.bb0!enumelt, %5 : $() 
  %8 = builtin "int_sin_FPIEEE32"(%0 : $Builtin.FPIEEE32) : $Builtin.FPIEEE32 
  %9 = struct $Float (%8 : $Builtin.FPIEEE32)
  %10 = builtin "int_cos_FPIEEE32"(%0 : $Builtin.FPIEEE32) : $Builtin.FPIEEE32 
  %11 = struct $Float (%12 : $Builtin.FPIEEE32)   
  %12 = tuple $(Float, Float): (%9, %10)
  %13 = tuple $(predecessor: _AD__$s4main3foo1xS2f_tF_bb1__Pred__src_0_wrt_0, Float, Float, (Float, Float)) (%7, %1, %1, %12) 
  %14 = enum $_AD__$s4main3foo1xS2f_tF_bb3__Pred__src_0_wrt_0, #_AD__$s4main3foo1xS2f_tF_bb3__Pred__src_0_wrt_0.bb1!enumelt, %13 : $(predecessor: _AD__$s4main3foo1xS2f_tF_bb1__Pred__src_0_wrt_0, Float, Float, (Float, Float)) 
  br bb3(%14 : $_AD__$s4main3foo1xS2f_tF_bb3__Pred__src_0_wrt_0) // id: %20

bb2:                                              // Preds: bb0
  %15 = enum $_AD__$s4main3foo1xS2f_tF_bb2__Pred__src_0_wrt_0, #_AD__$s4main3foo1xS2f_tF_bb2__Pred__src_0_wrt_0.bb0!enumelt, %5 : $() 
  %16 = tuple $(predecessor: _AD__$s4main3foo1xS2f_tF_bb2__Pred__src_0_wrt_0, Float, Float) (%15, %1, %1) 
  %17 = enum $_AD__$s4main3foo1xS2f_tF_bb3__Pred__src_0_wrt_0, #_AD__$s4main3foo1xS2f_tF_bb3__Pred__src_0_wrt_0.bb2!enumelt, %16 : $(predecessor: _AD__$s4main3foo1xS2f_tF_bb2__Pred__src_0_wrt_0, Float, Float) 
  br bb3(%17 : $_AD__$s4main3foo1xS2f_tF_bb3__Pred__src_0_wrt_0) // id: %30

bb3(%18 : $_AD__$s4main3foo1xS2f_tF_bb3__Pred__src_0_wrt_0): // Preds: bb1 bb2
  // function_ref pullback of foo(x:)
  %19 = function_ref @$s4main3foo1xS2f_tFTJpSpSr : $@convention(thin) (Float, @owned _AD__$s4main3foo1xS2f_tF_bb3__Pred__src_0_wrt_0) -> Float 
  %20 = partial_apply [callee_guaranteed] %19(%18) : $@convention(thin) (Float, @owned _AD__$s4main3foo1xS2f_tF_bb3__Pred__src_0_wrt_0) -> Float 
  %21 = tuple (%0 : $Float, %20 : $@callee_guaranteed (Float) -> Float) 
  return %21 : $(Float, @callee_guaranteed (Float) -> Float)       
} // end sil function '$s4main3foo1xS2f_tFTJrSpSr'

// pullback of foo(x:)
sil private @$s4main3foo1xS2f_tFTJpSpSr : $@convention(thin) (Float, @owned _AD__$s4main3foo1xS2f_tF_bb3__Pred__src_0_wrt_0) -> Float {
bb0(%0 : $Float, %1 : $_AD__$s4main3foo1xS2f_tF_bb3__Pred__src_0_wrt_0):
  %2 = integer_literal $Builtin.Int64, 0          
  %3 = builtin "sitofp_Int64_FPIEEE32"(%2 : $Builtin.Int64) : $Builtin.FPIEEE32 
  debug_value %3 : $Builtin.FPIEEE32, let, name "x", argno 1, type $Float, expr op_fragment:#Float._value // id: %4
  switch_enum %1 : $_AD__$s4main3foo1xS2f_tF_bb3__Pred__src_0_wrt_0, case #_AD__$s4main3foo1xS2f_tF_bb3__Pred__src_0_wrt_0.bb2!enumelt: bb1, case #_AD__$s4main3foo1xS2f_tF_bb3__Pred__src_0_wrt_0.bb1!enumelt: bb2 // id: %5

// %6                                             
bb1(%6 : $(predecessor: _AD__$s4main3foo1xS2f_tF_bb2__Pred__src_0_wrt_0, Float, Float)): // Preds: bb0
  %7 = tuple_extract %6 : $(predecessor: _AD__$s4main3foo1xS2f_tF_bb2__Pred__src_0_wrt_0, Float, Float), 1 
  %8 = tuple_extract %6 : $(predecessor: _AD__$s4main3foo1xS2f_tF_bb2__Pred__src_0_wrt_0, Float, Float), 2 
  
  // function_ref closure #1 in static Float._vjpAdd(lhs:rhs:)
  %9 = function_ref @$sSf16_DifferentiationE7_vjpAdd3lhs3rhsSf5value_Sf_SftSfc8pullbacktSf_SftFZSf_SftSfcfU_ : $@convention(thin) (Float) -> (Float, Float) 
  %10 = thin_to_thick_function %9 : $@convention(thin) (Float) -> (Float, Float) to $@callee_guaranteed (Float) -> (Float, Float) 

  %10 = apply %9(%0) : $@callee_guaranteed (Float) -> (Float, Float) 
  strong_release %9 : $@callee_guaranteed (Float) -> (Float, Float) // id: %11
  %12 = tuple_extract %10 : $(Float, Float), 0    
  %13 = tuple_extract %10 : $(Float, Float), 1    

  // function_ref closure #1 in _vjpCos(_:)
  %14 = function_ref @$s16_Differentiation7_vjpCosySf5value_S2fc8pullbacktSfFS2fcfU_ : $@convention(thin) (Float, Float) -> Float 
  %15 = partial_apply [callee_guaranteed] %14(%8) : $@convention(thin) (Float, Float) -> Float 
  
  %16 = struct_extract %13 : $Float, #Float._value 
  %17 = builtin "fadd_FPIEEE32"(%16 : $Builtin.FPIEEE32, %3 : $Builtin.FPIEEE32) : $Builtin.FPIEEE32 
  %18 = struct $Float (%17 : $Builtin.FPIEEE32)   
  
  %19 = apply %15(%17) : $@callee_guaranteed (Float) -> Float 
  strong_release %15 : $@callee_guaranteed (Float) -> Float // id: %21

  // function_ref closure #1 in _vjpSin(_:)
  %22 = function_ref @$s16_Differentiation7_vjpSinySf5value_S2fc8pullbacktSfFS2fcfU_ : $@convention(thin) (Float, Float) -> Float 
  %23 = partial_apply [callee_guaranteed] %22(%7) : $@convention(thin) (Float, Float) -> Float 
  
  %24 = struct_extract %12 : $Float, #Float._value 
  %25 = builtin "fadd_FPIEEE32"(%24 : $Builtin.FPIEEE32, %3 : $Builtin.FPIEEE32) : $Builtin.FPIEEE32 
  %26 = struct $Float (%25 : $Builtin.FPIEEE32)   
  
  %24 = apply %23(%26) : $@callee_guaranteed (Float) -> Float 
  strong_release %23 : $@callee_guaranteed (Float) -> Float // id: %25
  
  %26 = struct_extract %24 : $Float, #Float._value 
  %27 = struct_extract %19 : $Float, #Float._value 
  %28 = builtin "fadd_FPIEEE32"(%27 : $Builtin.FPIEEE32, %3 : $Builtin.FPIEEE32) : $Builtin.FPIEEE32 
  %29 = builtin "fadd_FPIEEE32"(%26 : $Builtin.FPIEEE32, %28 : $Builtin.FPIEEE32) : $Builtin.FPIEEE32 
  %30 = struct $Float (%29 : $Builtin.FPIEEE32)   
  br bb3(%30 : $Float)                            

// %31                                            
bb2(%31 : $(predecessor: _AD__$s4main3foo1xS2f_tF_bb1__Pred__src_0_wrt_0, Float, Float, (Float, Float))): // Preds: bb0
  // Corresponding pullback block similar to bb1
  ...

// %56                                            
bb3(%56 : $Float):                                // Preds: bb1 bb2
  debug_value %56 : $Float, let, name "x", argno 1 // id: %57
  return %56 : $Float                             // id: %58
} // end sil function '$s4main3foo1xS2f_tFTJpSpSr'
```

As mentioned in the steps for the general closure specialization optimization, the specialized pullback can get rid of the closure usage altogether via further peephole optimizations that can fold the `partial_apply` of a function `f` and its corresponding `apply` into just the `apply` of `f`.

### Design

This optimization is going to be implemented as a SILFunctionTransform.

#### Pre-optimization steps
1. Exit early if function is in a module that does not import _Differentiable.
2. Exit early if the function is not an Autodiff VJP.
3. Exit early if the VJP has already been optimized.
4. Exit early if the returned pullback is not a partial_apply. For instance, it may just be a `function_ref` converted from thin to thick.
5. Exit early if **both** are true -
	1. The returned pullback is not partially applied over other intermediate pullbacks. 
  	2. The returned pullback is partially applied over an AD branch-trace enum, but the enum is trivial, i.e.,

		1. The enum has no cases. 
      		- Eg: enum for the entry block. However, this case should not be possible if the original function indeed contains control-flow.
      	2. The enum has cases but trivial payloads. 
        	- Eg: when the predecessor blocks for a block do not have any associated pullbacks. In such a case the payload might just be `()` or another branch-trace enum, which itself must be trivial. 

#### Optimization steps
At a high-level, there's 2 kinds of closures that we will be specializing against.
1. Top-level closures - these are directly closed over by the returned pullback.
2. Opaque closures - these are contained in branch-trace enums that the returned pullback closes over.

The information that we need to store for these closures should help us with 2 things.
1. Determining the signature of the specialized pullback.
2. Determining the case payloads of the specialized branch-trace enums.

We will be replacing the 2 kinds of closures mentioned above with the values that they close over, but we will need some additional information in order to correctly generate specialized branch-trace enums and pullbacks.
1. Top-level closures
	1. List of types of values this closure closes over.
  	2. Parameter index in the function's signature.
2. Opaque closures
  	1. List of types of values this closure closes over.
	2. Original branch-trace enum this closure lives in.
  	3. Case of the original branch-trace enum this closure lives in.
  	4. Index in the payload of the case of the original branch-trace enum this closure lives in.

> Note - There's a concrete example coming up shortly but, both kinds of closures mentioned above may close over other closures. To represent information for those cases we will actually have a list of lists of types of values to represent closed-over values. 

We will be storing this information in the below types.
```c++=
struct ClosureInfo {
  // List of lists to account for cases where 
  // top-level and opaque closures close over other
  // closures.
  vec<vec<Type>> arg_types;
  // A tag that represents whether the closure needs to 
  // keep its original type in the branch-trace
  // enum or specialized pullback.
  bool is_original_type;
  // A tag that represents whether the closure will have no
  // affect on the branch-trace enums and specialized pullback
  // signatures. 
  // 
  // Such a closure will still be moved to the specialized pullback 
  // from the VJP.
  bool no_affect_on_types;
};

// For top-level closures
//
// Index of closure is implicitly stored in the vec.
vec<ClosureInfo> TopLevelClosureInfo;

// For opaque closures
map<(branch-trace enum, case, index), ClosureInfo> OpaqueClosureInfo;
```

1. Populate TopLevelClosureInfo and OpaqueClosureInfo types.
	1. For all blocks of the VJP, look at the terminator instruction. We may have the following cases, possibly in combination.
		1. Terminator is `return`.
          	- **[CONTINUE]** The returned pullback directly closes over only an AD branch-trace enum.
          	
          	- The returned pullback directly closes over one or more intermediate pullback closures.

				1. **[TopLevelClosureInfo]** The intermediate pullback closures close over "non-closure" values.

				```c
				%0 = ... // Float
				// function_ref closure #1 in static Float._vjpMultiply(lhs:rhs:)
				%5 = function_ref @$sSf16_DifferentiationE12_vjpMultiply3lhs3rhsSf5value_Sf_SftSfc8pullbacktSf_SftFZSf_SftSfcfU_ : $@convention(thin) (Float, Float, Float) -> (Float, Float) // user: %6
				%6 = partial_apply [callee_guaranteed] %5(%0, %0) : $@convention(thin) (Float, Float, Float) -> (Float, Float) // user: %8

				// function_ref pullback of square(x:)
				%7 = function_ref @$s4test6square1xS2f_tFTJpSpSr : $@convention(thin) (Float, @owned @callee_guaranteed (Float) -> (Float, Float)) -> Float // user: %8
				%8 = partial_apply [callee_guaranteed] %7(%6) : $@convention(thin) (Float, @owned @callee_guaranteed (Float) -> (Float, Float)) -> Float // user: %9

				%9 = tuple (%4 : $Float, %8 : $@callee_guaranteed (Float) -> Float) // user: %10
				return %9 : $(Float, @callee_guaranteed (Float) -> Float) // id: %10
				``` 
				2. **[TopLevelClosureInfo]** The intermediate pullback closures close over one or more closures.
				```c
				%0 = ... // Float
				%4 = ... // Float
				// function_ref closure #1 in static Float._vjpMultiply(lhs:rhs:)
				%8 = function_ref @$sSf16_DifferentiationE12_vjpMultiply3lhs3rhsSf5value_Sf_SftSfc8pullbacktSf_SftFZSf_SftSfcfU_ : $@convention(thin) (Float, Float, Float) -> (Float, Float) // user: %9
				%9 = partial_apply [callee_guaranteed] %8(%0, %4) : $@convention(thin) (Float, Float, Float) -> (Float, Float) // user: %11

				// function_ref autodiff subset parameters thunk for pullback from @escaping @callee_guaranteed (@unowned Float) -> (@unowned Float, @unowned Float)
				%10 = function_ref @$sS3fIegydd_TJSpSSUpSrUSUP : $@convention(thin) (Float, @guaranteed @callee_guaranteed (Float) -> (Float, Float)) -> Float // user: %11
				%11 = partial_apply [callee_guaranteed] %10(%9) : $@convention(thin) (Float, @guaranteed @callee_guaranteed (Float) -> (Float, Float)) -> Float // user: %13

				// function_ref pullback of twoTimes(x:)
				%12 = function_ref @$s4test8twoTimes1xS2f_tFTJpSpSr : $@convention(thin) (Float, @owned @callee_guaranteed (Float) -> Float) -> Float // user: %13
				%13 = partial_apply [callee_guaranteed] %12(%11) : $@convention(thin) (Float, @owned @callee_guaranteed (Float) -> Float) -> Float // user: %15

				// function_ref pullback of square(x:)
				%14 = function_ref @$s4test6square1xS2f_tFTJpSpSr : $@convention(thin) (Float, @owned @callee_guaranteed (Float) -> Float) -> Float // user: %15
				%15 = partial_apply [callee_guaranteed] %14(%13) : $@convention(thin) (Float, @owned @callee_guaranteed (Float) -> Float) -> Float // user: %16

				%16 = tuple (%7 : $Float, %15 : $@callee_guaranteed (Float) -> Float) // user: %17
				return %16 : $(Float, @callee_guaranteed (Float) -> Float) // id: %17
				``` 

		2. Terminator is a `br` where the second argument is an AD branch-trace enum.
          	- **[OpaqueClosureInfo]** The branch-trace enum contains closures that only close over "non-closure" values.

			```c
			// function_ref closure #1 in _vjpSin(_:)
			%10 = function_ref @$s16_Differentiation7_vjpSinySf5value_S2fc8pullbacktSfFS2fcfU_ : $@convention(thin) (Float, Float) -> Float // user: %11
			%11 = partial_apply [callee_guaranteed] %10(%0) : $@convention(thin) (Float, Float) -> Float // user: %20

			// function_ref closure #1 in _vjpCos(_:)
			%14 = function_ref @$s16_Differentiation7_vjpCosySf5value_S2fc8pullbacktSfFS2fcfU_ : $@convention(thin) (Float, Float) -> Float // user: %15
			%15 = partial_apply [callee_guaranteed] %14(%0) : $@convention(thin) (Float, Float) -> Float // user: %20

			// function_ref closure #1 in static Float._vjpMultiply(lhs:rhs:)
			%18 = function_ref @$sSf16_DifferentiationE12_vjpMultiply3lhs3rhsSf5value_Sf_SftSfc8pullbacktSf_SftFZSf_SftSfcfU_ : $@convention(thin) (Float, Float, Float) -> (Float, Float) // user: %19
			%19 = partial_apply [callee_guaranteed] %18(%13, %9) : $@convention(thin) (Float, Float, Float) -> (Float, Float) // user: %20

			%20 = tuple $(predecessor: _AD__$s4main3foo1xS2f_tF_bb1__Pred__src_0_wrt_0, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> (Float, Float)) (%7, %11, %15, %19) // user: %21
			%21 = enum $_AD__$s4main3foo1xS2f_tF_bb3__Pred__src_0_wrt_0, #_AD__$s4main3foo1xS2f_tF_bb3__Pred__src_0_wrt_0.bb1!enumelt, %20 : $(predecessor: _AD__$s4main3foo1xS2f_tF_bb1__Pred__src_0_wrt_0, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> (Float, Float)) // user: %22
			br bb3(%0 : $Float, %21 : $_AD__$s4main3foo1xS2f_tF_bb3__Pred__src_0_wrt_0) // id: %22
			``` 
      
          	- **[OpaqueClosureInfo]** The branch-trace enum contains closures that close over one or more closures.
			```c
			// function_ref closure #1 in static Float._vjpMultiply(lhs:rhs:)
			%11 = function_ref @$sSf16_DifferentiationE12_vjpMultiply3lhs3rhsSf5value_Sf_SftSfc8pullbacktSf_SftFZSf_SftSfcfU_ : $@convention(thin) (Float, Float, Float) -> (Float, Float) // user: %12
			%12 = partial_apply [callee_guaranteed] %11(%0, %0) : $@convention(thin) (Float, Float, Float) -> (Float, Float) // user: %14

			// function_ref pullback of square(x:)
			%13 = function_ref @$s4test6square1xS2f_tFTJpSpSr : $@convention(thin) (Float, @owned @callee_guaranteed (Float) -> (Float, Float)) -> Float // user: %14
			%14 = partial_apply [callee_guaranteed] %13(%12) : $@convention(thin) (Float, @owned @callee_guaranteed (Float) -> (Float, Float)) -> Float // user: %15

			%15 = tuple $(predecessor: _AD__$s4test3foo1xS2f_tF_bb1__Pred__src_0_wrt_0, @callee_guaranteed (Float) -> Float) (%7, %14) // user: %16
			%16 = enum $_AD__$s4test3foo1xS2f_tF_bb3__Pred__src_0_wrt_0, #_AD__$s4test3foo1xS2f_tF_bb3__Pred__src_0_wrt_0.bb1!enumelt, %15 : $(predecessor: _AD__$s4test3foo1xS2f_tF_bb1__Pred__src_0_wrt_0, @callee_guaranteed (Float) -> Float) // user: %17
			br bb3(%10 : $Float, %16 : $_AD__$s4test3foo1xS2f_tF_bb3__Pred__src_0_wrt_0) // id: %17
			```
  
	2. Top-level or opaque closures, not closing over any values will need to be marked as having `no_affect_on_types`.
     An example of such a closure can be seen below.

	    ```c
	    // function_ref pullback of add(x:)
	    %6 = function_ref @$s4test3add1xS2f_tFTJpSpSr067$sSf16_DifferentiationE7_vjpAdd3lhs3rhsSf5value_Sf_SftSfc8pullbacktj1_k5FZSf_K6SfcfU_Tf3npf_n : $@convention(thin) (Float) -> Float // user: %7
	    %7 = thin_to_thick_function %6 : $@convention(thin) (Float) -> Float to $@callee_guaranteed (Float) -> Float // user: %13

	    // function_ref closure #1 in static Float._vjpMultiply(lhs:rhs:)
	    %10 = function_ref @$sSf16_DifferentiationE12_vjpMultiply3lhs3rhsSf5value_Sf_SftSfc8pullbacktSf_SftFZSf_SftSfcfU_ : $@convention(thin) (Float, Float, Float) -> (Float, Float) // user: %11
	    %11 = partial_apply [callee_guaranteed] %10(%5, %0) : $@convention(thin) (Float, Float, Float) -> (Float, Float) // user: %13

	    // function_ref pullback of foo(x:)
	    %12 = function_ref @$s4test3foo1xS2f_tFTJpSpSr : $@convention(thin) (Float, @owned @callee_guaranteed (Float) -> Float, @owned @callee_guaranteed (Float) -> (Float, Float)) -> Float // user: %13

	    // %7 will be marked as having `no_affect_on_types` since it does not close over any arguments
	    %13 = partial_apply [callee_guaranteed] %12(%7, %11) : $@convention(thin) (Float, @owned @callee_guaranteed (Float) -> Float, @owned @callee_guaranteed (Float) -> (Float, Float)) -> Float // user: %14

	    %14 = tuple (%9 : $Float, %13 : $@callee_guaranteed (Float) -> Float) // user: %15
	    return %14 : $(Float, @callee_guaranteed (Float) -> Float) // id: %15
	    ``` 

	2. Top-level or opaque closures, not having their corresponding `partial_apply` instructions visible in
     the VJP (by way of inlining), will be marked as `is_original_type`.

	3. "Trivial" enums, as specified in pre-opt step 5, with no cases or cases with trivial
       payloads are not persisted in the above map at all.
    
2. Generate specialized branch-trace enums
	1. Use [existing](https://github.com/apple/swift/blob/2b594b20dbf746c6f6b098d2523b78920a68a43b/lib/SILOptimizer/Differentiation/LinearMapInfo.cpp#L378) logic to generate specialized branch-trace enums from the original function corresponding to the VJP.
	2. Use the information gathered in `OpaqueClosureInfo` to set the types of the case payloads while [populating](https://github.com/apple/swift/blob/c1e892418fff4a91d9abf5e957b21f748fd28e7a/lib/SILOptimizer/Differentiation/LinearMapInfo.cpp#L125), the specialized branch-trace enums.

3. Generate specialized pullback and move code from VJP to it.

	There are 2 high-level code-motion cases corresponding to the 2 types of closures we are specializing against. Following information is needed in order to correctly move code from the VJP to the specialized pullback.

	1. Top-level closures
		1. **[A]** **[TopLevelClosureInfo]** `PartialApplyInst` in the VJP that needs to be copied over to the specialized pullback.
		2. **[B]** **[TopLevelClosureInfo]** Parameter index signifying the position of the closure in function input tuple.
		3. **[C]** Location in bb0 where the `PartialApplyInst` will be copied to in the specialized pullback.
	
	2. Opaque closures
		1. **[A]** **[OpaqueClosureInfo]** `PartialApplyInst` in the VJP that needs to be copied over to the specialized pullback.
		2. **[B]** Basic block in the specialized pullback where the `PartialApplyInst` will be moved to.
		3. **[C]** **[OpaqueClosureInfo]** Parameter index in this basic block's input tuple that needs to be modified.
		4. **[D]** Location in this basic block where the `PartialApplyInst` will be moved to.

	Code-motion steps.
	1. **[For each]** Top-level
		1. Modify the function and bb0 inputs by appending the corresponding closed over arguments. 
		2. Determine copy location in bb0 using [B].
			1. It should be right before the first use of the closure at [B].
		3. Copy PartialApplyInst from VJP at the copy location determined in (2). 
			1. The arguments should be the values appended to the function and bb inputs in (1).
		4. Modify the closure's original first use - an apply instruction, to use the result of the newly created partial apply.

		The specialized pullback after the application of these steps should go from -
		```c
		sil private @$s4test3foo1xS2f_tFTJpSpSr : $@convention(thin) (Float, @owned _AD__$s4test3foo1xS2f_tF_bb3__Pred__src_0_wrt_0, @owned @callee_guaranteed (Float) -> (Float, Float)) -> Float {
		bb0(%0 : $Float, %1 : $_AD__$s4test3foo1xS2f_tF_bb3__Pred__src_0_wrt_0, %2 : $@callee_guaranteed (Float) -> (Float, Float)):
		  	%3 = apply %2(%0) : $@callee_guaranteed (Float) -> (Float, Float) 
		  	...
		``` 

		To -
		```
		sil private @$s4test3foo1xS2f_tFTJpSpSr : $@convention(thin) (Float, @owned _AD__$s4test3foo1xS2f_tF_bb3__Pred__src_0_wrt_0, @owned @callee_guaranteed (Float) -> (Float, Float), (Float, Float)) -> Float {
		bb0(%0 : $Float, %1 : $_AD__$s4test3foo1xS2f_tF_bb3__Pred__src_0_wrt_0, %2 : $@callee_guaranteed (Float) -> (Float, Float), %3: (Float, Float)):
			%4 = tuple_extract %3: (Float, Float), 0
			%5 = tuple_extract %3: (Float, Float), 1
			// function_ref closure #1 in static Float._vjpMultiply(lhs:rhs:)
			%6 = function_ref @$sSf16_DifferentiationE12_vjpMultiply3lhs3rhsSf5value_Sf_SftSfc8pullbacktSf_SftFZSf_SftSfcfU_ : $@convention(thin) (Float, Float, Float) -> (Float, Float) // user: %46
			%7 = partial_apply [callee_guaranteed] %6(%4, %5) : $@convention(thin) (Float, Float, Float) -> (Float, Float) // user: %48
		  	%8 = apply %7(%0) : $@callee_guaranteed (Float) -> (Float, Float) 
		  	...
		```

	2. Opaque
		1. Generate a map `pai_to_tt`, from the `PartialApplyInst`s to the full `Type` of the tuple they are stored in.
			1. This map, in combination with `tt_to_bbid` (described below) will be used to derive [B].
			2. Below is an example that shows the state of `pai_to_tt` for some example VJP basic-blocks.
				```
				bb1:
				  ...
		 		  // function_ref closure #1 in static Float._vjpMultiply(lhs:rhs:)
				  %19 = function_ref @$sSf16_DifferentiationE12_vjpMultiply3lhs3rhsSf5value_Sf_SftSfc8pullbacktSf_SftFZSf_SftSfcfU_ : $@convention(thin) (Float, Float, Float) -> (Float, Float) // user: %20
				  %20 = partial_apply [callee_guaranteed] %19(%14, %10) : $@convention(thin) (Float, Float, Float) -> (Float, Float) // user: %22
				  %22 = tuple $(predecessor: _AD__$s4test3foo1xS2f_tF_bb1__Pred__src_0_wrt_0, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> (Float, Float)) (%8, %12, %16, %20) // user: %23
				  %23 = enum $_AD__$s4test3foo1xS2f_tF_bb3__Pred__src_0_wrt_0, #_AD__$s4test3foo1xS2f_tF_bb3__Pred__src_0_wrt_0.bb1!enumelt, %22 : $(predecessor: _AD__$s4test3foo1xS2f_tF_bb1__Pred__src_0_wrt_0, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> (Float, Float)) // user: %24
				  br bb3(%23 : $_AD__$s4test3foo1xS2f_tF_bb3__Pred__src_0_wrt_0, %18 : $Float) // id: %24

				bb2:
				  ...
				  // function_ref closure #1 in static Float._vjpAdd(lhs:rhs:)
				  %34 = function_ref @$sSf16_DifferentiationE7_vjpAdd3lhs3rhsSf5value_Sf_SftSfc8pullbacktSf_SftFZSf_SftSfcfU_ : $@convention(thin) (Float) -> (Float, Float) // user: %35
				  %35 = thin_to_thick_function %34 : $@convention(thin) (Float) -> (Float, Float) to $@callee_guaranteed (Float) -> (Float, Float) // user: %37
				  %37 = tuple $(predecessor: _AD__$s4test3foo1xS2f_tF_bb2__Pred__src_0_wrt_0, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> (Float, Float)) (%25, %28, %31, %35) // user: %38
				  %38 = enum $_AD__$s4test3foo1xS2f_tF_bb3__Pred__src_0_wrt_0, #_AD__$s4test3foo1xS2f_tF_bb3__Pred__src_0_wrt_0.bb2!enumelt, %37 : $(predecessor: _AD__$s4test3foo1xS2f_tF_bb2__Pred__src_0_wrt_0, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> (Float, Float)) // user: %39
				  br bb3(%38 : $_AD__$s4test3foo1xS2f_tF_bb3__Pred__src_0_wrt_0, %33 : $Float) // id: %39


				// PAI to tuple type map
				map<SILInstruction *, TupleType> pai_to_tt = {
					%20 => (_AD__$s4test3foo1xS2f_tF_bb1__Pred__src_0_wrt_0, (Float) -> Float, (Float) -> Float, (Float) -> (Float, Float)),
					%35 => (_AD__$s4test3foo1xS2f_tF_bb2__Pred__src_0_wrt_0, (Float) -> Float, (Float) -> Float, (Float) -> (Float, Float))

				};

				```
	
		2. Generate a map `tt_to_bbid`, from the input tuple-type of a basic block in the pullback to the basic block id.
			1. This map, in combination with `pai_to_tt` (described above) will be used to derive [B]. 
			2. This map will only include basic blocks whose inputs look as below -
				```
				(
					// First argument is the predecessor enum
					predecessor: _AD__$s4test3foo1xS2f_tF_bb2__Pred__src_0_wrt_0, 
					// Then there should be one or more closures
					@callee_guaranteed (Float) -> Float, 
					...
				)
				``` 
			3. bb0 will be excluded as it will be handled by the steps for top-level closures.
			4. Below is an example that shows the state of `tt_to_bbid` for some example pullback basic-blocks.
				```
				bb0(%0 : $Float, %1 : $_AD__$s4test3foo1xS2f_tF_bb3__Pred__src_0_wrt_0, %2 : $@callee_guaranteed (Float) -> (Float, Float)):
					...
				bb1(%19 : $(predecessor: _AD__$s4test3foo1xS2f_tF_bb2__Pred__src_0_wrt_0, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> (Float, Float))): // Preds: bb0
					...
				bb2(%47 : $(predecessor: _AD__$s4test3foo1xS2f_tF_bb1__Pred__src_0_wrt_0, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> (Float, Float))): // Preds: bb0
					...
				bb3(%75 : $Builtin.FPIEEE32):                     
					...

				// Tuple type to basic-block id map
				map<TupleType, size_t> tt_to_bbid = {
					(_AD__$s4test3foo1xS2f_tF_bb2__Pred__src_0_wrt_0, (Float) -> Float, (Float) -> Float, (Float) -> (Float, Float)) => bb1,

					(_AD__$s4test3foo1xS2f_tF_bb1__Pred__src_0_wrt_0, (Float) -> Float, (Float) -> Float, (Float) -> (Float, Float)) => bb2
				};
				```

		
		3. Generate a map `orig_to_spec_enum`, from the old branch-trace enum to the new branch-trace enum.
			1. Signature -
				```
				map<EnumDecl *, EnumDecl *> orig_to_spec_enum;
				```
			2. Given an original branch-trace enum + case, this map can help us determine the payload type of the corresponding specialized branch-trace enum + case.

		4. Update the specialized pullback to use the specialized branch-trace enums and case payloads. There will be 4 cases here.
			1. Function input
				```
				// From original branch-trace enum
				sil private @$s4test3foo1xS2f_tFTJpSpSr : $@convention(thin) (Float, @owned _AD__$s4test3foo1xS2f_tF_bb3__Pred__src_0_wrt_0, (Float, Float)) -> Float

				// To specialized branch-trace enum
				// Closures in the function input should already have been replaced while handling top-level closures. Same with bb0's input tuple.
				sil private @$s4test3foo1xS2f_tFTJpSpSr : $@convention(thin) (Float, @owned _AD__$s4test3foo1xS2f_tF_bb3__Pred__src_0_wrt_0_spec, (Float, Float)) -> Float
				```
			2. Basic-block inputs
				```
				// From original branch-trace enum
				bb1(%19 : $(predecessor: _AD__$s4test3foo1xS2f_tF_bb2__Pred__src_0_wrt_0, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> (Float, Float))): // Preds: bb0

				// To specialized branch-trace enum
				// Closures are replaced with their respective closed over values
				bb1(%19 : $(predecessor: _AD__$s4test3foo1xS2f_tF_bb2__Pred__src_0_wrt_0_spec, Float, Float, (Float, Float)): // Preds: bb0
				```
			3. `switch_enum` basic-block terminators
				```
				// From original branch-trace enum
				switch_enum %1 : $_AD__$s4test3foo1xS2f_tF_bb3__Pred__src_0_wrt_0, case #_AD__$s4test3foo1xS2f_tF_bb3__Pred__src_0_wrt_0.bb2!enumelt: bb1, case #_AD__$s4test3foo1xS2f_tF_bb3__Pred__src_0_wrt_0.bb1!enumelt: bb2 // id: %18

				// To specialized branch-trace enum
				switch_enum %1 : $_AD__$s4test3foo1xS2f_tF_bb3__Pred__src_0_wrt_0_spec, case #_AD__$s4test3foo1xS2f_tF_bb3__Pred__src_0_wrt_0_spec.bb2!enumelt: bb1, case #_AD__$s4test3foo1xS2f_tF_bb3__Pred__src_0_wrt_0_spec.bb1!enumelt: bb2 // id: %18
				```
			4. `br` basic-block terminators
				```
				// From original branch-trace enum
				%69 = unchecked_enum_data %40 : $_AD__$s4main3foo1xS2f_tF_bb5__Pred__src_0_wrt_0, #_AD__$s4main3foo1xS2f_tF_bb5__Pred__src_0_wrt_0.bb3!enumelt // user: %70
				br bb8(%49 : $Builtin.FPIEEE32, %67 : $Float, %69 : $(predecessor: _AD__$s4main3foo1xS2f_tF_bb3__Pred__src_0_wrt_0, @callee_guaranteed (Float) -> Float)) // id: %70

				// To specialized branch-trace enum
				// Closures are replaced with their respective closed over values
				%69 = unchecked_enum_data %40 : $_AD__$s4main3foo1xS2f_tF_bb5__Pred__src_0_wrt_0_spec, #_AD__$s4main3foo1xS2f_tF_bb5__Pred__src_0_wrt_0_spec.bb3!enumelt // user: %70
				br bb8(%49 : $Builtin.FPIEEE32, %67 : $Float, %69 : $(predecessor: _AD__$s4main3foo1xS2f_tF_bb3__Pred__src_0_wrt_0_spec, Float) // id: %70
				```
		
		5. For each opaque closure
			> Info for opaque closures.
			>	1. **[A]** **[OpaqueClosureInfo]** `PartialApplyInst` in the VJP that needs to be copied over to the specialized pullback.
			>	2. **[B]** Basic block in the specialized pullback where the `PartialApplyInst` will be moved to.
			>	3. **[C]** **[OpaqueClosureInfo]** Parameter index in this basic block's input tuple that needs to be modified.
			>	4. **[D]** Location in this basic block where the `PartialApplyInst` will be moved to.
			1. Determine [B] using [A], `pai_to_tt` and `tt_to_bbid`.
			2. First N instructions in [B], where N is the number of input parameters, destructure the input tuple, from 0th or 1st to the Nth element.
				```
				bb2(%47 : $(predecessor: _AD__$s4test3foo1xS2f_tF_bb1__Pred__src_0_wrt_0, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> (Float, Float))): // Preds: bb0
				  %48 = tuple_extract %47 : $(predecessor: _AD__$s4test3foo1xS2f_tF_bb1__Pred__src_0_wrt_0, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> (Float, Float)), 1 // users: %70, %69
				  %49 = tuple_extract %47 : $(predecessor: _AD__$s4test3foo1xS2f_tF_bb1__Pred__src_0_wrt_0, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> (Float, Float)), 2 // users: %65, %64
				  %50 = tuple_extract %47 : $(predecessor: _AD__$s4test3foo1xS2f_tF_bb1__Pred__src_0_wrt_0, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> Float, @callee_guaranteed (Float) -> (Float, Float)), 3 // users: %55, %54
			 	```
		 	3. These `tuple_extract` instructions will now be extracting closed-over arguments instead of closures. To fix the gap between the actual type of these arguments and their usage as closures, we will follow steps similar to the handling of top-level closures.
		 		1. Determine copy location in [B].
					1. It should be right before the first use of the closure in [B].
				2. Copy PartialApplyInst from VJP at the copy location determined in (1). 
					1. The arguments should be coming from the corresponding `tuple_extract` at the start of the pullback.
				3. Modify the closure's original first use - an apply instruction, to use the result of the newly created partial apply.
		6. Below is an example showing the state of a pullback basic-block after the application of the above steps.
			1. From this -
				```
				bb1(%6 : $(predecessor: _AD__$s4test3foo1xS2f_tF_bb2__Pred__src_0_wrt_0, @callee_guaranteed (Float) -> Float)): // Preds: bb0
				  %7 = tuple_extract %6 : $(predecessor: _AD__$s4test3foo1xS2f_tF_bb2__Pred__src_0_wrt_0, @callee_guaranteed (Float) -> Float), 1 // user: %8
				  %8 = apply %7(%0) : $@callee_guaranteed (Float) -> Float // user: %9
				  %9 = struct_extract %8 : $Float, #Float._value  // user: %10
				  %10 = builtin "fadd_FPIEEE32"(%9 : $Builtin.FPIEEE32, %3 : $Builtin.FPIEEE32) : $Builtin.FPIEEE32 // user: %11
				  %11 = struct $Float (%10 : $Builtin.FPIEEE32)   // users: %13, %12
				  br bb3(%11 : $Float)                            // id: %13
				```
			2. To this -
				```
				bb1(%6 : $(predecessor: _AD__$s4test3foo1xS2f_tF_bb2__Pred__src_0_wrt_0_spec, Float): // Preds: bb0
				  %7 = tuple_extract %6 : $(predecessor: _AD__$s4test3foo1xS2f_tF_bb2__Pred__src_0_wrt_0_spec, Float, 1 // user: %8
				  // function_ref closure #1 in _vjpSin(_:)
				  %8 = function_ref @$s16_Differentiation7_vjpSinySf5value_S2fc8pullbacktSfFS2fcfU_ : $@convention(thin) (Float, Float) -> Float // user: %19
				  %9 = partial_apply [callee_guaranteed] %8(%7) : $@convention(thin) (Float, Float) -> Float // user: %20
				  %10 = apply %9(%0) : $@callee_guaranteed (Float) -> Float // user: %9
				  %11 = struct_extract %10 : $Float, #Float._value  // user: %10
				  %12 = builtin "fadd_FPIEEE32"(%11 : $Builtin.FPIEEE32, %3 : $Builtin.FPIEEE32) : $Builtin.FPIEEE32 // user: %11
				  %13 = struct $Float (%12 : $Builtin.FPIEEE32)   // users: %13, %12
				  br bb3(%11 : $Float)                            // id: %13
				```
	3. Some special cases in closure-code motion, that we need to keep in mind, but should be able to handle with slight tweaks in the above described algorithm.
		1. `thin_to_thick_function` closures.
			1. These closures won't have corresponding closed-over arguments in both the top-level and the opaque cases.
		2. Closures that close over other closures.
			1. During code motion, the entire chain of closures will need to be moved from the VJP to the pullback.
			2. The corresponding closed-over arguments that will be created for such closures will be the arguments that the first closure in the chain closes over.

#### Post-pptimization steps
1. Peephole optimizations to get rid of any partial applies moved to the cloned callee.
2. Peephole optimizations to get rid of any dead instructions in the caller or the cloned callee.

#### Location in optimization pipeline
Key points for this optimization to work.
1. Maximal inlining into VJPs should have happened.
2. VJPs themselves should not have been inlined. 
	1. Once a VJP has been inlined there is no point specializing the pullback it returns.

Considering the above points, this pass should run after inlining into VJPs but before most of the other performance inlining. The exact location of the pass can be discussed once implementation is complete and perhaps using trial and error.
