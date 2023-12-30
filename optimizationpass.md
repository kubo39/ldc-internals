# 最適化パス

## LLVM pass manager

TODO:

## LDC固有

LLVMの最適化パスにhookしてD言語固有の最適化パスを設定している。

### SimplifyDRuntimeCalls

この最適化パスは特定のランタイム関数をよりシンプルな形式に落とし込む。

このパスはLLVMの最適化パスにフックしている。
デフォルトでこれらは同時に最適化が行われるが、個別にdisableすることも可能である。

```cpp
  if (!disableLangSpecificPasses) {
    if (!disableSimplifyDruntimeCalls) {
      // FIXME: Is this registerOptimizerLastEPCallback correct here
      //(had registerLoopOptimizerEndEPCallback) but that seems wrong
      pb.registerOptimizerLastEPCallback(addSimplifyDRuntimeCallsPass);
    }
```

現在であれば以下の関数が最適化対象となっている。

```cpp
/// Optimizations - Populate the Optimizations map with all the optimizations
/// we know.
void SimplifyDRuntimeCalls::InitOptimizations() {
  // Some array-related optimizations
  Optimizations["_d_arraysetlengthT"] = &ArraySetLength;
  Optimizations["_d_arraysetlengthiT"] = &ArraySetLength;
  Optimizations["_d_array_slice_copy"] = &ArraySliceCopy;

  /* Delete calls to runtime functions which aren't needed if their result is
   * unused. That comes down to functions that don't do anything but
   * GC-allocate and initialize some memory.
   * We don't need to do this for functions which are marked 'readnone' or
   * 'readonly', since LLVM doesn't need our help figuring out when those can
   * be deleted.
   * (We can't mark allocating calls as readonly/readnone because they don't
   * return the same pointer every time when called with the same arguments)
   */
  Optimizations["_d_allocmemoryT"] = &Allocation;
  Optimizations["_d_newarrayT"] = &Allocation;
  Optimizations["_d_newarrayiT"] = &Allocation;
  Optimizations["_d_newarrayU"] = &Allocation;
  Optimizations["_d_newarraymT"] = &Allocation;
  Optimizations["_d_newarraymiT"] = &Allocation;
  Optimizations["_d_newarraymvT"] = &Allocation;
  Optimizations["_d_newclass"] = &Allocation;
  Optimizations["_d_allocclass"] = &Allocation;
}
```

例としてまずArraySliceCopyの最適化をみていく。

この関数はコメントにあるとおりにランタイム関数ArraySliceCopyをllvm.memcpyに変換する。

```cpp
/// ArraySliceCopyOpt - Turn slice copies into llvm.memcpy when safe
Value *ArraySliceCopyOpt::CallOptimizer(Function *Callee, CallInst *CI,
                     IRBuilder<> &B) {
```

ArraySliceCopyはLDCで独自にもっている実装になっている。
DMDのものとの違いは引数の数と明示的にllvm.memcpy intrinsicを利用している点である。

```d
version (LDC)
{
    void _d_array_slice_copy(void* dst, size_t dstlen, void* src, size_t srclen, size_t elemsz)
    {
        import ldc.intrinsics : llvm_memcpy;

        enforceRawArraysConformable("copy", elemsz, src[0..srclen], dst[0..dstlen]);
        llvm_memcpy!size_t(dst, src, dstlen * elemsz, 0);
    }
}
```

上記の関数をみると、実行時に都度enforceRawArraysConfrmableを呼び出している点が気になる。
この処理は長さが同じであるかとメモリのオーバーラップが存在するか(つまり2つのスライスがエイリアスになっているか)を動的に検査している。

これらをコンパイル時に検査できれば単なるllvm.memcpyに置き換えられる、というのがこの最適化パスの目的である。なので行っていることはそんなに複雑ではない。

以下のコードは長さチェックを行っている処理である。
単純に一致しないケースでは最適化を諦めてnullptrを返し、オリジナルの動的検査を行う関数をそのまま利用する。(実行時にエラーになる)

```cpp
  // Check the lengths match
  if (CI->getOperand(3) != DstLength) {
    return nullptr;
  }

  const auto ElemSz = llvm::cast<ConstantInt>(CI->getOperand(4));

  // Assume unknown size unless we have constant length
  std::uint64_t Sz = llvm::MemoryLocation::UnknownSize;
  if (ConstantInt *Int = dyn_cast<ConstantInt>(DstLength)) {
    Sz = (Int->getValue() * ElemSz->getValue()).getZExtValue();
  }
```

次にLLVMのエイリアス解析の力を借りて二つのスライスがエイリアスになっていないかをチェックしている。

```cpp
  // Check if the pointers may alias
  if (AA->alias(CI->getOperand(0), Sz, CI->getOperand(2), Sz)) {
    return nullptr;
  }
```

長さが同じでポインタがエイリアスになっていないことが確実であれば、安全にllvm.memcpyに置き換えられるとしている。

