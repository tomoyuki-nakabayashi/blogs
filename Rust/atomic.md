# Rust Atomic compare and swap 2018 editionのRISC-Vソース〜LLVMを添えて〜

## はじめに

本記事は、[Rust Advent Calendar 2018](https://qiita.com/advent-calendar/2018/rust)の22日目として書かれました。
タイトルは少しふざけていますが、内容は真剣です。ぜひ楽しんでいって下さい。

### 事の発端あるいは茶番

2018年11月初旬

僕「[RustでRISC-VをターゲットにOS自作](https://qiita.com/tomoyuki-nakabayashi/items/76f912adb6b7da6030c7)するの楽すぎ！至高！」
僕「UARTデバイスを複数threadから安全にアクセスできるようにするぞ！」
僕「これ、[Writing an OS in Rust](https://os.phil-opp.com/)でやったところだ！`lazy_static`とか`spin` crate使えば良いんでしょ？」

```toml
$ tail Cargo.toml
...
[dependencies]
spin = "0.4.10"
```

僕「よし、Cargo.tomlも書いたし、簡単な実装もした。いざ！」

```
$ cargo build
   Compiling spin v0.4.10
error[E0599]: no method named `compare_and_swap` found for type `core::sync::atomic::AtomicBool` in the current scope
   -->
    |
157 |         while self.lock.compare_and_swap(false, true, Ordering::Acquire) != false
    |                         ^^^^^^^^^^^^^^^^

error[E0599]: no method named `compare_and_swap` found for type `core::sync::atomic::AtomicBool` in the current scope
```

僕「めっちゃ辛そうなエラー出るやん・・・」
僕「なんか納得いかんから、調べたろ」

### この記事 is 何？

Rust core libraryの`core::sync::atomic::Atomic*.compare_and_swap`関数を解析した経過と結果を記したものです。それ以外のことは出てきません。
途中からLLVMに突入しますので、C++成分がそれなりに混じっています。RISC-Vのアセンブリソースコードも少々あります。RISC-Vのアセンブリは難しくないので大丈夫です。

今回、この記事に関わる調査を行ったおかげで、次の知見を得ることができました。

- Rust target tripleの設定
- rustc compilerとLLVMとの関係性を少し
- LLVMのコード生成を少し

一番大きかった収穫は、私は言語処理系素人ですが、意外と読める、ということです。これは、コード自体の品質が高いこともありますが、コメントやテストコードが充実しているためです。自分自身でもこのようなリテラシーの高いコードを書いて行こう、という気持ちになれて、非常に良い刺激を得ることができました。
やりたいことができない、がスタート地点でしたが、結果として前よりもっとRustが好きになりました。

つまり、この記事の趣旨は、**広めよう！Rust言語処理系リーディングの輪！** ということです。

以降、本記事は次の流れで展開します。

1. プロセッサのAtomicなデータ更新 (compare and swapとは？)
2. Rust core library解析
3. rustc compiler解析
4. LLVMコード生成部解析

## プロセッサのAtomicなデータ更新

排他制御の一種です。近代のプロセッサでは、マルチスレッドにおいて、Atomicなメモリ操作が行えるロードストア命令が用意されています (確かMIPS R3000とかにはなかったと思います)。
[ARMプロセッサにおけるロックフリーなデータ更新](https://qiita.com/RKX1209/items/7ba1e7d439cf28c92041)、に詳しく書かれています。以降のコード例で引用させて頂いています。ありがとうございます。

排他制御の必要性について、簡単に説明します。よく銀行取引の例をみかける、あれです。
プロセッサレベルでも同様で、スレッドAとスレッドBが、共有メモリにアクセスする場合に、スレッドAがデータ更新している途中で、スレッドBがデータを更新してしまうと、データの整合性がなくなってしまいます。
シングルコアプロセッサでは、単に割り込みを禁止すれば良いのですが、マルチコアプロセッサにおいては、他のプロセッサが共有データを変更していないこと、を保証する必要があります。
そこで、近代の命令セットアーキテクチャではAtomicなメモリ操作命令、というものを持っています。Atomicとは、その操作を行っている間、別の操作に割り込まれないことを意味します。

典型的な排他制御では、spin lockやMutexのように、共有データにロックをかけることで、排他制御します。ロックがかかっている間は、他スレッドの共有データへのアクセスを禁止します。
ロックがかかっている間、他スレッドは待ち状態になってしまうため、パフォーマンス低下に繋がる恐れがあります。

そこで、プロセッサではロックを使用せずに、Atomicな共有データアクセスをするための命令を備えています。
この命令には、2つの方法があります。

- compare and swap (cas)
- load link/store condition (ll/sc)

RISC-Vでは、ll/scではなく、load reserved/store conditional (lr/sc)というニーモニックになっています。

### compare and swap

x86では`cmpxchg`という名前の命令です。x86のアセンブリを説明するのは大変心が折れるので、C言語の擬似コードで見ていきます([ARMプロセッサにおけるロックフリーなデータ更新](https://qiita.com/RKX1209/items/7ba1e7d439cf28c92041)からの引用です)。
あるメモリアドレス`ptr`の現在値が、古い値`old`と等しければ、新しい値`old`を`ptr`に書き込みます(古い値も新しい値もメモリ上にある)。新しい値を書き込んだ場合、`1`を返します。

```c
int CAS(void *ptr, void *old, void *new) {
  if (memcmp(ptr, old) == 0) {
    memcpy(ptr, new); // Set new value to ptr
    return 1;
  }
  memcpy(old, ptr);
  return 0;
}
```

使い方を見ると理解しやすいと思うので、[ARMプロセッサにおけるロックフリーなデータ更新](https://qiita.com/RKX1209/items/7ba1e7d439cf28c92041)の例をさらに引用します。

```c
void AtomicOp(int *ptr)
{
    while(1)
    {
        const int OldValue = *ptr;  // ★1
        const int NewValue = UpdateData(OldValue); //Update old value to new value

        // If the result is the original value, the new value was stored.
        if(CAS(ptr, &OldValue, &NewValue))  // ★2
        {
            return;
        }
    }
}
```

上記コードを説明します。あるアドレス`ptr`から、古い値`OldValue`を読み込み、新しい値`NewValue`を計算します。
compare and swap命令では、`ptr`に格納されている値が`OldValue`と等しいときに、`NewValue`を`ptr`に書き込みます。`NewValue`を書き込んだ時、結果は`1`に、それ以外のときは`0`になります。
compare and swap命令実行時、`ptr`に格納されている値が`OldValue`と等しい、ということは、`★1`から`★2`の間にデータが変更されていない、ということになります。これは、他のスレッドから値が書き換えられていないことを意味します。実際は擬陽性の問題があり、`OldValue⇨AnotherValue⇨OldValue`という変更があった場合に、検出できません。
もし、データが書き換えられていた場合は、データ更新は行われず、もう1度`OldValue`を読み込むところからやり直します。

### ll/scまたはlr/sc

ARMやRISC-Vがサポートしている命令です。こちらの命令では、ll (load link)/sc (store conditional)というロードストア命令をペアで使います。
ll命令で、 **「このアドレスのデータ、予約しといて」** ということで、reservedフラグをつけておきます。sc命令で値を更新する際、 **「予約しといたアドレスのデータ、まだ予約されてる？されてるんやったら更新しといて！」** ということをやります。データが別スレッドにより更新されている場合は、 **「何ィ？予約しといて言うのに、他の人が触ったぁ！？じゃあ、もっかい予約するから、今度こそ取っといてな。頼むでほんま。」** となります。

RISC-Vのlr/scについては、[RISC-V specifications](https://riscv.org/specifications/)の`7. "A" Standard Extension for Atomic Instructions, Version 2.0`に記載があります。

RISC-V命令セットで、compare and swapではなくlr/scを採用する理由は、擬陽性(ABA)問題と、必要となるオペランド数によるところが大きいです。
compare and swapでは、ソースオペランドが3つ必要になります。それに対して、lr/scでは、2オースオペランドで済みます。
これは、データパスを単純に保つ上で重要であるため、RISCプロセッサの選択として、納得のいくものです。

> Both compare-and-swap (CAS) and LR/SC can be used to build lock-free data structures. After
> extensive discussion, we opted for LR/SC for several reasons: 1) CAS suffers from the ABA
> problem, which LR/SC avoids because it monitors all accesses to the address rather than only
> checking for changes in the data value; 2) CAS would also require a new integer instruction format
> to support three source operands (address, compare value, swap value) as well as a different
> memory system message format, which would complicate microarchitectures;

他2つの理由も併記されているので、興味がある方は、specificationをご参照下さい。

lr/scを使って、compare and swap機能を実現する場合、次のようなアセンブラになります。

```asm
# a0 holds address of memory location                  -> ptr
# a1 holds expected value                              -> old
# a2 holds desired value                               -> new
# a0 holds return value, 0 if successful, !0 otherwise -> return
cas:
    lr.w t0, (a0)     # Load original value.
    bne t0, a1, fail  # Doesn’t match, so fail.
    sc.w a0, a2, (a0) # Try to update.
    bnez a0, cas      # Retry if store-conditional failed.
    jr ra # Return.
fail:
    li a0, 1          # Set return to failure.
    jr ra # Return.
```

このように、RISC-Vに備わっているlr/sc命令を組み合わせることでcompare and swapが実現できるため、spin crateをコンパイルしようとしたときに、`Atomic*`型に`compare_and_swap`関数が実装されていない、というのは不自然であると考えました。
そこで、実装がどうなっているのか気になったため、調査を行うこととしました。

## Dive into the Rust Error Message

改めて、エラーコードを再掲載します。やはり何事も基本はエラーメッセージを解析するところからです。

```
$ cargo build
   Compiling spin v0.4.10
error[E0599]: no method named `compare_and_swap` found for type `core::sync::atomic::AtomicBool` in the current scope
   -->
    |
157 |         while self.lock.compare_and_swap(false, true, Ordering::Acquire) != false
    |                         ^^^^^^^^^^^^^^^^

error[E0599]: no method named `compare_and_swap` found for type `core::sync::atomic::AtomicBool` in the current scope
```

まず、エラーメッセージはそのまま、compare_and_swap関数がない、というものです。
一応、`E0599`というError Indexも確認しておきましょう。
[Rust Compiler Error Index](https://doc.rust-lang.org/error-index.html)には、サンプルコードを含めた、Rust compilerのエラー解説があります。

> This error occurs when a method is used on a type which doesn't implement it:
> Erroneous code example:
> ```rust
> struct Mouth;
>
> let x = Mouth;
> x.chocolate(); // error: no method named `chocolate` found for type `Mouth`
>               //        in the current scope
> ```

残念ながら、今回は役に立ちそうもありません。

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
この`target_has_atomic`がどこでどのように定義されているか、を追いかけます。
現在のRustでは、2つのRISC-Vアーキテクチャがサポートされています。

[riscv32imc_unknown_none_elf.rs](https://github.com/rust-lang/rust/blob/master/src/librustc_target/spec/riscv32imc_unknown_none_elf.rs)
[riscv32imac_unknown_none_elf.rs](https://github.com/rust-lang/rust/blob/master/src/librustc_target/spec/riscv32imac_unknown_none_elf.rs)

余談ですが、RISC-Vでは、命令セットを任意の組み合わせで拡張できるようになっています。
`rv32`は、32 bitアーキテクチャを意味しており、その後に、`i, m, a, c, f, d`といった、どのような命令をサポートしているか、の情報が続きます。
i -> 整数、m -> 整数乗除算、a -> Atomic、c -> 16 bit圧縮命令、f -> 単精度浮動小数点、d -> 倍精度浮動小数点、といった感じです。

Rustでサポートしている1つ目は、Atomic命令をサポートしない`rv32imc`です。こちらをターゲットにする場合は、compare and swapがなくても納得です。
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

compare_and_swap関数では、`compare_exchange`に処理を移譲して、その結果がOk/Err、どちらでもその中身を返しています。

```rust
    // 再掲載
    #[cfg(target_has_atomic = "cas")]
    pub fn compare_and_swap(&self, current: bool, new: bool, order: Ordering) -> bool {
        match self.compare_exchange(current, new, order, strongest_failure_ordering(order)) {
            Ok(x) => x,
            Err(x) => x,
        }
    }
```

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

`Acquire`や`Release`を説明し始めると長くなってしまうので、省略させて下さい。
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

externされているだけなので、FFIっぽいな、と思いましたが、the bookを見るとどうやら違うようです。

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
                            let pair = self.atomic_cmpxchg(  // ここがあやしい
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

`atomic_cmpxchg`関数は、Builderの関数なので、Builderの実装を見てみましょう。とうとうLLVM FFIへの入り口に到達します。
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

`Value`と、`llvm::LLVMRustBuildAtomicCmpXchg`は、LLVMのFFIとしてexternされています。
`src/librustc_codegen_llvm/llvm/ffi.rs`
[LLVM Value](http://llvm.org/doxygen/classllvm_1_1Value.html#details)を見ると、InstructionやFunctionの基底クラスである、となっています。

```rust
// Opaque pointer types
...
extern { pub type Value; }
...
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

LLVM側は、`AtomicCmpXchgInst`のポインタを返してきます。`AtomicCmpXchgInst`は、`Value`の派生クラスです。これでRust側でcompare and swap命令のオブジェクトを得られたことになります。
深い…。

## Dive into the LLVM IR

さて、Rust側は大方見終わったので、LLVMにバトンタッチしましょう。まずは、LLVMでcompare and swapがどのように扱われているか、です。
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

うーん、よくわからん！ということで、期待値を予測してみましょう。きっとどこかにテストコードがあるはずなので、探します。
`test/CodeGen/RISCV/atomic-cmpxchg.ll`それっぽいものが見つかりました。

```
; RUN: llc -mtriple=riscv32 -mattr=+a -verify-machineinstrs < %s \
; RUN:   | FileCheck -check-prefix=RV32IA %s
...

; RV32IA-LABEL: cmpxchg_i8_monotonic_monotonic:
; RV32IA:       # %bb.0:
; RV32IA-NEXT:    slli a3, a0, 3
; RV32IA-NEXT:    andi a3, a3, 24
; RV32IA-NEXT:    addi a4, zero, 255
; RV32IA-NEXT:    sll a4, a4, a3
; RV32IA-NEXT:    andi a2, a2, 255
; RV32IA-NEXT:    sll a2, a2, a3
; RV32IA-NEXT:    andi a1, a1, 255
; RV32IA-NEXT:    sll a1, a1, a3
; RV32IA-NEXT:    andi a0, a0, -4
; RV32IA-NEXT:  .LBB0_1: # =>This Inner Loop Header: Depth=1
; RV32IA-NEXT:    lr.w a3, (a0)
; RV32IA-NEXT:    and a5, a3, a4
; RV32IA-NEXT:    bne a5, a1, .LBB0_3
; RV32IA-NEXT:  # %bb.2: # in Loop: Header=BB0_1 Depth=1
; RV32IA-NEXT:    xor a5, a3, a2
; RV32IA-NEXT:    and a5, a5, a4
; RV32IA-NEXT:    xor a5, a3, a5
; RV32IA-NEXT:    sc.w a5, a5, (a0)
; RV32IA-NEXT:    bnez a5, .LBB0_1
; RV32IA-NEXT:  .LBB0_3:
; RV32IA-NEXT:    ret
```

なんか長いですね。compare and swapはlr/scを使えば4命令で実現できるはずです。
アセンブリを眺めてみると、何やらshiftとandを使ってデータの前処理をしているように見えます。
というところでピーンと来ました。lr/scは、32bit単位の命令しかないため、byteやhalf wordをうまく処理できるような前処理が必要なはずです。
32bitの方を見ると、すっきりしています。

```
; RV32IA-LABEL: cmpxchg_i32_monotonic_monotonic:
; RV32IA:       # %bb.0:
; RV32IA-NEXT:  .LBB20_1: # =>This Inner Loop Header: Depth=1
; RV32IA-NEXT:    lr.w a3, (a0)
; RV32IA-NEXT:    bne a3, a1, .LBB20_3
; RV32IA-NEXT:  # %bb.2: # in Loop: Header=BB20_1 Depth=1
; RV32IA-NEXT:    sc.w a4, a2, (a0)
; RV32IA-NEXT:    bnez a4, .LBB20_1
; RV32IA-NEXT:  .LBB20_3:
; RV32IA-NEXT:    ret
```

compare and swapは、次のように定義されています。

```cpp
/// Compare and exchange

class PseudoCmpXchg
    : Pseudo<(outs GPR:$res, GPR:$scratch),
             (ins GPR:$addr, GPR:$cmpval, GPR:$newval, i32imm:$ordering), []> {
  let Constraints = "@earlyclobber $res,@earlyclobber $scratch";
  let mayLoad = 1;
  let mayStore = 1;
  let hasSideEffects = 0;
}

// Ordering constants must be kept in sync with the AtomicOrdering enum in
// AtomicOrdering.h.
multiclass PseudoCmpXchgPat<string Op, Pseudo CmpXchgInst> {
  def : Pat<(!cast<PatFrag>(Op#"_monotonic") GPR:$addr, GPR:$cmp, GPR:$new),
            (CmpXchgInst GPR:$addr, GPR:$cmp, GPR:$new, 2)>;
  def : Pat<(!cast<PatFrag>(Op#"_acquire") GPR:$addr, GPR:$cmp, GPR:$new),
            (CmpXchgInst GPR:$addr, GPR:$cmp, GPR:$new, 4)>;
  def : Pat<(!cast<PatFrag>(Op#"_release") GPR:$addr, GPR:$cmp, GPR:$new),
            (CmpXchgInst GPR:$addr, GPR:$cmp, GPR:$new, 5)>;
  def : Pat<(!cast<PatFrag>(Op#"_acq_rel") GPR:$addr, GPR:$cmp, GPR:$new),
            (CmpXchgInst GPR:$addr, GPR:$cmp, GPR:$new, 6)>;
  def : Pat<(!cast<PatFrag>(Op#"_seq_cst") GPR:$addr, GPR:$cmp, GPR:$new),
            (CmpXchgInst GPR:$addr, GPR:$cmp, GPR:$new, 7)>;
}

def PseudoCmpXchg32 : PseudoCmpXchg;
defm : PseudoCmpXchgPat<"atomic_cmp_swap_32", PseudoCmpXchg32>;

def PseudoMaskedCmpXchg32
    : Pseudo<(outs GPR:$res, GPR:$scratch),
             (ins GPR:$addr, GPR:$cmpval, GPR:$newval, GPR:$mask,
              i32imm:$ordering), []> {
  let Constraints = "@earlyclobber $res,@earlyclobber $scratch";
  let mayLoad = 1;
  let mayStore = 1;
  let hasSideEffects = 0;
}

def : Pat<(int_riscv_masked_cmpxchg_i32
            GPR:$addr, GPR:$cmpval, GPR:$newval, GPR:$mask, imm:$ordering),
          (PseudoMaskedCmpXchg32
            GPR:$addr, GPR:$cmpval, GPR:$newval, GPR:$mask, imm:$ordering)>;
```

さて、ここまで見てくださった読者の皆様、何かおかしいと思いませんか？
そう、**RISC-Vターゲットにcompare and swap命令、実装されているじゃん！** ということです。

実はこのLLVMのコードは、2018/12月初旬に、LLVMのmaster branchから持ってきています。

## LLVM RISC-Vのcompare and swap対応状況

下記のパッチでcompare and swapが実装されたようです。
2018/11/29に(svn的な意味で)コミットされています。

[`[RISCV]` Implement codegen for cmpxchg on RV32IA](https://reviews.llvm.org/D48131)

今後、このコミットが反映されたLLVMがRustで採用されれば、RISC-Vがターゲットでもspinやlazy_staticが使えそうです！
それまでは、間に合せの方法で凌いでも良い気がしますね。

## まとめ

RISC-Vをターゲットに至高のプログラミングを行っていたところ、core::sync:atomic::Atomic*.compare_and_swap関数が実装されていない、というコンパイルエラーが発生し、使いたいspin crateが使えませんでした。
RISC-V命令セットでは、compare and swap命令を直接サポートしていませんが、lr/sc命令を組み合わせることで、compare and swap命令の機能を実現することができます。

調査を実施したところ、Rustのtarget tripleでRISC-V向けには、compare and swapをdisableする設定がされており、それに伴い、compare_and_swap関数が実装されないようになっていました。
さらにLLVMの実装まで読み進めたところ、RISC-VをターゲットとするLLVMのコード生成部に、lr/sc命令を使ってcompare and swap命令を生成する処理が見つかりました。この処理は、2018/11/29に、パッチがコミット(svn的な意味で)されていることが判明しました。
しばらく待てば、RISC-Vがターゲットでも、spin crateが使えそうであることがわかりました。

最も重要なこととして、Rustの言語処理系を解析することが楽しいことが判明しました。
**広げよう、Rust言語処理系リーディングの輪！**

## 参考

[rust-lang/rust](https://github.com/rust-lang/rust)
[rust-doc core::sync::atomic](https://doc.rust-lang.org/core/sync/atomic/)
[Rust nomicon Atomics](https://doc.rust-lang.org/nomicon/atomics.html)