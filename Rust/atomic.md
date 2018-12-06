# core::sync::atomic

## compare and swap

x86にはcompare and swapが〜

## RISC-V Atomic Extension

RISC-Vには、compare and swap命令が存在していません。

### lr/sc

load reserved/store conditionalという命令がatomicなロードストア命令として定義されています。

## Dive into the Rust core library

Atomic*であれば何でも良いので、一旦、`AtomicBool`の実装を解析していきましょう。
これはRust core libraryの中にあります。

:libcore/sync/atomic.rs

```rust
#[cfg(target_has_atomic = "8")]
impl AtomicBool {
...
    #[inline]
    #[stable(feature = "rust1", since = "1.0.0")]
    #[cfg(target_has_atomic = "cas")]
    pub fn compare_and_swap(&self, current: bool, new: bool, order: Ordering) -> bool {
        match self.compare_exchange(current, new, order, strongest_failure_ordering(order)) {
            Ok(x) => x,
            Err(x) => x,
        }
    }
```

いきなり重要なアトリビュートが見つかります。
`#[cfg(target_has_atomic = "cas")]`、この部分ですね。`cas`は、compare_and_swapの略で、命令セットの記述で頻繁に用いられます。

この`target_has_atomic`がどこでどのように定義されているか、を追いかける必要がありそうです。
現在のRustでは、2つのRISC-Vアーキテクチャがサポートされています。