```cpp
  // Equal length and the pointers definitely don't alias, so it's safe to
  // replace the call with memcpy
  auto size = Sz != llvm::MemoryLocation::UnknownSize
                  ? DtoConstSize_t(Callee->getContext(),
                                   Callee->getParent()->getDataLayout(), Sz)
                  : B.CreateMul(DstLength, ElemSz);
  return EmitMemCpy(CI->getOperand(0), CI->getOperand(2), size, 1, B);
```

#### 実験コード

```d
pragma(mangle, "arrayCopy")
auto arrayCopy()
{
    int[2] a = [1, 2];
    int[] x = a[0..2];
    int[2] b = [3, 4];
    int[] y = b[0..2];
    x[] = y[];
    return;
}
```

##### `ldc2 --output-ll arraycopy.d` の場合

最適化を行っていないのでやや長いが、`_d_array_slice_copy`を利用していることがわかる。

```ll
(...)
define void @arrayCopy() #0 {
  %a = alloca [2 x i32], align 4                  ; [#uses = 2, size/byte = 8]
  %x = alloca { i64, i32* }, align 8              ; [#uses = 4, size/byte = 16]
  %b = alloca [2 x i32], align 4                  ; [#uses = 2, size/byte = 8]
  %y = alloca { i64, i32* }, align 8              ; [#uses = 4, size/byte = 16]
  store [2 x i32] [i32 1, i32 2], [2 x i32]* %a, align 4
  %1 = bitcast [2 x i32]* %a to i32*              ; [#uses = 1]
  %lowerbound = getelementptr inbounds i32, i32* %1, i64 0 ; [#uses = 2, type = i32*]
  %2 = insertvalue { i64, i32* } { i64 2, i32* undef }, i32* %lowerbound, 1 ; [#uses = 0]
  %3 = getelementptr inbounds { i64, i32* }, { i64, i32* }* %x, i32 0, i32 0 ; [#uses = 1, type = i64*]
  store i64 2, i64* %3, align 8
  %4 = getelementptr inbounds { i64, i32* }, { i64, i32* }* %x, i32 0, i32 1 ; [#uses = 1, type = i32**]
  store i32* %lowerbound, i32** %4, align 8
  store [2 x i32] [i32 3, i32 4], [2 x i32]* %b, align 4
  %5 = bitcast [2 x i32]* %b to i32*              ; [#uses = 1]
  %lowerbound1 = getelementptr inbounds i32, i32* %5, i64 0 ; [#uses = 2, type = i32*]
  %6 = insertvalue { i64, i32* } { i64 2, i32* undef }, i32* %lowerbound1, 1 ; [#uses = 0]
  %7 = getelementptr inbounds { i64, i32* }, { i64, i32* }* %y, i32 0, i32 0 ; [#uses = 1, type = i64*]
  store i64 2, i64* %7, align 8
  %8 = getelementptr inbounds { i64, i32* }, { i64, i32* }* %y, i32 0, i32 1 ; [#uses = 1, type = i32**]
  store i32* %lowerbound1, i32** %8, align 8
  %9 = getelementptr inbounds { i64, i32* }, { i64, i32* }* %x, i32 0, i32 1 ; [#uses = 1, type = i32**]
  %.ptr = load i32*, i32** %9, align 8            ; [#uses = 2]
  %10 = getelementptr inbounds { i64, i32* }, { i64, i32* }* %x, i32 0, i32 0 ; [#uses = 1, type = i64*]
  %.len = load i64, i64* %10, align 8             ; [#uses = 2]
  %11 = insertvalue { i64, i32* } undef, i64 %.len, 0 ; [#uses = 1]
  %12 = insertvalue { i64, i32* } %11, i32* %.ptr, 1 ; [#uses = 0]
  %13 = getelementptr inbounds { i64, i32* }, { i64, i32* }* %y, i32 0, i32 1 ; [#uses = 1, type = i32**]
  %.ptr2 = load i32*, i32** %13, align 8          ; [#uses = 2]
  %14 = getelementptr inbounds { i64, i32* }, { i64, i32* }* %y, i32 0, i32 0 ; [#uses = 1, type = i64*]
  %.len3 = load i64, i64* %14, align 8            ; [#uses = 2]
  %15 = insertvalue { i64, i32* } undef, i64 %.len3, 0 ; [#uses = 1]
  %16 = insertvalue { i64, i32* } %15, i32* %.ptr2, 1 ; [#uses = 0]
  %17 = bitcast i32* %.ptr to i8*                 ; [#uses = 1]
  %18 = bitcast i32* %.ptr2 to i8*                ; [#uses = 1]
  call void @_d_array_slice_copy(i8* nocapture %17, i64 %.len, i8* nocapture %18, i64 %.len3, i64 4) #1
  ret void
}
```

##### `ldc2 -O2 --output-ll arraycopy.d` の場合

