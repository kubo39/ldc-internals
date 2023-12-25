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

#### 再現コード

```d
auto arrayCopy()
{
    static void copy(int[] a, int[] b)
    {
        a[] = b[];
    }
    int[2] a = [1, 2];
    int[2] b = [3, 4];
    copy(a, b);
    return a;
}
```

##### `ldc2 --output-ll arraycopy.d` の場合

`_d_array_slice_copy`を利用していることがわかる。

```ll
(...)
define void @_D9arraycopy9arrayCopyFZ4copyFNaNbNiNfAiQcZv({ i64, i32* } %a_arg, { i64, i32* } %b_arg) #0 {
  %a = alloca { i64, i32* }, align 8              ; [#uses = 3, size/byte = 16]
  %b = alloca { i64, i32* }, align 8              ; [#uses = 3, size/byte = 16]
  store { i64, i32* } %a_arg, { i64, i32* }* %a, align 8
  store { i64, i32* } %b_arg, { i64, i32* }* %b, align 8
  %1 = getelementptr inbounds { i64, i32* }, { i64, i32* }* %a, i32 0, i32 1 ; [#uses = 1, type = i32**]
  %.ptr = load i32*, i32** %1, align 8            ; [#uses = 2]
  %2 = getelementptr inbounds { i64, i32* }, { i64, i32* }* %a, i32 0, i32 0 ; [#uses = 1, type = i64*]
  %.len = load i64, i64* %2, align 8              ; [#uses = 2]
  %3 = insertvalue { i64, i32* } undef, i64 %.len, 0 ; [#uses = 1]
  %4 = insertvalue { i64, i32* } %3, i32* %.ptr, 1 ; [#uses = 0]
  %5 = getelementptr inbounds { i64, i32* }, { i64, i32* }* %b, i32 0, i32 1 ; [#uses = 1, type = i32**]
  %.ptr1 = load i32*, i32** %5, align 8           ; [#uses = 2]
  %6 = getelementptr inbounds { i64, i32* }, { i64, i32* }* %b, i32 0, i32 0 ; [#uses = 1, type = i64*]
  %.len2 = load i64, i64* %6, align 8             ; [#uses = 2]
  %7 = insertvalue { i64, i32* } undef, i64 %.len2, 0 ; [#uses = 1]
  %8 = insertvalue { i64, i32* } %7, i32* %.ptr1, 1 ; [#uses = 0]
  %9 = bitcast i32* %.ptr to i8*                  ; [#uses = 1]
  %10 = bitcast i32* %.ptr1 to i8*                ; [#uses = 1]
  call void @_d_array_slice_copy(i8* nocapture %9, i64 %.len, i8* nocapture %10, i64 %.len2, i64 4) #1
  ret void
```

##### `ldc2  -O2 --release --checkaction=halt --boundscheck=off --output-ll arraycopy.d` の場合

copy関数が`llvm.memcpy`を利用するように変化していることがわかる。

```ll
(...)
define void @_D9arraycopy9arrayCopyFZ4copyFNaNbNiNfAiQcZv({ i64, i32* } %a_arg, { i64, i32* } %b_arg) local_unnamed_addr #1 {
  %a_arg.fca.0.extract = extractvalue { i64, i32* } %a_arg, 0 ; [#uses = 1]
  %a_arg.fca.1.extract = extractvalue { i64, i32* } %a_arg, 1 ; [#uses = 1]
  %b_arg.fca.1.extract = extractvalue { i64, i32* } %b_arg, 1 ; [#uses = 1]
  %1 = bitcast i32* %a_arg.fca.1.extract to i8*   ; [#uses = 1]
  %2 = bitcast i32* %b_arg.fca.1.extract to i8*   ; [#uses = 1]
  %3 = shl i64 %a_arg.fca.0.extract, 2            ; [#uses = 1]
  tail call void @llvm.memcpy.p0i8.p0i8.i64(i8* align 1 %1, i8* align 1 %2, i64 %3, i1 false)
  ret void
(...)
```

- おまけ: --disable-simplify-drtcallsをさらに追加した場合

```console
$ ldc2 -O2 --release --checkaction=halt --boundscheck=off --disable-simplify-drtcalls --output-ll arraycopy.d
```

llvm.memcpyにおちてるので、simplifyDRuntimeCallsなくてもLLVMがいい感じにやってくれている可能性がある。

```ll
(...)
define void @_D9arraycopy9arrayCopyFZ4copyFNaNbNiNfAiQcZv({ i64, i32* } %a_arg, { i64, i32* } %b_arg) local_unnamed_addr #1 {
  %a_arg.fca.0.extract = extractvalue { i64, i32* } %a_arg, 0 ; [#uses = 1]
  %a_arg.fca.1.extract = extractvalue { i64, i32* } %a_arg, 1 ; [#uses = 1]
  %b_arg.fca.1.extract = extractvalue { i64, i32* } %b_arg, 1 ; [#uses = 1]
  %1 = bitcast i32* %a_arg.fca.1.extract to i8*   ; [#uses = 1]
  %2 = bitcast i32* %b_arg.fca.1.extract to i8*   ; [#uses = 1]
  %3 = shl i64 %a_arg.fca.0.extract, 2            ; [#uses = 1]
  tail call void @llvm.memcpy.p0i8.p0i8.i64(i8* align 1 %1, i8* align 1 %2, i64 %3, i1 false)
  ret void
```
