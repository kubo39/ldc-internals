# naked function

## `@naked`

LLVMのnaked関数属性と対応している。

- runtime/druntime/src/ldc/attributes.d

```d
/**
 * Adds LLVM's "naked" attribute to a function, disabling function prologue /
 * epilogue emission, incl. LDC's.
 * Intended to be used in combination with a function body defined via
 * ldc.llvmasm.__asm() and/or ldc.simd.inlineIR().
 */
enum naked = llvmAttr("naked");
```

- gen/functions.cpp

```cpp
  // @naked: emit body and return, no prologue/epilogue
  if (func->hasFnAttribute(llvm::Attribute::Naked)) {
    Statement_toIR(fd->fbody, gIR);
    const bool wasDummy = eraseDummyAfterReturnBB(gIR->scopebb());
    if (!wasDummy && !gIR->scopereturned()) {
      // this is what clang does to prevent LLVM complaining about
      // non-terminated function
      gIR->ir->CreateUnreachable();
    }
    return;
  }
```

RISC-V 32bit向けに実際にアセンブリを生成してみる。

```d
import ldc.attributes : naked;
import ldc.llvmasm;

pragma(mangle, "foo")
@naked void foo()
{
  __asm(`
      ret
  `, "");
}
```

```console
ldc2 -mtriple=riscv32-unknown-none-elf -betterC --output-s app.d
```

```s
        .text
        .attribute      4, 16
        .attribute      5, "rv32i2p0"
        .file   "app.d"
        .section        .text.foo,"ax",@progbits
        .globl  foo
        .p2align        2
        .type   foo,@function
foo:
        .cfi_startproc
        #APP
        ret
        #NO_APP
.Lfunc_end0:
        .size   foo, .Lfunc_end0-foo
        .cfi_endproc

        .ident  "ldc version 1.35.0"
        .section        ".note.GNU-stack","",@progbits
```

## D's x86 Inline Assembler

[D言語のInlineAssembler構文のnaked](https://dlang.org/spec/iasm.html#naked)はLLVMのNaked関数属性ではなくModuleInlineAsmとして実現されている。

```cpp
  gIR->module.appendModuleInlineAsm(asmstr.str());
```

そのほかで特徴はOS環境ごとにASMディレクティブを用いて関数として実装している。以下はLinuxやベアメタルの例。

```cpp
  if (isDarwin) {
(...)
  else if (isWin) {
(...)
  } else {
    if (fd->isInstantiated()) {
      asmstr << "\t.section\t.text." << mangle << ",\"axG\",@progbits,"
             << mangle << ",comdat" << std::endl;
      asmstr << "\t.weak\t" << mangle << std::endl;
    } else {
      asmstr << "\t.text" << std::endl;
      asmstr << "\t.globl\t" << mangle << std::endl;
    }
    asmstr << "\t.p2align\t4, 0x90" << std::endl;
    asmstr << "\t.type\t" << mangle << ",@function" << std::endl;
    asmstr << mangle << ":" << std::endl;
  }
```
