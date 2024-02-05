# moduleinfo

ここになんの変哲もないHello, World!を出力するお決まりのプログラムがある。

```d
import std.stdio;

void main()
{
    writeln("Hello, World!");
}
```

これを普通に`ldc2 hello.d`コンパイルして生成したバイナリは以下のようなセクションを含んでいる。

```console
$ readelf -WS hello
There are 35 section headers, starting at offset 0x119f10:

Section Headers:
  [Nr] Name              Type            Address          Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            0000000000000000 000000 000000 00      0   0  0
  [ 1] .interp           PROGBITS        0000000000000350 000350 00001c 00   A  0   0  1
  [ 2] .note.gnu.property NOTE            0000000000000370 000370 000020 00   A  0   0  8
  [ 3] .note.gnu.build-id NOTE            0000000000000390 000390 000024 00   A  0   0  4
  [ 4] .note.ABI-tag     NOTE            00000000000003b4 0003b4 000020 00   A  0   0  4
  [ 5] .gnu.hash         GNU_HASH        00000000000003d8 0003d8 000024 00   A  6   0  8
  [ 6] .dynsym           DYNSYM          0000000000000400 000400 000ea0 18   A  7   1  8
  [ 7] .dynstr           STRTAB          00000000000012a0 0012a0 000815 00   A  0   0  1
  [ 8] .gnu.version      VERSYM          0000000000001ab6 001ab6 000138 02   A  6   0  2
  [ 9] .gnu.version_r    VERNEED         0000000000001bf0 001bf0 000100 00   A  7   4  8
  [10] .rela.dyn         RELA            0000000000001cf0 001cf0 021138 18   A  6   0  8
  [11] .rela.plt         RELA            0000000000022e28 022e28 000d68 18  AI  6  27  8
  [12] .init             PROGBITS        0000000000024000 024000 00001b 00  AX  0   0  4
  [13] .plt              PROGBITS        0000000000024020 024020 000900 10  AX  0   0 16
  [14] .plt.got          PROGBITS        0000000000024920 024920 000008 08  AX  0   0  8
  [15] .text             PROGBITS        0000000000024930 024930 057670 00  AX  0   0 16
  [16] .fini             PROGBITS        000000000007bfa0 07bfa0 00000d 00  AX  0   0  4
  [17] .rodata           PROGBITS        000000000007c000 07c000 018e57 00   A  0   0 16
  [18] .eh_frame_hdr     PROGBITS        0000000000094e58 094e58 004994 00   A  0   0  4
  [19] .eh_frame         PROGBITS        00000000000997f0 0997f0 013e60 00   A  0   0  8
  [20] .gcc_except_table PROGBITS        00000000000ad650 0ad650 000bc8 00   A  0   0  4
  [21] .tdata            PROGBITS        00000000000af400 0ae400 000008 00 WATo  0   0 16
  [22] .tbss             NOBITS          00000000000af410 0ae408 0001c1 00 WAT  0   0 16
  [23] .init_array       INIT_ARRAY      00000000000af410 0ae410 000040 08  WA  0   0  8
  [24] .fini_array       FINI_ARRAY      00000000000af450 0ae450 000010 08  WA  0   0  8
  [25] .data.rel.ro      PROGBITS        00000000000af460 0ae460 00a470 00  WA  0   0 16
  [26] .dynamic          DYNAMIC         00000000000b98d0 0b88d0 000220 10  WA  7   0  8
  [27] .got              PROGBITS        00000000000b9af0 0b8af0 000510 08  WA  0   0  8
  [28] .data             PROGBITS        00000000000ba000 0b9000 00db70 00  WA  0   0 64
  [29] __minfo           PROGBITS        00000000000c7b70 0c6b70 0004d8 00 WAo  0   0  8
  [30] .bss              NOBITS          00000000000c8048 0c7048 0006e0 00  WA  0   0  8
  [31] .comment          PROGBITS        0000000000000000 0c7048 000053 01  MS  0   0  1
  [32] .symtab           SYMTAB          0000000000000000 0c70a0 0186a8 18     33 514  8
  [33] .strtab           STRTAB          0000000000000000 0df748 03a681 00      0   0  1
  [34] .shstrtab         STRTAB          0000000000000000 119dc9 000145 00      0   0  1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  D (mbind), l (large), p (processor specific)
```

大半はGCCやclangといったC言語のコンパイラで生成したバイナリと違いはないが、LDCの場合はここで`__minfo`というセクションが生えていることに気がつく。

先に説明してしまうと、このセクションはD言語のModuleInfoへの参照を格納している。このセクションを作ってるコードはgen/modules.cppにある。

```cpp
// Emits a pointer to the specified ModuleInfo into the special
// .minfo (COFF & MachO) / __minfo section.
void emitModuleRefToSection(std::string moduleMangle,
                            llvm::Constant *thisModuleInfo) {
  const auto moduleInfoPtrTy = DtoPtrToType(getModuleInfoType());

  const auto &triple = *global.params.targetTriple;
  const auto sectionName =
      triple.isOSBinFormatCOFF()
          ? ".minfo"
          : triple.isOSBinFormatMachO() ? "__DATA,.minfo" : "__minfo";

  const auto thismrefIRMangle =
      getIRMangledModuleRefSymbolName(moduleMangle.c_str());
  auto thismref = defineGlobal(Loc(), gIR->module, thismrefIRMangle,
                               DtoBitCast(thisModuleInfo, moduleInfoPtrTy),
                               LLGlobalValue::LinkOnceODRLinkage, false, false);
  thismref->setVisibility(LLGlobalValue::HiddenVisibility);
  thismref->setSection(sectionName);
  gIR->usedArray.push_back(thismref);
}
```

LLVM IRはこのようになっている。こちらをみたほうがわかりやすいかもしれない。
`@llvm.used`を使っているのは最適化で消されるのを阻止するため。

```ll
(snip...)
@_D5hello12__ModuleInfoZ = global %0 { i32 -2147483644, i32 0, [6 x i8] c"hello\00" } ; [#uses = 1]
@_D5hello11__moduleRefZ = linkonce_odr hidden global ptr @_D5hello12__ModuleInfoZ, section "__minfo" ; [#uses = 1]
@llvm.used = appending global [1 x ptr] [ptr @_D5hello11__moduleRefZ], section "llvm.metadata" ; [#uses = 0]
(snip...)
```