O2の場合この最適化パスを通ってllvm.memcpyにloweringされる。

```ll
(...)
define void @arrayCopy() local_unnamed_addr #0 {
  %a = alloca [2 x i32], align 4                  ; [#uses = 3, size/byte = 8]
  %b = alloca [2 x i32], align 4                  ; [#uses = 3, size/byte = 8]
  %.fca.0.gep4 = getelementptr inbounds [2 x i32], [2 x i32]* %a, i64 0, i64 0 ; [#uses = 1, type = i32*]
  store i32 1, i32* %.fca.0.gep4, align 4
  %.fca.1.gep5 = getelementptr inbounds [2 x i32], [2 x i32]* %a, i64 0, i64 1 ; [#uses = 1, type = i32*]
  store i32 2, i32* %.fca.1.gep5, align 4
  %.fca.0.gep = getelementptr inbounds [2 x i32], [2 x i32]* %b, i64 0, i64 0 ; [#uses = 1, type = i32*]
  store i32 3, i32* %.fca.0.gep, align 4
  %.fca.1.gep = getelementptr inbounds [2 x i32], [2 x i32]* %b, i64 0, i64 1 ; [#uses = 1, type = i32*]
  store i32 4, i32* %.fca.1.gep, align 4
  %1 = bitcast [2 x i32]* %a to i8*               ; [#uses = 1]
  %2 = bitcast [2 x i32]* %b to i8*               ; [#uses = 1]
  call void @llvm.memcpy.p0i8.p0i8.i64(i8* align 1 %1, i8* align 1 %2, i64 8, i1 false)
  ret void
}
```

##### `--disable-simplify-drtcalls`を付与した場合

これだけでは実際にsimplify-drtcallsが効いたのか判別できないので、この最適化だけ無効にした状態で再度LLVM IRをみてみる。

```ll
(...)
define void @arrayCopy() local_unnamed_addr #0 {
  %a = alloca [2 x i32], align 4                  ; [#uses = 3, size/byte = 8]
  %b = alloca [2 x i32], align 4                  ; [#uses = 3, size/byte = 8]
  %.fca.0.gep4 = getelementptr inbounds [2 x i32], [2 x i32]* %a, i64 0, i64 0 ; [#uses = 1, type = i32*]
  store i32 1, i32* %.fca.0.gep4, align 4
  %.fca.1.gep5 = getelementptr inbounds [2 x i32], [2 x i32]* %a, i64 0, i64 1 ; [#uses = 1, type = i32*]
  store i32 2, i32* %.fca.1.gep5, align 4
  %.fca.0.gep = getelementptr inbounds [2 x i32], [2 x i32]* %b, i64 0, i64 0 ; [#uses = 1, type = i32*]
  store i32 3, i32* %.fca.0.gep, align 4
  %.fca.1.gep = getelementptr inbounds [2 x i32], [2 x i32]* %b, i64 0, i64 1 ; [#uses = 1, type = i32*]
  store i32 4, i32* %.fca.1.gep, align 4
  %1 = bitcast [2 x i32]* %a to i8*               ; [#uses = 1]
  %2 = bitcast [2 x i32]* %b to i8*               ; [#uses = 1]
  call void @_d_array_slice_copy(i8* nocapture nonnull %1, i64 2, i8* nocapture nonnull %2, i64 2, i64 4) #1
  ret void
}
```

`_d_array_slice_copy`が呼ばれていることが確認できたので、simplify-drtcallsが効いていることがわかった。

#### 実験コード(だめな例)

意図的にだめな例を作ってみよう。

以下のコードはxとyがオーバーラップするように調整している。そのため、このコードが実行されると`_enforceNoOverlap`でひっかかってエラーになるはずである。

```d
pragma(mangle, "arrayCopy")
auto arrayCopy()
{
    int[4] a = [1, 2, 3, 4];
    int[] x = a[0..2];
    int[] y = a[1..3];
    x[] = y[];
    return;
}
```

このコードに対してもし最適化されてしまうと挙動が変わってしまうことになるため、最適化オプションを有効にしてもllvm.memcpyへのloweringは行われない。

```ll
(...)
define void @arrayCopy() local_unnamed_addr #0 {
  %a = alloca <4 x i32>, align 16                 ; [#uses = 3, size/byte = 16]
  %.fca.1.gep = getelementptr inbounds <4 x i32>, <4 x i32>* %a, i64 0, i64 1 ; [#uses = 1, type = i32*]
  store <4 x i32> <i32 1, i32 2, i32 3, i32 4>, <4 x i32>* %a, align 16
  %1 = bitcast <4 x i32>* %a to i8*               ; [#uses = 1]
  %2 = bitcast i32* %.fca.1.gep to i8*            ; [#uses = 1]
  call void @_d_array_slice_copy(i8* nocapture nonnull %1, i64 2, i8* nocapture nonnull %2, i64 2, i64 4) #1
  ret void
}
```
