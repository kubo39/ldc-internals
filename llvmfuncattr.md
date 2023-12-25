# llvm function attribute

LLVMが関数の最適化を行うとき、関数属性は重要である。

## gen/functions.cpp

関数属性の付与は主にこのファイル内で行われている。

例えば以下の処理をみてみる。

NoAlias属性はC言語のrestrict相当で、これをつけても安全か考えてみる。

```cpp
    if (abi->returnInArg(f, fd && fd->needThis())) {
      // sret return
#if LDC_LLVM_VER >= 1400
      llvm::AttrBuilder sretAttrs(getGlobalContext());
#else
      llvm::AttrBuilder sretAttrs;
#endif
#if LDC_LLVM_VER >= 1200
      sretAttrs.addStructRetAttr(DtoType(rt));
#else
      sretAttrs.addAttribute(LLAttribute::StructRet);
#endif
      sretAttrs.addAttribute(LLAttribute::NoAlias);
      if (unsigned alignment = DtoAlignment(rt))
        sretAttrs.addAlignmentAttr(alignment);
      newIrFty.arg_sret = new IrFuncTyArg(rt, true, std::move(sretAttrs));
      rt = Type::tvoid;
```

returnInArgの実装はアーキテクチャによって異なる。
x86_64の場合は以下。
返り値が参照でないこと・x87 crealでない・passInMemoryがtrueになること。

```cpp
bool X86_64TargetABI::returnInArg(TypeFunction *tf, bool) {
  if (tf->isref()) {
    return false;
  }

  Type *rt = tf->next->toBasetype();

  // x87 creal is returned on the x87 stack
  if (returnsComplexReal(tf))
    return false;

  return passInMemory(rt);
}
```

passInMemoryの実装もアーキテクチャごとに異なる。
ここでは関数の引数が空になっていることを要求している。

```cpp
  static bool passInMemory(Type* t) {
    TypeTuple *argTypes = getArgTypes(t);
    return argTypes && argTypes->arguments->empty();
  }
```

もとの関数では(arg_sretはいったんおいておいて)、返り値をTvoidに書き換えてる。
以上からこれはRVOを行っているということがなんとなくわかる。
この場合はNoAlias属性を付与してもおそらく問題はないだろう。

NoAlias属性がRVOで付与されているかは以下のようなコードで確認ができる。

```d
struct S
{
    int[8] arr;
}

/**
// ldc2 -O --output-ll foo.d
(...)
; Function Attrs: mustprogress nofree nosync nounwind willreturn memory(argmem: readwrite) uwtable
define void @_D3fooQeFZSQj1S(%foo.S* noalias nocapture writeonly sret(%foo.S) align 4 %.sret_arg) local_unnamed_addr #0 {
  %1 = bitcast %foo.S* %.sret_arg to i8*          ; [#uses = 1]
  tail call void @llvm.memcpy.p0i8.p0i8.i64(i8* noundef nonnull align 4 dereferenceable(32) %1, i8* noundef nonnull align 16 dereferenceable(32) bitcast ([8 x i32]* @.arrayliteral to i8*), i64 32, i1 false)
  ret void
}
*/
S foo()
{
    return S([0, 0, 0, 0, 0, 0, 0, 42]);
}
```

## gen/runtime.cpp

LDCではruntime関数を特別扱いしてカスタムで関数属性を付与している。

たとえばいくつかの関数属性について説明を書くと:

- ReadOnly: メモリの読み込みしか起こらないという情報を与える
  - 関数呼び出しを超えたグローバル変数の受け渡しをレジスタで行える
- Cold: ホットパスの関数ではないという情報を与える
  - コンパイラは最適化をしない、コードの配置をCold関数で固めるなどを行う
- NoWind: 例外処理が起きないという情報を与える
  - 関数に余計なunwinding情報を付与しない
- NoCapture: ポインタがキャプチャされていない

