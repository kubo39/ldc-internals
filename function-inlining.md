# インライン化について

## cross module inlining

LDCはinlineヒント(`pragma(inline, true)`であれ`pragma(LDC_always_inline)`であれ)があるか、`willCrossModuleInline`がtrueの場合はcross module inliningを行うようになっている。

- https://github.com/ldc-developers/ldc/blob/32f6b5ba62429335c92a260265ae4060ce8342da/gen/function-inlining.cpp#L94-L112

```cpp
bool defineAsExternallyAvailable(FuncDeclaration &fdecl) {
  IF_LOG Logger::println("Enter defineAsExternallyAvailable");
  LOG_SCOPE

  // Implementation note: try to do cheap checks first.

  if (fdecl.neverInline || fdecl.inlining == PINLINE::never) {
    IF_LOG Logger::println("pragma(inline, false) specified");
    return false;
  }

  // pragma(inline, true) functions will be inlined even at -O0
  if (fdecl.inlining == PINLINE::always) {
    IF_LOG Logger::println(
        "pragma(inline, true) specified, overrides cmdline flags");
  } else if (!willCrossModuleInline()) {
    IF_LOG Logger::println("Commandline flags indicate no inlining");
    return false;
  }
```

`willCrossModuleInline`は`cross-module-inlining`コマンドラインオプションが有効かつコンパイラ内のインラインコスト計算でインライン可能であると判定した場合にtrueを返す。
(`cross-module-inlining`は隠しフラグになっている)

- https://github.com/ldc-developers/ldc/blob/32f6b5ba62429335c92a260265ae4060ce8342da/gen/optimizer.cpp#L167-L173

```cpp
bool willCrossModuleInline() {
#ifdef IN_JITRT
  return false;
#else
  return enableCrossModuleInlining == llvm::cl::BOU_TRUE && willInline();
#endif
}
```

cross-module inliningを行う場合、関数定義に影響する。

- https://github.com/ldc-developers/ldc/blob/32f6b5ba62429335c92a260265ae4060ce8342da/gen/functions.cpp#L540-L544

```cpp
  } else if (defineAsExternallyAvailable(*fdecl)) {
    Logger::println("Function is an externally_available inline candidate, "
                    "will be defined after declaration.");
    defineAtEnd = true;
    defineAsAvailableExternally = true;
  }
```

最終的にLLVM側に情報を渡すのはこちら

- https://github.com/ldc-developers/ldc/blob/32f6b5ba62429335c92a260265ae4060ce8342da/gen/functions.cpp#L1147-L1154

```cpp
void DtoDefineFunction(FuncDeclaration *fd, bool linkageAvailableExternally) {
(...)
  if (linkageAvailableExternally) {
    func->setLinkage(llvm::GlobalValue::AvailableExternallyLinkage);
    func->setDLLStorageClass(llvm::GlobalValue::DefaultStorageClass);
    // Assert that we are not overriding a linkage type that disallows inlining
    assert(lwc.first != llvm::GlobalValue::WeakAnyLinkage &&
           lwc.first != llvm::GlobalValue::ExternalWeakLinkage &&
           lwc.first != llvm::GlobalValue::LinkOnceAnyLinkage);
  } else {
(...)
```
