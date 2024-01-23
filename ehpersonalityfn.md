# eh personality

LLVM17の時点では `llvm/include/llvm/IR/EHPersonalities.h` の定義にD言語がない。

```cpp
enum class EHPersonality {
  Unknown,
  GNU_Ada,
  GNU_C,
  GNU_C_SjLj,
  GNU_CXX,
  GNU_CXX_SjLj,
  GNU_ObjC,
  MSVC_X86SEH,
  MSVC_TableSEH,
  MSVC_CXX,
  CoreCLR,
  Rust,
  Wasm_CXX,
  XL_CXX
};
```

Unknownになるかわからないけど、このへんの最適化とか効く可能性がある？

```cpp
/// Return true if this personality may be safely removed if there
/// are no invoke instructions remaining in the current function.
inline bool isNoOpWithoutInvoke(EHPersonality Pers) {
  switch (Pers) {
  case EHPersonality::Unknown:
    return false;
  // All known personalities currently have this behavior
  default:
    return true;
  }
  llvm_unreachable("invalid enum");
}
```

## LDCではどのpersonality関数が設定されるのか

デフォルト実装がどうなるか。

以下のふたつの関数が関わってきそう。

classifyEHPersonality関数の場合: Unknownが返ってきそう。

```cpp
/// See if the given exception handling personality function is one that we
/// understand.  If so, return a description of it; otherwise return Unknown.
EHPersonality llvm::classifyEHPersonality(const Value *Pers) {
  const GlobalValue *F =
      Pers ? dyn_cast<GlobalValue>(Pers->stripPointerCasts()) : nullptr;
  if (!F || !F->getValueType() || !F->getValueType()->isFunctionTy())
    return EHPersonality::Unknown;
  return StringSwitch<EHPersonality>(F->getName())
      .Case("__gnat_eh_personality", EHPersonality::GNU_Ada)
      .Case("__gxx_personality_v0", EHPersonality::GNU_CXX)
      .Case("__gxx_personality_seh0", EHPersonality::GNU_CXX)
      .Case("__gxx_personality_sj0", EHPersonality::GNU_CXX_SjLj)
      .Case("__gcc_personality_v0", EHPersonality::GNU_C)
      .Case("__gcc_personality_seh0", EHPersonality::GNU_C)
      .Case("__gcc_personality_sj0", EHPersonality::GNU_C_SjLj)
      .Case("__objc_personality_v0", EHPersonality::GNU_ObjC)
      .Case("_except_handler3", EHPersonality::MSVC_X86SEH)
      .Case("_except_handler4", EHPersonality::MSVC_X86SEH)
      .Case("__C_specific_handler", EHPersonality::MSVC_TableSEH)
      .Case("__CxxFrameHandler3", EHPersonality::MSVC_CXX)
      .Case("ProcessCLRException", EHPersonality::CoreCLR)
      .Case("rust_eh_personality", EHPersonality::Rust)
      .Case("__gxx_wasm_personality_v0", EHPersonality::Wasm_CXX)
      .Case("__xlcxx_personality_v1", EHPersonality::XL_CXX)
      .Default(EHPersonality::Unknown);
}
```

getDefaultEHPersonality関数だとGNU Cスタイル。

```cpp
EHPersonality llvm::getDefaultEHPersonality(const Triple &T) {
  if (T.isPS5())
    return EHPersonality::GNU_CXX;
  else
    return EHPersonality::GNU_C;
}
```

getDefaultEHPersonality関数はEscapeEnumeratorというところでしか使われていない。

```cpp
static FunctionCallee getDefaultPersonalityFn(Module *M) {
  LLVMContext &C = M->getContext();
  Triple T(M->getTargetTriple());
  EHPersonality Pers = getDefaultEHPersonality(T);
  return M->getOrInsertFunction(getEHPersonalityName(Pers),
                                FunctionType::get(Type::getInt32Ty(C), true));
}
```

ただし、このコードはShadow stack GCとthread sanitizerでしか使われていないらしい。

```console
$ rg EscapeEnumerator ./llvm/lib/
./llvm/lib/CodeGen/ShadowStackGCLowering.cpp
40:#include "llvm/Transforms/Utils/EscapeEnumerator.h"
362:  EscapeEnumerator EE(F, "gc_cleanup", /*HandleExceptions=*/true,

./llvm/lib/Transforms/Utils/CMakeLists.txt
25:  EscapeEnumerator.cpp

./llvm/lib/Transforms/Utils/EscapeEnumerator.cpp
1://===- EscapeEnumerator.cpp -----------------------------------------------===//
14:#include "llvm/Transforms/Utils/EscapeEnumerator.h"
30:IRBuilder<> *EscapeEnumerator::Next() {

./llvm/lib/Transforms/Instrumentation/ThreadSanitizer.cpp
47:#include "llvm/Transforms/Utils/EscapeEnumerator.h"
483:  EscapeEnumerator EE(F, "tsan_ignore_cleanup", ClHandleCxxExceptions);
577:    EscapeEnumerator EE(F, "tsan_cleanup", ClHandleCxxExceptions);
```

