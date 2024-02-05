# 文字列リテラル

D言語の文字列リテラルはLLVM IR、およびそれによって生成されるバイナリ上でどのような表現になるか。

以下の単純な文字列を利用したプログラムについてみていく。

```d
$ cat app.d
import std.stdio;

void main()
{
    writeln("foobar");
    writeln("bar");
}
```

一般的なケースと同様に、文字列リテラルはバイナリ内の.rodataセクションに配置される。

```console
$ ldc2 --linker=lld app.d
$ readelf -p .rodata app| grep bar
  [   a5b]  foobar
  [   ea6]  bar
```

実はLLDはoptimizeフラグを明示的に渡すことで、tail-optimized stringsの最適化をやってくれる。

- lld/OutputSections.cpp

```cpp
static MergeSyntheticSection *createMergeSynthetic(StringRef name,
                                                   uint32_t type,
                                                   uint64_t flags,
                                                   uint32_t addralign) {
  if ((flags & SHF_STRINGS) && config->optimize >= 2)
    return make<MergeTailSection>(name, type, flags, addralign);
  return make<MergeNoTailSection>(name, type, flags, addralign);
}
```

実際に試してみよう。

```console
$ ldc2 --linker=lld -L-O2 app.d
$ readelf -p .rodata app| grep bar
  [   392]  foobar
```

リンカオプションに最適化フラグを与えるとちゃんとmergeしてくれていることがわかる。

次にセクションのフラグ情報をみてみる。

```console
$ ldc2 --output-o app.d
$ readelf -WS app.o | grep \.rodata
  [50] .rodata.str1.1    PROGBITS        0000000000000000 0013df 00000b 01 AMS  0   0  1
  [51] .rodata.str1.16   PROGBITS        0000000000000000 0013f0 000169 01 AMS  0   0 16
$ readelf -p .rodata.str1.1 app.o

String dump of section '.rodata.str1.1':
  [     0]  foobar
  [     7]  bar
```

フラグを表すのはFlgである。
どうやら文字列リテラルを含んでいるセクションのフラグには`SHF_STRINGS`と`SHF_MERGE`はついてないということがわかった。

実はここで`sh_entsize`の値についても確認しておく必要がある。
というのも、リンカはここの値が0の場合mergeableと判定してくれないからだ。

- lld:ELF/InputFiles.cpp

```cpp
template <class ELFT>
bool ObjFile<ELFT>::shouldMerge(const Elf_Shdr &sec, StringRef name) {
  // On a regular link we don't merge sections if -O0 (default is -O1). This
  // sometimes makes the linker significantly faster, although the output will
  // be bigger.
  //
  // Doing the same for -r would create a problem as it would combine sections
  // with different sh_entsize. One option would be to just copy every SHF_MERGE
  // section as is to the output. While this would produce a valid ELF file with
  // usable SHF_MERGE sections, tools like (llvm-)?dwarfdump get confused when
  // they see two .debug_str. We could have separate logic for combining
  // SHF_MERGE sections based both on their name and sh_entsize, but that seems
  // to be more trouble than it is worth. Instead, we just use the regular (-O1)
  // logic for -r.
  if (config->optimize == 0 && !config->relocatable)
    return false;

  // A mergeable section with size 0 is useless because they don't have
  // any data to merge. A mergeable string section with size 0 can be
  // argued as invalid because it doesn't end with a null character.
  // We'll avoid a mess by handling them as if they were non-mergeable.
  if (sec.sh_size == 0)
    return false;

  // Check for sh_entsize. The ELF spec is not clear about the zero
  // sh_entsize. It says that "the member [sh_entsize] contains 0 if
  // the section does not hold a table of fixed-size entries". We know
  // that Rust 1.13 produces a string mergeable section with a zero
  // sh_entsize. Here we just accept it rather than being picky about it.
  uint64_t entSize = sec.sh_entsize;
  if (entSize == 0)
    return false;
```

`sh_entsize`は`readelf -S`の出力のESに対応する。
`.rodata.str1.1`セクションの`sh_entsize`が1になっていることが確認できた。

生成するLLVM IRをみてみよう。

```console
$ ldc2 --output-ll app.d
$ cat app.ll
(省略)
@.str = private unnamed_addr constant [7 x i8] c"foobar\00" ; [#uses = 1]
@.str.1 = private unnamed_addr constant [4 x i8] c"bar\00" ; [#uses = 1]
```

まず、`private unnamed_addr constant`の意味をみていく。

