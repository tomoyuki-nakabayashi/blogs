# core::sync::atomic

## compare and swap

x86にはcompare and wapが〜

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

## 参考

[rust-lang/rust](https://github.com/rust-lang/rust)
[rust-doc core::sync::atomic](https://doc.rust-lang.org/core/sync/atomic/)
[Rust nomicon Atomics](https://doc.rust-lang.org/nomicon/atomics.html)