Shadow Stack GCについては、これは[LLVMが提供しているGC APIのうちのひとつ](https://llvm.org/docs/GarbageCollection.html#the-shadow-stack-gc)で、LDCはこれを利用していない。

Thread Sanitizerに関してはsanitizerオプションとして提供しているが、普通のプログラムの場合はこの実装は考慮しなくてもよさそうだ。

## classifyEHPersonality関数

そうなるとclassifyEHPersonality関数のほうが気になってくるが、この関数に渡す文字列はどこからくるだろうか。

いろいろ利用箇所をざっくりみたが、先述したisNoOpWithoutInvoke関数との組み合わせであるXCOFFフォーマットでEHBlockを生成するかの判定箇所で使われているものが関係ありそうな箇所だった。

```cpp
//===----------------------------------------------------------------------===//
//                                  XCOFF
//===----------------------------------------------------------------------===//
bool TargetLoweringObjectFileXCOFF::ShouldEmitEHBlock(
    const MachineFunction *MF) {
  if (!MF->getLandingPads().empty())
    return true;

  const Function &F = MF->getFunction();
  if (!F.hasPersonalityFn() || !F.needsUnwindTableEntry())
    return false;

  const GlobalValue *Per =
      dyn_cast<GlobalValue>(F.getPersonalityFn()->stripPointerCasts());
  assert(Per && "Personality routine is not a GlobalValue type.");
  if (isNoOpWithoutInvoke(classifyEHPersonality(Per)))
    return false;

  return true;
}
```

上の関数ではgetPersonalityFn関数でpersonality関数の取得を行っている。

llvm/lib/IR/Function.cppではpersonality関数を設定する関数がある。

```cpp
Constant *Function::getPersonalityFn() const {
  assert(hasPersonalityFn() && getNumOperands());
  return cast<Constant>(Op<0>());
}

void Function::setPersonalityFn(Constant *Fn) {
  setHungoffOperand<0>(Fn);
  setValueSubclassDataBit(3, Fn != nullptr);
}
```

## LDC側からみるpersonality関数の設置

設定はLDC側で行っているはずだ。

LDC内ではsetLLVMPersonalityFn関数でラップしている。

```cpp
void IrFunction::setLLVMPersonalityFn(llvm::Constant *personality) {
  assert(func != nullptr);
  func->setPersonalityFn(personality);
}
```

gen/trycatchfinally.cppファイル内で、MSVC向けとDランタイムでlandingPadの生成ロジックがわかれている。

```cpp
namespace {
  llvm::LandingPadInst *createLandingPadInst(IRState &irs) {
    LLType *retType = LLStructType::get(LLType::getInt8PtrTy(irs.context()),
                                        LLType::getInt32Ty(irs.context()));
    if (!irs.func()->hasLLVMPersonalityFn()) {
      irs.func()->setLLVMPersonalityFn(
          getRuntimeFunction(Loc(), irs.module, "_d_eh_personality"));
    }
  return irs.ir->CreateLandingPad(retType, 0);
}
}

llvm::BasicBlock *TryCatchFinallyScopes::emitLandingPad() {
  if (useMSVCEH()) {
    assert(currentCleanupScope() > 0);
    return emitLandingPadMSVC(currentCleanupScope() - 1);
  }

  // save and rewrite scope
  const auto savedInsertPoint = irs.saveInsertPoint();

  // insert landing pads at the end of the function, in emission order,
  // to improve human-readability of the IR
  llvm::BasicBlock *beginBB = irs.insertBBBefore(nullptr, "landingPad");
  irs.ir->SetInsertPoint(beginBB);

  llvm::LandingPadInst *landingPad = createLandingPadInst(irs);

  // Stash away the exception object pointer and selector value into their
  // stack slots.
  llvm::Value *ehPtr = DtoExtractValue(landingPad, 0);
  irs.ir->CreateStore(ehPtr, getOrCreateEhPtrSlot());
(snip...)
```

## ここまでの論点

- LLVM17ではEHPersonalityの定義にD言語は含まれていない
- EHPersonality::Unknownでほとんどのケースでは影響はない
- XCOFFの場合、ShouldEmitEHBlock関数の結果がtrueになってしまうことで最適化に影響が出てしまうかもしれない
- 他にも例外情報(landingPadsなど)を生成する箇所で最適化の影響が出るかもしれない