まずprivateはわかりやすいが、これは[linkage types](https://llvm.org/docs/LangRef.html#linkage-types)で同一モジュール内からのアクセスのみが許可されていることを表す。

`unnamed_addr`はちょっとわかりにくいが、[これはアドレスは重要ではなく内容のみが重要である](https://llvm.org/docs/LangRef.html#global-variables)ことを表す。また`constant`もこのセクションで説明されているとおりで、まあ定数となる(変更されない)くらいで解釈して問題ない。

上記記述をみてみると、文字列リテラルはLLVMのGlobal Variableとして表現されているようだ。

次に`[7 x i8] c"foobar\00"`をみる。

`[7 x i8]`は[LLVMのArray型](https://llvm.org/docs/LangRef.html#array-type)に対応しており、8ビット型整数の配列型となる。
そして`c"foobar\00"`は[文字列を表す配列定数](https://llvm.org/docs/LangRef.html#complex-constants)となる。
この2つの情報によって、これは文字列リテラルの本体をLLVMのArray型で表現したものだとわかる。

このGlobalVariableを生成している箇所は`gen/irstate.cpp`のgetCachedStringLiteralImpl関数にある。

- ldc:gen/irstate.cpp

```cpp
template <typename F>
LLGlobalVariable *
getCachedStringLiteralImpl(llvm::Module &module,
                           llvm::StringMap<LLGlobalVariable *> &cache,
                           llvm::StringRef key, F initFactory) {
  auto iter = cache.find(key);
  if (iter != cache.end()) {
    return iter->second;
  }

  LLConstant *constant = initFactory();

  auto gvar =
      new LLGlobalVariable(module, constant->getType(), true,
                           LLGlobalValue::PrivateLinkage, constant, ".str");
  gvar->setUnnamedAddr(LLGlobalValue::UnnamedAddr::Global);

  cache[key] = gvar;

  return gvar;
}
```

GlobalVariableのコンストラクタの定義は以下のようになっている。

- llvm:include/llvm/IR/GlobalVariable.h

```cpp
/// GlobalVariable ctor - This creates a global and inserts it before the
  /// specified other global.
  GlobalVariable(Module &M, Type *Ty, bool isConstant, LinkageTypes Linkage,
                 Constant *Initializer, const Twine &Name = "",
                 GlobalVariable *InsertBefore = nullptr,
                 ThreadLocalMode = NotThreadLocal,
                 std::optional<unsigned> AddressSpace = std::nullopt,
                 bool isExternallyInitialized = false);
```

`Type *Ty`と`Constant *Initializer`が重要そうだ。
なのでinitFactoryにわたってくる関数をみてみる。D言語の文字列リテラルの場合はgen/llvmhelper.cppのbuildStringLiteralConstant関数が渡されてくる。

- ldc:gen/llvmhelper.cpp

```cpp
llvm::Constant *buildStringLiteralConstant(StringExp *se,
                                           uint64_t bufferLength) {
  const auto stringLength = se->numberOfCodeUnits();
  assert(bufferLength >= stringLength);

  if (se->sz == 1 && bufferLength <= stringLength + 1) {
    const DString data = se->peekString();
    const bool nullTerminate = (bufferLength == stringLength + 1);
    return llvm::ConstantDataArray::getString(
        gIR->context(), {data.ptr, data.length}, nullTerminate);
  }

  Type *dtype = se->type->toBasetype();
  Type *cty = dtype->nextOf()->toBasetype();
  LLType *ct = DtoMemType(cty);
  LLArrayType *at = LLArrayType::get(ct, bufferLength);

  std::vector<LLConstant *> vals;
  vals.reserve(bufferLength);
  for (uint64_t i = 0; i < stringLength; ++i) {
    vals.push_back(LLConstantInt::get(ct, se->getCodeUnit(i), false));
  }
  const auto nullChar = LLConstantInt::get(ct, 0, false);
  for (uint64_t i = stringLength; i < bufferLength; ++i) {
    vals.push_back(nullChar);
  }
  return LLConstantArray::get(at, vals);
}
```

文字列リテラルが直接渡されてくる今回の場合は以下のケースだけ着目すればよい。

```cpp
    const DString data = se->peekString();
    const bool nullTerminate = (bufferLength == stringLength + 1);
    return llvm::ConstantDataArray::getString(
        gIR->context(), {data.ptr, data.length}, nullTerminate);
```

なのでここではllvm::ConstantDataArrayが返ってきてる。

constant->getType()が返す型はこのArray型の要素となるので、この場合はint8_tになる。

よってこのセクションは、

- 要素の型がint8_tなConstantDataArray型のconstantなGlobalVariable
- 内部のデータはnull終端な文字列リテラル

で構成されている。

型情報によってmergeable判定を決めている箇所は、LLVMのTargetLoweringObjectFile.cppのgetKindForGlobal関数となっている。

- llvm:lib/Target/TargetLoweringObjectFile.cpp

```cpp
SectionKind TargetLoweringObjectFile::getKindForGlobal(const GlobalObject *GO,
                                                       const TargetMachine &TM){
(省略...)
      // If initializer is a null-terminated string, put it in a "cstring"
      // section of the right width.
      if (ArrayType *ATy = dyn_cast<ArrayType>(C->getType())) {
        if (IntegerType *ITy =
              dyn_cast<IntegerType>(ATy->getElementType())) {
          if ((ITy->getBitWidth() == 8 || ITy->getBitWidth() == 16 ||
               ITy->getBitWidth() == 32) &&
              IsNullTerminatedString(C)) {
            if (ITy->getBitWidth() == 8)
              return SectionKind::getMergeable1ByteCString();
            if (ITy->getBitWidth() == 16)
              return SectionKind::getMergeable2ByteCString();

            assert(ITy->getBitWidth() == 32 && "Unknown width");
            return SectionKind::getMergeable4ByteCString();
          }
        }
```

SectionKind::Mergeable1ByteCStringやSectionKind::Mergeable4ByteCStringを返す条件として、セクションの型が整数型のArrayTypeにキャスト可能な型であること・セクション内の文字列がnull終端となっていることなどが求められている。

これらのSectionKindは`SHF_MERGE`と`SHF_STRINGS`に影響することが以下の関数でわかる。

- llvm:lib/CodeGen/TargetLoweringObjectFileImpl.cpp

```cpp
static unsigned getELFSectionFlags(SectionKind K) {
  unsigned Flags = 0;
(省略...)
  if (K.isMergeableCString() || K.isMergeableConst())
    Flags |= ELF::SHF_MERGE;

  if (K.isMergeableCString())
    Flags |= ELF::SHF_STRINGS;

  return Flags;
}
```

このようにして、D言語の文字列リテラルはtail-optimizedなmergeが可能となっている。
