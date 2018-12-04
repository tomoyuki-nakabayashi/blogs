# core::sync::atomic

## compare and swap

x86にはcompare and swapが〜

## RISC-V Atomic Extension

RISC-Vには、compare and swap命令が存在していません。

### lr/sc

load reserved/store conditionalという命令がatomicなロードストア命令として定義されています。

## AtomicBoolソースコード解析

`AtomicBool`の実装を解析していきましょう。

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

intrinsics、これは、llvmの[intrinsic](https://llvm.org/docs/ExtendingLLVM.html#intrinsic-function)に関係あるものでしょうか？
atomic.rsでは、`use intrinsics`としています。こいつは何者でしょうか？

```rust
use intrinsics;
```

[Rust doc std::intrinsics](https://doc.rust-lang.org/std/intrinsics/index.html)

> The corresponding definitions are in librustc_codegen_llvm/intrinsic.rs.

お、LLVM codegenへの言及がなされています。
これは、本格的にLLVMが原因で、RISC-Vでspinが使えない予感がします。

[LLVM Atomic Instructions and Concurrency Guide](http://llvm.org/docs/Atomics.html)

[Atomics and Codegen](http://llvm.org/docs/Atomics.html#atomics-and-codegen)

## LLVM CodeGen

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

## LLVMの状況

どうやら、下記のパッチでcompare and swapが実装されたようです。
2018/11/29にコミットされています。

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