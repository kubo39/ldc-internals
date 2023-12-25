# ldc.llvmasm

- [LDC Wiki](https://wiki.dlang.org/LDC_inline_assembly_expressions)

LDCのインラインアセンブラはほぼLLVMへ素通しするようになっている。そのためLLVMインラインアセンブラの構文(constraintsなど)はそのまま利用できる。

- [LLVM Inline Assembly Expressions](https://llvm.org/docs/LangRef.html#inline-assembler-expressions)

`__asm`は関数なので、関数の実装からくる。以下の`ToElemVisitor::call`からInlineAsmの構築に入っていく。

```cpp
if (auto ce = rhs->isCallExp()) {
  // Direct construction by rhs call via sret?
  // E.g., `T v = foo();` if the callee `T foo()` uses sret.
  // In this case, pass `&v` as hidden sret argument, i.e., let `foo()`
  // construct the return value directly into the lhs lvalue.
  if (DtoIsReturnInArg(ce)) {
    Logger::println("success, in-place-constructing sret return value");
    ToElemVisitor::call(gIR, ce, DtoLVal(lhs));
    return true;
  }
```

LDCのインラインアセンブラは関数宣言を内部的に特殊扱いしていることで特別なパスにくるようになっている。

```cpp
// handle magic inline asm
if (auto ve = e->e1->isVarExp()) {
  if (auto fd = ve->var->isFuncDeclaration()) {
    if (fd->llvmInternal == LLVMinline_asm) {
      return DtoInlineAsmExpr(e->loc, fd, e->arguments, sretPointer);
    }
    if (fd->llvmInternal == LLVMinline_ir) {
      return DtoInlineIRExpr(e->loc, fd, e->arguments, sretPointer);
    }
  }
}
```

constraintsのパースはLLVMの提供しているものを使っている。
LLVMの持つConstraintInfoは各オペランドがInputかOutputかClobberか、Constraintsの数はいくつか、などの情報を持つ。

```cpp
const DString constraintsStr = se->peekString();
const llvm::StringRef constraints = {constraintsStr.ptr,
                                     constraintsStr.length};

auto constraintInfo = llvm::InlineAsm::ParseConstraints(constraints);
```

LLVMを呼ぶところでは常に`sideeeffect=true`としている。これはvolatile相当なので、LDCのインラインアセンブラは常にvolatileつきとなっている。

```cpp
// build asm call
bool sideeffect = true;
llvm::InlineAsm *ia = llvm::InlineAsm::get(FT, code, constraints, sideeffect);
```

ここでは省略されているが、IsAlignStack/AsmDialect/HasUnwindClobberがそれぞれデフォルトの値が設定されている。これはLDC側からは変更する手段を提供していない。

clangなどと同様に、`llvm::InlineAsm`から`llvm::CallInst`を構築する。

```cpp
auto call = gIR->createInlineAsmCall(loc, ia, operands, indirectTypes);
```

LLVMのCallInstを構築するだけで、他はソースコードの行対応を行う処理くらい。
関数属性などはいじらない。

```cpp
llvm::CallInst *
IRState::createInlineAsmCall(const Loc &loc, llvm::InlineAsm *ia,
                             llvm::ArrayRef<llvm::Value *> args,
                             llvm::ArrayRef<llvm::Type *> indirectTypes) {
  llvm::CallInst *call = ir->CreateCall(ia, args);
  addInlineAsmSrcLoc(loc, call);
```

LDC inlime assemblyにsideeffectが付与されていることは以下のコードで確認できる。

```d
import ldc.llvmasm;

/**
// ldc2 -O --output-ll llvmasm.d
(...)
; Function Attrs: uwtable
define void @_D7llvmasm3fooFZv() local_unnamed_addr #0 {
  tail call void asm sideeffect "nop", "~{memory}"() #1, !srcloc !1
  ret void
(...)
attributes #1 = { nounwind }
*/
void foo()
{
    __asm("nop", "~{memory}");
}
```