[riscv32imc_unknown_none_elf.rs](https://github.com/rust-lang/rust/blob/master/src/librustc_target/spec/riscv32imc_unknown_none_elf.rs)
[riscv32imac_unknown_none_elf.rs](https://github.com/rust-lang/rust/blob/master/src/librustc_target/spec/riscv32imac_unknown_none_elf.rs)

1つは、Atomic命令をサポートしない`rv32imc`です。こちらをターゲットにする場合は、compare and swapがなくても納得です。
target tripleを見ても、`atomic_cas`が`false`になっています。

```rust
pub fn target() -> TargetResult {
...
        options: TargetOptions {
            linker: Some("rust-lld".to_string()),
            cpu: "generic-rv32".to_string(),
            // https://gcc.gnu.org/bugzilla/show_bug.cgi?id=86005
            max_atomic_width: None, //Some(32),
            atomic_cas: false,  // atomic_casはdisableされている？
            features: "+m,+c".to_string(),
...
        },
```

これに対して、Atomic命令をサポートする`rv32imac`(`a`の追加がAtomic命令サポートの意味)では、compare and swapが使えても良さそうです。
上述したとおり、lr/scでcompare and swapが実現できるためです。

```rust
pub fn target() -> TargetResult {
...
        options: TargetOptions {
            linker: Some("rust-lld".to_string()),
            cpu: "generic-rv32".to_string(),
            max_atomic_width: Some(32),
            atomic_cas: false, // incomplete +a extension
            features: "+m,+a,+c".to_string(),
...
        },
```

何やら、`// incomplete +a extension`なる不穏なコメントがあります(筆者が追記したものではなく、元々ソースコードに存在するコメントです)。
ここの`atomic_cas`が`false`なので、`compare_and_swap`のアトリビュートで弾かれている、と考えるのが自然です。

`atomic_cas: false`が`target_has_atomic = "cas"`に変換される過程も追いたいところですが、一旦先に進みましょう。

それはそうとして、compare_and_swap関数も中身を追っておきましょう。
`compare_exchange`に処理を移譲して、その結果がOk/Err、どちらでもその中身を返しています。

`compare_exchange`の中身に移ります。

:libcore/sync/atomic.rs

```rust
impl AtomicBool {
    #[inline]
    #[stable(feature = "extended_compare_and_swap", since = "1.10.0")]
    #[cfg(target_has_atomic = "cas")]
    pub fn compare_exchange(&self,
                            current: bool,
                            new: bool,
                            success: Ordering,
                            failure: Ordering)
                            -> Result<bool, bool> {
        match unsafe {
            atomic_compare_exchange(self.v.get(), current as u8, new as u8, success, failure)
        } {
            Ok(x) => Ok(x != 0),
            Err(x) => Err(x != 0),
        }
    }
```

`unsafe`な`atomic_compare_exchange`が出てきましたね。これは、機械語が近そうな予感がします。
次に、`atomic_compare_exchange`を確認していきましょう。
これは、Atomic系で共通するジェネリック関数として定義されています。

```rust
#[inline]
#[cfg(target_has_atomic = "cas")]
unsafe fn atomic_compare_exchange<T>(dst: *mut T,
                                     old: T,
                                     new: T,
                                     success: Ordering,
                                     failure: Ordering)
                                     -> Result<T, T> {
    let (val, ok) = match (success, failure) {
        (Acquire, Acquire) => intrinsics::atomic_cxchg_acq(dst, old, new),
        (Release, Relaxed) => intrinsics::atomic_cxchg_rel(dst, old, new),
        (AcqRel, Acquire) => intrinsics::atomic_cxchg_acqrel(dst, old, new),
        (Relaxed, Relaxed) => intrinsics::atomic_cxchg_relaxed(dst, old, new),
        (SeqCst, SeqCst) => intrinsics::atomic_cxchg(dst, old, new),
        (Acquire, Relaxed) => intrinsics::atomic_cxchg_acq_failrelaxed(dst, old, new),
        (AcqRel, Relaxed) => intrinsics::atomic_cxchg_acqrel_failrelaxed(dst, old, new),
        (SeqCst, Relaxed) => intrinsics::atomic_cxchg_failrelaxed(dst, old, new),
        (SeqCst, Acquire) => intrinsics::atomic_cxchg_failacq(dst, old, new),
        (_, AcqRel) => panic!("there is no such thing as an acquire/release failure ordering"),
        (_, Release) => panic!("there is no such thing as a release failure ordering"),
        _ => panic!("a failure ordering can't be stronger than a success ordering"),
    };
    if ok { Ok(val) } else { Err(val) }
}
```

compare and swapは、compare_exchangeを経て、`cxchg`と略されています。
intrinsics、の意味を知らなかったので、google先生に聞いてみます。
**本来備わっている、固有の、本質的な** という意味のようです。わかったような、わからないような気がするため、もう少し調べてみました。

[What are intrinsics?](https://stackoverflow.com/questions/2268562/what-are-intrinsics)

> Normally, "intrinsics" refers to functions that are built-in

なるほど。built-inで、コンパイラが最適化したコードを生成できる機能であることを意味するようです。腑に落ちました。
これは、LLVMの[intrinsic](https://llvm.org/docs/ExtendingLLVM.html#intrinsic-function)に関係あることが推測できます。
最終的にはLLVMに関係がありそうですが、まだRustのコードで見るところがあります。atomic.rsでは、`use intrinsics`としています。こいつは何者でしょうか？

```rust
use intrinsics;
```

[Rust doc std::intrinsics](https://doc.rust-lang.org/std/intrinsics/index.html)

> The corresponding definitions are in librustc_codegen_llvm/intrinsic.rs.

お、LLVM codegenへの言及がなされています。
これは、本格的にLLVMが原因で、RISC-Vでspinが使えない予感がします。

`src/libcore/intrinsics.rs`に進みましょう。

```rsut
extern "rust-intrinsic" {
    // NB: These intrinsics take raw pointers because they mutate aliased
    // memory, which is not valid for either `&` or `&mut`.

    /// Stores a value if the current value is the same as the `old` value.
    /// The stabilized version of this intrinsic is available on the
    /// `std::sync::atomic` types via the `compare_exchange` method by passing
    /// [`Ordering::SeqCst`](../../std/sync/atomic/enum.Ordering.html)
    /// as both the `success` and `failure` parameters. For example,
    /// [`AtomicBool::compare_exchange`][compare_exchange].
    ///
    /// [compare_exchange]: ../../std/sync/atomic/struct.AtomicBool.html#method.compare_exchange
    pub fn atomic_cxchg<T>(dst: *mut T, old: T, src: T) -> (T, bool);
```

FFIっぽいな、と思いましたが、the bookを見るとどうやら違うようです。

[The Book `intrinsics`](https://doc.rust-jp.rs/the-rust-programming-language-ja/1.6/book/intrinsics.html)

> intrinsicsは特別なABI rust-intrinsic を用いて、FFIの関数で有るかのようにインポートされます。

じゃあ実体はどこにあるねん？と思ったら、ソースコードの先頭に書いてありました。

> The corresponding definitions are in librustc_codegen_llvm/intrinsic.rs.

ということで、次はrustcです。

## Dive into the rustc compiler

`src/librustc_codegen_llvm/intrinsic.rs`

兎にも角にも、`cxchg`で検索してみましょう。下のコードが引っかかりました。
どうやら、巨大なパターンマッチの一部のようです。

```rust
            // This requires that atomic intrinsics follow a specific naming pattern:
            // "atomic_<operation>[_<ordering>]", and no ordering means SeqCst
            name if name.starts_with("atomic_") => {
                use rustc_codegen_ssa::common::AtomicOrdering::*;
                use rustc_codegen_ssa::common::
                    {SynchronizationScope, AtomicRmwBinOp};

                let split: Vec<&str> = name.split('_').collect();

                let is_cxchg = split[1] == "cxchg" || split[1] == "cxchgweak";
...
```

ものすごく上に進むと関数の入り口と、パターンマッチの入り口が見つかりました。

```rust
impl IntrinsicCallMethods<'tcx> for Builder<'a, 'll, 'tcx> {
    fn codegen_intrinsic_call(
        &mut self,
        callee_ty: Ty<'tcx>,
        fn_ty: &FnType<'tcx, Ty<'tcx>>,
        args: &[OperandRef<'tcx, &'ll Value>],
        llresult: &'ll Value,
        span: Span,
    ) {
...
        let llval = match name {
```

返り値なしで、`Builder`へのimplなので、self (Builder)にコード生成の何らかの変化を起こす作用がある関数だと予測できます。
`name`は、intrinsicの名前です。例えば、libcore::intrinsicsにあった`"atomic_cxchg"`が該当します。少しわかりやすいものを抜粋します。

```rust
            "unreachable" => { ... },
            "likely" => { ... },
```

と、このような感じです。では肝心の`atomic_cxchg`は、と言うと上のコードにあるアームです。

```rust
            name if name.starts_with("atomic_") => { ... }
```

`start_with`のガードでパターンマッチしています。さらに、`_`でsplitして、後半部分で`cxchg`かどうか、を判定しています。

```rust
            name if name.starts_with("atomic_") => {
                use rustc_codegen_ssa::common::AtomicOrdering::*;
                use rustc_codegen_ssa::common::
                    {SynchronizationScope, AtomicRmwBinOp};

                let split: Vec<&str> = name.split('_').collect();

                let is_cxchg = split[1] == "cxchg" || split[1] == "cxchgweak";
```

細かいところを少し飛ばして、関係が深そうな部分を見ます。パッと見ですが、下の部分で命令を生成しているように見えます。
`self.atomic_cmpxchg()`を追いかけると、self (Builder)に命令を追加していそうな気がします。

```rust
                match split[1] {
                    "cxchg" | "cxchgweak" => {
                        let ty = substs.type_at(0);
                        if int_type_width_signed(ty, self).is_some() {
                            let weak = split[1] == "cxchgweak";
                            let pair = self.atomic_cmpxchg(
                                args[0].immediate(),
                                args[1].immediate(),
                                args[2].immediate(),
                                order,
                                failorder,
                                weak);
                            let val = self.extract_value(pair, 0);
                            let success = self.extract_value(pair, 1);
                            let success = self.zext(success, self.type_bool());

                            let dest = result.project_field(self, 0);
                            self.store(val, dest.llval, dest.align);
                            let dest = result.project_field(self, 1);
                            self.store(success, dest.llval, dest.align);
                            return;
                        } else {
                            return invalid_monomorphization(ty);
                        }
                    }
```

builderの方を見てみましょう。
`src/librustc_codegen_llvm/builder.rs`

```rust
    // Atomic Operations
    fn atomic_cmpxchg(
        &mut self,
        dst: &'ll Value,
        cmp: &'ll Value,
        src: &'ll Value,
        order: rustc_codegen_ssa::common::AtomicOrdering,
        failure_order: rustc_codegen_ssa::common::AtomicOrdering,
        weak: bool,
    ) -> &'ll Value {
        let weak = if weak { llvm::True } else { llvm::False };
        unsafe {
            llvm::LLVMRustBuildAtomicCmpXchg(
                self.llbuilder,
                dst,
                cmp,
                src,
                AtomicOrdering::from_generic(order),
                AtomicOrdering::from_generic(failure_order),
                weak
            )
        }
    }
```

`Value`とは、`llvm::LLVMRustBuildAtomicCmpXchg`は、LLVMのFFIとしてexternされています。
`src/librustc_codegen_llvm/llvm/ffi.rs`

```rust
extern { pub type Value; }

extern "C" {
    pub fn LLVMRustBuildAtomicCmpXchg(B: &Builder<'a>,
                                      LHS: &'a Value,
                                      CMP: &'a Value,
                                      RHS: &'a Value,
                                      Order: AtomicOrdering,
                                      FailureOrder: AtomicOrdering,
                                      Weak: Bool)
                                      -> &'a Value;
}
```

LLVMにつなぐためのラッパーにたどり着きました。
`src/rustllvm/RustWrapper.cpp`

```cpp
extern "C" LLVMValueRef
LLVMRustBuildAtomicCmpXchg(LLVMBuilderRef B, LLVMValueRef Target,
                           LLVMValueRef Old, LLVMValueRef Source,
                           LLVMAtomicOrdering Order,
                           LLVMAtomicOrdering FailureOrder, LLVMBool Weak) {
  AtomicCmpXchgInst *ACXI = unwrap(B)->CreateAtomicCmpXchg(
      unwrap(Target), unwrap(Old), unwrap(Source), fromRust(Order),
      fromRust(FailureOrder));
  ACXI->setWeak(Weak);
  return wrap(ACXI);
}
```

LLVMのIRBuilderに繋がっていることでしょう。多分。
`llvm::IRBuilder.h`

```cpp
   AtomicCmpXchgInst *
   CreateAtomicCmpXchg(Value *Ptr, Value *Cmp, Value *New,
                       AtomicOrdering SuccessOrdering,
                       AtomicOrdering FailureOrdering,
                       SyncScope::ID SSID = SyncScope::System) {
     return Insert(new AtomicCmpXchgInst(Ptr, Cmp, New, SuccessOrdering,
                                         FailureOrdering, SSID));
   }
```

## Dive into the LLVM IR

さて、ここでLLVMにバトンタッチしましょう。まずは、LLVMでcompare and swapがどのように扱われているか、です。
LLVM IRを調べると、`cmpxchg`が該当しそうです。

[`cmpxchg` instruction](https://llvm.org/docs/LangRef.html#cmpxchg-instruction)

> The ‘cmpxchg’ instruction is used to atomically modify memory. It loads a value in memory and compares it to a given value. If they are equal, it tries to store a new value into the memory.

はい、間違いなくこれですね。LLVMを調査する上では、`cmpxchg`をキーワードに辿れば良いことがわかりました。

## Dive into the LLVM CodeGen

LLVMで関連する箇所は、ターゲットアーキテクチャへの機械語命令生成部であると推測できます。
`llvm/lib/Target/RISCV`のソースファイル一覧を眺めていると、下記のいかにもな名前のファイルがあります。
RISC-Vでは、compare and swapを1命令で実現できないため、擬似命令として実装されているはずです。

[RISCVExpandPseudoInsts.cpp](https://github.com/llvm-mirror/llvm/blob/master/lib/Target/RISCV/RISCVExpandPseudoInsts.cpp)

LLVMのことは全くわからないので、良い機会と考えて少し深堀りしてみましょう。ちなみに私はコンパイラも素人ですので、ツッコミをお待ちしております。
コードジェネレーターのドキュメントを見てみます。まずは、high-level designから。

[The high-level design of the code generator](https://llvm.org/docs/CodeGenerator.html#the-high-level-design-of-the-code-generator)

コード生成は次のステップに分割されています。

1. Instruction Selection
2. Scheduling and Formation
3. SSA-based Machine Code Optimizations
4. Register Allocation
5. Prolog/Epilog Code Insertion
6. Late Machine Code Optimization
7. Code Emission

意外とわかりそうな感じです。

### Instruction Selection & Scheduling and Formation

LLVMコードを、ターゲットアーキテクチャの命令に変換します。SelectionDAGという方法をベースにしているようです。
ブロック内の命令をDAGで表現して、最適化する、とあります。最適化の部分はたくさんやっているので省略します。

このフェーズでは、最終的に`MachineInstr`のリストに変換されるようです。
[MachineInstr](https://llvm.org/docs/CodeGenerator.html#machineinstr)は、ターゲットマシンの命令を表現するクラスです。命令のセマンティクスの情報は持っていないため、`TargetInstrInfo`を別途参照します。
opcodeとoperandsを保有しているようなので、1命令1インスタンスになる、という理解で良いのでしょうか？
命令の情報は、target description (*.td)ファイルに記載します。

機械語命令は、`MachineInstrBuilder`で生成します。

```cpp
// Create a 'DestReg = mov 42' (rendered in X86 assembly as 'mov DestReg, 42')
// instruction and insert it at the end of the given MachineBasicBlock.
const TargetInstrInfo &TII = ...
MachineBasicBlock &MBB = ...
DebugLoc DL;
MachineInstr *MI = BuildMI(MBB, DL, TII.get(X86::MOV32ri), DestReg).addImm(42);
```

これで、`mov DestReg, 42`になるということですね。なるほど。

### SSA-based Machine Code Optimizations

詳細が書かれていないので、あまりわかりませんが、peephole optimizationなどを行うようです。

### Register Allocation

ここまでのコードは、無限個の仮想レジスタを対象としており、実際のターゲットアーキテクチャに即したレジスタ割り当てを行います。

### Prolog/Epilog Code Insertion

各関数の入り口、出口のコードを挿入します。stack pointerの操作や、registerの退避などを自動生成する、ということでしょう。

### Late Machine Code Optimization

もう1回コードに最適化をかけます。

### Code Emission

最終的なアセンブラを出力します。

ということで、今回一番関連がありそうなのは、`MachineInstr`への変換あたりと考えられます。
改めて、[RISCVExpandPseudoInsts.cpp](https://github.com/llvm-mirror/llvm/blob/master/lib/Target/RISCV/RISCVExpandPseudoInsts.cpp)を見ていきましょう。
まず、こいつが何をしそうか、をincludeしているファイルから推測します。

```cpp
#include "RISCV.h"
#include "RISCVInstrInfo.h"
#include "RISCVTargetMachine.h"

#include "llvm/CodeGen/LivePhysRegs.h"
#include "llvm/CodeGen/MachineFunctionPass.h"
#include "llvm/CodeGen/MachineInstrBuilder.h"
```

`RISCVInstrInfo`は、命令のセマンティクス情報であることが、ドキュメントから判明しています。
`RISCVTargetMachine`は、[The TargetMachine class](https://llvm.org/docs/CodeGenerator.html#the-targetmachine-class)によると、下のように書いてあり、`RISCVInstrInfo`を使うときに参照されるみたいです。

> The TargetMachine class provides virtual methods that are used to access the target-specific implementations of the various target description classes via the get*Info methods (getInstrInfo, getRegisterInfo, getFrameInfo, etc.).

必須の項目は、`DataLayout`で、Rustでもtarget記述でよく見る`e-m:e-p:32:32-i64:64-n32-S128`←こういう奴です。
それ以外には、target-specific passを設定しています。ターゲットアーキテクチャ特有の処理をしたい場合に、任意の処理を挿入できるようです。

`RISCVPassConfig`では、`addPreEmitPass2()`をoverrideしており、その中で、`createRISCVExpandPseudoPass()`をPassに追加しています。これできっとコード生成のフローの中で、`RISCVExpandPseudoInsts`の処理が呼び出されるのでしょう。

RISCVTargetMachine.cpp

```cpp
class RISCVPassConfig : public TargetPassConfig {
public:
  RISCVPassConfig(RISCVTargetMachine &TM, PassManagerBase &PM)
      : TargetPassConfig(TM, PM) {}

  RISCVTargetMachine &getRISCVTargetMachine() const {
    return getTM<RISCVTargetMachine>();
  }
...
  void addPreEmitPass2() override;
...
};

void RISCVPassConfig::addPreEmitPass2() {
  // Schedule the expansion of AMOs at the last possible moment, avoiding the
  // possibility for other passes to break the requirements for forward
  // progress in the LR/SC block.
  addPass(createRISCVExpandPseudoPass());
}
```

`Pass`とはなんでしょうか？[Writing an LLVM Pass Introduction — What is a pass?](http://llvm.org/docs/WritingAnLLVMPass.html#introduction-what-is-a-pass)を参照すると、複数の`Pass`が変換と最適化を行うことでコンパイラを作り上げる、とあります。
`FunctionPass`は関数のように振る舞う`Pass`のことを意味するようです。

```cpp
class RISCVExpandPseudo : public MachineFunctionPass {
public:
  const RISCVInstrInfo *TII;
...
  bool runOnMachineFunction(MachineFunction &MF) override;
...

private:
  bool expandMBB(MachineBasicBlock &MBB);
  bool expandMI(MachineBasicBlock &MBB, MachineBasicBlock::iterator MBBI,
                MachineBasicBlock::iterator &NextMBBI);
...
  bool expandAtomicCmpXchg(MachineBasicBlock &MBB,
                           MachineBasicBlock::iterator MBBI, bool IsMasked,
                           int Width, MachineBasicBlock::iterator &NextMBBI);
};
```

`RISCVExpandPseudo`は`MachineFunctionPass`を継承しています。`MachineFunctionPass`は`FunctionPass`の実装であり、ターゲットマシン固有のコードジェネレータとして利用するようです。
RISC-Vでは、compare and swapをlr/scを使った擬似命令で実現するため、ターゲットマシン固有の`MachineFunctionPass`として実装するのは、納得です。
overrideしている重要そうな仮想関数は、`runOnMachineFunction`です。この中から、private関数のexpand*を呼び出すようになっています。

```cpp
bool RISCVExpandPseudo::runOnMachineFunction(MachineFunction &MF) {
  TII = static_cast<const RISCVInstrInfo *>(MF.getSubtarget().getInstrInfo());
  bool Modified = false;
  for (auto &MBB : MF)
    Modified |= expandMBB(MBB);
  return Modified;
}
```

`MachineFunction`は、`MachineBasicBlock`のイテレータを実装しているようです。全てのBasicBlockに対して、`expandMBB`を呼び出しています。

```cpp
bool RISCVExpandPseudo::expandMBB(MachineBasicBlock &MBB) {
  bool Modified = false;

  MachineBasicBlock::iterator MBBI = MBB.begin(), E = MBB.end();
  while (MBBI != E) {
    MachineBasicBlock::iterator NMBBI = std::next(MBBI);
    Modified |= expandMI(MBB, MBBI, NMBBI);
    MBBI = NMBBI;
  }

  return Modified;
}
```

`MachineBasicBlock`にもイテレータがあるようです？`MachineBasicBlock`自身と、イテレータの現在位置と、イテレータの次の位置をパラメータに、`expandMI`を呼び出します。

```cpp
bool RISCVExpandPseudo::expandMI(MachineBasicBlock &MBB,
                                 MachineBasicBlock::iterator MBBI,
                                 MachineBasicBlock::iterator &NextMBBI) {
  switch (MBBI->getOpcode()) {
  case RISCV::PseudoAtomicLoadNand32:
    return expandAtomicBinOp(MBB, MBBI, AtomicRMWInst::Nand, false, 32,
                             NextMBBI);
...
  case RISCV::PseudoCmpXchg32:
    return expandAtomicCmpXchg(MBB, MBBI, false, 32, NextMBBI);
  case RISCV::PseudoMaskedCmpXchg32:
    return expandAtomicCmpXchg(MBB, MBBI, true, 32, NextMBBI);
  }

  return false;
}
```

使い方から予測するに、MachineBasicBlockは、複数の命令からできていて、イテレータでは命令をイテレートしているみたいです。
そして、ある命令のopcodeが、`PseudoCmpXchg32`であれば、`expandAtomicCmpXchg`を呼び出します。
`expandAtomicCmpXchg`は長いのですが、一度全部を貼ります。

```cpp
bool RISCVExpandPseudo::expandAtomicCmpXchg(
    MachineBasicBlock &MBB, MachineBasicBlock::iterator MBBI, bool IsMasked,
    int Width, MachineBasicBlock::iterator &NextMBBI) {
  assert(Width == 32 && "RV64 atomic expansion currently unsupported");
  MachineInstr &MI = *MBBI;
  DebugLoc DL = MI.getDebugLoc();
  MachineFunction *MF = MBB.getParent();
  auto LoopHeadMBB = MF->CreateMachineBasicBlock(MBB.getBasicBlock());
  auto LoopTailMBB = MF->CreateMachineBasicBlock(MBB.getBasicBlock());
  auto DoneMBB = MF->CreateMachineBasicBlock(MBB.getBasicBlock());

  // Insert new MBBs.
  MF->insert(++MBB.getIterator(), LoopHeadMBB);
  MF->insert(++LoopHeadMBB->getIterator(), LoopTailMBB);
  MF->insert(++LoopTailMBB->getIterator(), DoneMBB);

  // Set up successors and transfer remaining instructions to DoneMBB.
  LoopHeadMBB->addSuccessor(LoopTailMBB);
  LoopHeadMBB->addSuccessor(DoneMBB);
  LoopTailMBB->addSuccessor(DoneMBB);
  LoopTailMBB->addSuccessor(LoopHeadMBB);
  DoneMBB->splice(DoneMBB->end(), &MBB, MI, MBB.end());
  DoneMBB->transferSuccessors(&MBB);
  MBB.addSuccessor(LoopHeadMBB);

  unsigned DestReg = MI.getOperand(0).getReg();
  unsigned ScratchReg = MI.getOperand(1).getReg();
  unsigned AddrReg = MI.getOperand(2).getReg();
  unsigned CmpValReg = MI.getOperand(3).getReg();
  unsigned NewValReg = MI.getOperand(4).getReg();
  AtomicOrdering Ordering =
      static_cast<AtomicOrdering>(MI.getOperand(IsMasked ? 6 : 5).getImm());

  if (!IsMasked) {
    // .loophead:
    //   lr.w dest, (addr)
    //   bne dest, cmpval, done
    BuildMI(LoopHeadMBB, DL, TII->get(getLRForRMW32(Ordering)), DestReg)
        .addReg(AddrReg);
    BuildMI(LoopHeadMBB, DL, TII->get(RISCV::BNE))
        .addReg(DestReg)
        .addReg(CmpValReg)
        .addMBB(DoneMBB);
    // .looptail:
    //   sc.w scratch, newval, (addr)
    //   bnez scratch, loophead
    BuildMI(LoopTailMBB, DL, TII->get(getSCForRMW32(Ordering)), ScratchReg)
        .addReg(AddrReg)
        .addReg(NewValReg);
    BuildMI(LoopTailMBB, DL, TII->get(RISCV::BNE))
        .addReg(ScratchReg)
        .addReg(RISCV::X0)
        .addMBB(LoopHeadMBB);
  } else {
    // .loophead:
    //   lr.w dest, (addr)
    //   and scratch, dest, mask
    //   bne scratch, cmpval, done
    unsigned MaskReg = MI.getOperand(5).getReg();
    BuildMI(LoopHeadMBB, DL, TII->get(getLRForRMW32(Ordering)), DestReg)
        .addReg(AddrReg);
    BuildMI(LoopHeadMBB, DL, TII->get(RISCV::AND), ScratchReg)
        .addReg(DestReg)
        .addReg(MaskReg);
    BuildMI(LoopHeadMBB, DL, TII->get(RISCV::BNE))
        .addReg(ScratchReg)
        .addReg(CmpValReg)
        .addMBB(DoneMBB);

    // .looptail:
    //   xor scratch, dest, newval
    //   and scratch, scratch, mask
    //   xor scratch, dest, scratch
    //   sc.w scratch, scratch, (adrr)
    //   bnez scratch, loophead
    insertMaskedMerge(TII, DL, LoopTailMBB, ScratchReg, DestReg, NewValReg,
                      MaskReg, ScratchReg);
    BuildMI(LoopTailMBB, DL, TII->get(getSCForRMW32(Ordering)), ScratchReg)
        .addReg(AddrReg)
        .addReg(ScratchReg);
    BuildMI(LoopTailMBB, DL, TII->get(RISCV::BNE))
        .addReg(ScratchReg)
        .addReg(RISCV::X0)
        .addMBB(LoopHeadMBB);
  }

  NextMBBI = MBB.end();
  MI.eraseFromParent();

  LivePhysRegs LiveRegs;
  computeAndAddLiveIns(LiveRegs, *LoopHeadMBB);
  computeAndAddLiveIns(LiveRegs, *LoopTailMBB);
  computeAndAddLiveIns(LiveRegs, *DoneMBB);

  return true;
}
```

## LLVMの状況

どうやら、下記のパッチでcompare and swapが実装されたようです。
2018/11/29に(svn的な意味で)コミットされています。

[`[RISCV]` Implement codegen for cmpxchg on RV32IA](https://reviews.llvm.org/D48131)

[LLVM Atomics and Codegen](http://llvm.org/docs/Atomics.html#atomics-and-codegen)

> cmpxchg -> loop with load-linked/store-conditional by overriding shouldExpandAtomicCmpXchgInIR(), emitLoadLinked(), emitStoreConditional()

```cpp
TargetLowering::AtomicExpansionKind
RISCVTargetLowering::shouldExpandAtomicCmpXchgInIR(
    AtomicCmpXchgInst *CI) const {
  unsigned Size = CI->getCompareOperand()->getType()->getPrimitiveSizeInBits();
  if (Size == 8 || Size == 16)
    return AtomicExpansionKind::MaskedIntrinsic;
  return AtomicExpansionKind::None;
}

Value *RISCVTargetLowering::emitMaskedAtomicCmpXchgIntrinsic(
    IRBuilder<> &Builder, AtomicCmpXchgInst *CI, Value *AlignedAddr,
    Value *CmpVal, Value *NewVal, Value *Mask, AtomicOrdering Ord) const {
  Value *Ordering = Builder.getInt32(static_cast<uint32_t>(Ord));
  Type *Tys[] = {AlignedAddr->getType()};
  Function *MaskedCmpXchg = Intrinsic::getDeclaration(
      CI->getModule(), Intrinsic::riscv_masked_cmpxchg_i32, Tys);
  return Builder.CreateCall(MaskedCmpXchg,
                            {AlignedAddr, CmpVal, NewVal, Mask, Ordering});
}
```

## 参考

[rust-lang/rust](https://github.com/rust-lang/rust)
[rust-doc core::sync::atomic](https://doc.rust-lang.org/core/sync/atomic/)
[Rust nomicon Atomics](https://doc.rust-lang.org/nomicon/atomics.html)