```cpp
  // Construct some attribute lists used below (possibly multiple times)
  AttrSet NoAttrs,
      Attr_NoUnwind(NoAttrs, LLAttributeList::FunctionIndex,
                    llvm::Attribute::NoUnwind),
#if LDC_LLVM_VER >= 1600
      Attr_ReadOnly(llvm::AttributeList().addFnAttribute(
          context, llvm::Attribute::getWithMemoryEffects(
                       context, llvm::MemoryEffects::readOnly()))),
#else
      Attr_ReadOnly(NoAttrs, LLAttributeList::FunctionIndex,
                    llvm::Attribute::ReadOnly),
#endif
      Attr_Cold(NoAttrs, LLAttributeList::FunctionIndex, llvm::Attribute::Cold),
      Attr_Cold_NoReturn(Attr_Cold, LLAttributeList::FunctionIndex,
                         llvm::Attribute::NoReturn),
      Attr_Cold_NoReturn_NoUnwind(Attr_Cold_NoReturn,
                                  LLAttributeList::FunctionIndex,
                                  llvm::Attribute::NoUnwind),
      Attr_ReadOnly_NoUnwind(Attr_ReadOnly, LLAttributeList::FunctionIndex,
                             llvm::Attribute::NoUnwind),
      Attr_ReadOnly_1_NoCapture(Attr_ReadOnly, LLAttributeList::FirstArgIndex,
                                llvm::Attribute::NoCapture),
      Attr_ReadOnly_1_3_NoCapture(Attr_ReadOnly_1_NoCapture,
                                  LLAttributeList::FirstArgIndex + 2,
                                  llvm::Attribute::NoCapture),
      Attr_ReadOnly_NoUnwind_1_NoCapture(Attr_ReadOnly_1_NoCapture,
                                         LLAttributeList::FunctionIndex,
                                         llvm::Attribute::NoUnwind),
      Attr_ReadOnly_NoUnwind_1_2_NoCapture(Attr_ReadOnly_NoUnwind_1_NoCapture,
                                           LLAttributeList::FirstArgIndex + 1,
                                           llvm::Attribute::NoCapture),
      Attr_1_NoCapture(NoAttrs, LLAttributeList::FirstArgIndex,
                       llvm::Attribute::NoCapture),
      Attr_1_2_NoCapture(Attr_1_NoCapture, LLAttributeList::FirstArgIndex + 1,
                         llvm::Attribute::NoCapture),
      Attr_1_3_NoCapture(Attr_1_NoCapture, LLAttributeList::FirstArgIndex + 2,
                         llvm::Attribute::NoCapture),
      Attr_1_4_NoCapture(Attr_1_NoCapture, LLAttributeList::FirstArgIndex + 3,
                         llvm::Attribute::NoCapture);
```

例えば以下の関数では、

- C assert: Cold + NoReturn + NoUnwind
  - assertは正常パスではないのであまり実行されない前提で最適化をしたい
  - assert関数は返ってこない
  - C言語のassertは例外処理を扱えない
- D assert: Cold + NoReturn
  - C assertと同様にCold+NoReturnをつけたい
  - ただしD言語の例外なのでUnwinding可能でなければならない

```cpp
  // C assert function
  createFwdDecl(LINK::c, Type::tvoid, {getCAssertFunctionName()},
                getCAssertFunctionParamTypes(), {},
                Attr_Cold_NoReturn_NoUnwind);

  // void _d_assert(string file, uint line)
  // void _d_arraybounds(string file, uint line)
  createFwdDecl(LINK::c, Type::tvoid, {"_d_assert", "_d_arraybounds"},
                {stringTy, uintTy}, {}, Attr_Cold_NoReturn);
```

## 未定義動作

かつでLDCでは以下のコードは未定義動作になった。

```d
noreturn infiniteLoop()
{
    for (;;) {  }
}

void main()
{
    auto x = infiniteLoop();
}
```

現在では[readonlyがwillreturnをinferするのをやめた](https://reviews.llvm.org/D95288)ため、上のコードはちゃんと無限ループになる。

```console
$ ldc2 -O3 --output-ll infiniteloop.d
```

```ll
(...)
; [#uses = 0]
; Function Attrs: nofree norecurse noreturn nosync nounwind memory(none) uwtable
define void @_D12infiniteloop12infiniteLoopFZNn() local_unnamed_addr #0 {
  br label %forcond

forcond:                                          ; preds = %forcond, %0
  br label %forcond
}

; [#uses = 1]
; Function Attrs: nofree norecurse noreturn nosync nounwind memory(none) uwtable
define i32 @_Dmain({ i64, { i64, i8* }* } %unnamed) #1 {
  br label %forcond.i

forcond.i:                                        ; preds = %forcond.i, %0
  br label %forcond.i
}
(...)
```

willreturn属性を付与すると以前の挙動を確認できる。

```d
import ldc.attributes;

@llvmAttr("willreturn")
noreturn infiniteLoop()
{
    for (;;) {  }
}

void main()
{
    auto x = infiniteLoop();
}
```

吐かれるLLVM IRが変わっている。

```ll
(...)
; [#uses = 0]
; Function Attrs: mustprogress nofree norecurse noreturn nosync nounwind willreturn memory(none) uwtable
define void @_D12infiniteloop12infiniteLoopFZNn() local_unnamed_addr #0 {
  unreachable
}

; [#uses = 1]
; Function Attrs: mustprogress nofree norecurse noreturn nosync nounwind willreturn memory(none) uwtable
define i32 @_Dmain({ i64, { i64, i8* }* } %unnamed) #1 {
  unreachable
}
(...)
```

```console
$ ldc2 -O3 infiniteloop.d
$ ./infiniteloop
Segmentation fault
```
