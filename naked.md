# naked function

LDCのnaked functionはLLVMのNaked関数属性ではなくModuleInlineAsmとして実現されている。

```cpp
  gIR->module.appendModuleInlineAsm(asmstr.str());
```

そのほかで特徴はOS環境ごとにASMディレクティブを用いて関数として実装している。

## Align Directive

ここではLinuxやベアメタルに絞って内容をみていく。

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

たとえばRISC-V 32bit向けに実際にアセンブリを生成してみると、少し上のコードとは違った内容が出力されていることがわかる。

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

一番大きな違いは`\t.p2align\t4, 0x90`が`\t.p2align\t2`に変わっている点である。

LLVMがAlign Directiveを出力する箇所は以下のようになっている。

```cpp
void MCAsmStreamer::emitAlignmentDirective(unsigned ByteAlignment,
                                           std::optional<int64_t> Value,
                                           unsigned ValueSize,
                                           unsigned MaxBytesToEmit) {
  if (MAI->useDotAlignForAlignment()) {
    if (!isPowerOf2_32(ByteAlignment))
      report_fatal_error("Only power-of-two alignments are supported "
                         "with .align.");
    OS << "\t.align\t";
    OS << Log2_32(ByteAlignment);
    EmitEOL();
    return;
  }

  // Some assemblers don't support non-power of two alignments, so we always
  // emit alignments as a power of two if possible.
  if (isPowerOf2_32(ByteAlignment)) {
    switch (ValueSize) {
    default:
      llvm_unreachable("Invalid size for machine code value!");
    case 1:
      OS << "\t.p2align\t";
      break;
    case 2:
      OS << ".p2alignw ";
      break;
    case 4:
      OS << ".p2alignl ";
      break;
    case 8:
      llvm_unreachable("Unsupported alignment size!");
    }

    OS << Log2_32(ByteAlignment);

    if (Value.has_value() || MaxBytesToEmit) {
      if (Value.has_value()) {
        OS << ", 0x";
        OS.write_hex(truncateToSize(*Value, ValueSize));
      } else {
        OS << ", ";
      }

      if (MaxBytesToEmit)
        OS << ", " << MaxBytesToEmit;
    }
    EmitEOL();
    return;
  }
```

[p2align directive](https://sourceware.org/binutils/docs/as/P2align.html)の説明をみたところ、2つめのオペランドは多くのアーキテクチャではnop命令がくるようだ。`0x90`はx86のnop命令だがRISC-Vではnop命令は違っているため、
単に無視されてしまっているのだろう。

実際に`x86_64`では`.p2align\t, 0x90`が出力されている。

```console
ldc2 -betterC --output-s app.d
```

```s
        .text
        .file   "app.d"
        .section        .text.foo,"ax",@progbits
        .globl  foo
        .p2align        4, 0x90
        .type   foo,@function
foo:
        .cfi_startproc
        #APP
        retq
        #NO_APP
.Lfunc_end0:
        .size   foo, .Lfunc_end0-foo
        .cfi_endproc

        .ident  "ldc version 1.35.0"
        .section        ".note.GNU-stack","",@progbits
```
