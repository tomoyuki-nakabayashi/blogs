# 組込みRust考察〜効率良く安全な組込み開発をしたい〜

続きです。

## 言語仕様、エコシステム、コミュニティによる生産性向上

生産性は可視化が難しいです。
そのため、本トピックでもわかりやすく数値を比較するようなことは、できません。

Rustは厳格なルールをプログラマに課しますが、Rustの厳格なルールはバグを減らすことで、生産性の向上に寄与するでしょう。
Rustは、とりあえず動くものを作る時には、堅苦しい言語かもしれません。しかし、正しく動くものを作る時は、絶大な力を発揮します。
`unsafe`ブロックを含まないRustの関数は、コンパイルできた時点でスレッドセーフです。

C/C++では、実行時の挙動からしか発見できないバグの多くが、Rustではコンパイラによって暴かれます。
Rustにおいて、コンパイルエラーと格闘することは、Rustのシンタックスを守るというより、正しい設計を考えることなのです。

また、プログラマが生産性を高めるための機能やエコシステムが整備されています。
次のトピックについて、紹介します。

- トレイト / ジェネクリクス
- 代数的データ型 / パターンマッチ
- Cargo
- LLVM
- Rust Embedded WG

## トレイト / ジェネリクス

他のプログラミング言語と同様に、Rustでもポリモーフィズムが利用できます。
ポリモーフィズムを適切に利用することで、再利用性の高いコードを記述することができます。

Rustでは、トレイトとジェネリクス、という機能を使います。

### トレイト

トレイトは、C++の抽象基底クラス、Javaのインタフェースに似たものです。
例として、SPIデバイスにデータを書き込むためのトレイトを考えてみます。

世の中には様々なSPIデバイス（ここではSPI Masterを意味します）があります。
ただ、SPIをインタフェースとして使うデバイスドライバやアプリケーションとしては、それを意識せず、汎用的に書きたいと思うのが普通です。

例えば、8-bit単位で書き込みを行うSPIデバイスに書き込む関数は、次のように書くことができるでしょう。

```rust
/// ある8-bit単位で書き込めるSPIデバイスを表すstruct
struct MySpiU8 {
    ...
}

impl MySpiU8 {
    /// 書き込み関数。エラー処理は今は、考えません。
    fn write(&mut self, byte: u8) {
        // Do something for MySpiU8
        // 生ポインタを使って、メモリマップドデバイスに書き込みます。
        unsafe { &self.addr as *const _ as *mut u8　= word };
    }
}
```

この`MySpiU8`を使うアプリケーションは、`MySpiU8`専用のアプリケーションになってしまいます。

```rust
fn spi_write(spi: &mut MySpiU8, byte: u8) {
    spi.write(byte);
}
```

デバイスが例えば`AnotherSpiU8`に変わると、`MySpiU8`としている部分を書き直す必要があります。
SPIデバイスが変更になっても、アプリケーションを変更しなくて良いようにしたいです。

そこでトレイトの出番です。
例えば、SPIデバイスの書き込み処理を行うトレイトを次のように定義します。これはあくまでも、インタフェースの定義です。

```rust
pub trait SpiWrite {
    fn write(&mut self, word: u8);
}
```

このトレイトを、`MySpiU8`や`AnotherSpiU8`に対して**実装**します。

```rust
impl SpiWrite for MySpiU8 {
    fn write(&mut self, word :u8) {
        // Do something for MySpiU8
        unsafe { unsafe { &self.addr as *const _ as *mut u8　= word };
    }
}

impl SpiWrite for AnotherSpiU8 {
    fn write(&mut self, word :u8) {
        // Do something for AnotherSpiU8
        unsafe { unsafe { &self.addr as *const _ as *mut u8　= word };
    }
}
```

このように`SpiWrite`トレイトを実装している両者は、次のように同じコードで扱うことができます。

```rust
fn spi_write(spi: &mut SpiWrite, byte: u8) {
    spi.write(byte);
}
```

### ジェネリクス

Rustでポリモーフィズムを行うための、もう1つの方法です。これは、C++のテンプレートに相当します。

トレイトで使用したSPIの例（下記に再掲載）では、データが8-bit固定でした。しかし、SPI Masterによっては、16-bit単位で転送できるかもしれません。

```rust
pub trait SpiWrite {
    fn write(&mut self, word: u8);
}
```

今のトレイトでは、転送のデータ幅が変更になると、結局アプリケーションを変更しなければなりません。
そこで、`SpiWrite`トレイトをジェネリック関数を使用するように修正します。

```rust
pub trait SpiWrite<W> {
    fn write(&mut self, word: W);
}
```

ここで、`W`は任意の型であることを意味します。
この修正により、`write()`関数は、任意の型`W`に対して適用できるようになります。

`MySpiU8`と`AnotherSpiU16`があったとして、次のように定義します。任意の型`W`に対して、`u8`と`u16`という具体的な型を与えています。

```rust
impl SpiWrite<u8> for MySpi {
    fn write(&mut self, word :u8) {
        // Do something for MySpiU8
        unsafe { unsafe { &self.addr as *const _ as *mut u8　= word };
    }
}

impl SpiWrite<u16> for AnotherSpi {
    fn write(&mut self, word :u16) {
        // Do something for AnotherSpiU8
        unsafe { unsafe { &self.addr as *const _ as *mut u16　= word };
    }
}
```

これらの書き込みデータの幅が異なる2つの実装に対して、下記の関数は共通して使用することができます。

```rust
fn spi_write<W: Clone>(spi: &mut SpiWrite<W>, data: W) {
    // `clone()`は明示的なコピーを行う関数で、`Clone`トレイトによって実装されます。
    spi.write(data.clone());
}
```

ここで、`W: Clone`は、**Cloneトレイトを実装する任意の型**を意味します。この`: Clone`の部分は、`trait bound`と呼ばれ、型`W`に対して満たすべき制約を与えます。
C++をご存知の方は、次こそ入るであろうConceptのことだと考えて下さい。

上記関数内で、`data`は`clone()`関数を持っていなければなりません。これを保証するのが`Clone`トレイトです。
trait boundにより、ジェネリクスが求めている型が、どのような特性を満足しなければならないか、を明示することができます。

### トレイトオブジェクトとジェネリクス

さて、下記の`spi_write()`では、第一引数は`&mut SpiWrite<W>`を、第二引数は`<W: Clone>`を受け取ります。

```rust
fn spi_write<W: Clone>(spi: &mut SpiWrite<W>, data: W) {
    // `clone()`は明示的なコピーを行う関数で、`Clone`トレイトによって実装されます。
    spi.write(data.clone());
}
```

実は、この2つの使い方は、似て非なるものです。

前者の`&mut SpiWrite<W>`はトレイトオブジェクトと呼ばれます。後者の`<W: Clone>`はジェネリクスです。
重要な点は、トレイトオブジェクトが動的なポリモーフィズムを実現しているのに対して、ジェネリクスは静的なポリモーフィズムを実現している点です。

一般的に、ジェネリクスの方がより高速に動作します。ジェネリクスに対しては、コンパイラがコードを自動生成します。
大雑把に言うと、今回の場合、`spi_write()`は次のような2つの関数に展開されます。

```rust
fn spi_write_u8(spi: &mut SpiWrite<u8>, data: u8) {
    spi.write(data.clone());
}

fn spi_write_u16(spi: &mut SpiWrite<u16>, data: u16) {
    spi.write(data.clone());
}
```

Rustでは、第一の選択肢はジェネリクスです。
ただし、`Vec`のようなコレクションに複数の型を格納したい場合は、ジェネリクスは使えません。

トレイトオブジェクトとジェネリクスを使い分けることで、高速かつ抽象度の高いコードを記述することができます。

## 代数的データ型 / パターンマッチ

代数的データ型とパターンマッチの組み合わせを使うことで、直感的で安全な場合分け処理が記述できます。
代数的データ型はenumとも呼ばれますが、想定読者が知っているC/C++のenumよりはるかに強力です。

### 代数的データ型

Rustのenumは*データを持つ*ことができます。これは、どちらかというと共用体 (union) に似たものです。
しかし、Rustのenumは安全です。

[enc28j60 driver](https://github.com/japaric/enc28j60)のコードから、enumを利用している箇所を引用します。

```rust
enum Register {
    Bank0(bank0::Register),
    Bank1(bank1::Register),
    Bank2(bank2::Register),
    Bank3(bank3::Register),
    Common(common::Register),
}
```

enumの要素である、`Bank0`や`Bank1`はそれぞれ**別の型**を持っています。
構造体を要素として持つこともできます。

もちろん、Cのenumと同様の使い方も可能です。

```rust
pub enum Register {
    ERDPTL = 0x00,
    ERDPTH = 0x01,
    EWRPTL = 0x02,
    EWRPTH = 0x03,
    ETXSTL = 0x04,
    ETXSTH = 0x05,
    ETXNDL = 0x06,
...
}
```

### パターンマッチ

enumから値を取り出す時は、パターンマッチを使います。

```rust
    fn addr(&self) -> u8 {
        match *self {
            Register::Bank0(r) => r.addr(),
            Register::Bank1(r) => r.addr(),
            Register::Bank2(r) => r.addr(),
            Register::Bank3(r) => r.addr(),
            Register::Common(r) => r.addr(),
        }
    }
```

ここで重要なことは、`Bank0`型を格納すると、`Bank0`型としてしか取り出すことができない点です。
Cのunionと異なり、異なる型として処理することは、できません。

### enumによるエラー定義

Rustのenumは、エラーを定義するときも便利です。例えば、異なるレイヤで発生したエラーを便利に定義することができます。
`enc28j60`は、SPIを使用したEthernet MACデバイスであるため、SPIレイヤのエラーと、MACレイヤのエラーが発生します。
MACレイヤでのエラーはより上位レイヤでのエラーと考えることができます。

SPIレイヤでは複数のエラーが、MACレイヤでは`Late Collision`のエラーが発生するとします。

```rust
pub enum Error<E> {
    /// Late collision
    LateCollision,
    /// SPI error
    Spi(E),
}
```

上記のように`Error` enumを定義すると、SPIレイヤのエラーは、さらに任意の`E`型のデータを持つことができます。
まず、`LateCollision`と`Spi`どっちのエラーなのかをパターンマッチで処理し、`Spi`の場合はさらに、`E`型に対する処理を行うことができます。

このように異なるデータを持つenumに対して、switch caseのノリで処理することができます。

```rust
    match error {
        LateCollision => { /* error handling for `LateCollision` */ }
        Spi(e) => {
            match e {
                /* error handling for each of `E` type. */
            }
        }
    }
```

### enumを使ったエラーハンドリング

Rustのエラーハンドリングでは、`Result<T, E>`を使用します。処理が成功した時には、`T`型を、エラー発生時には`E`を保持します。
これは、ジェネリック (`T`と`E`が任意の型) な代数的データ型です。

例えば、SPIデバイスに8-bitのデータを書き込む関数を考えます。

Rustの`embedded_hal`には、[blocking::spi::Write](https://docs.rs/embedded-hal/0.1.0/embedded_hal/blocking/spi/trait.Write.html)というtraitがあります。
このtraitの`write()`関数は、SPIスレーブに`W`型のビット数単位で、データを送信します。
一般的なSPIデバイスの送信動作です。受信はしないため、成功時に受け取るデータはありません。そのため、成功時には空のタプル`()`が返ってきます。

```rust
pub trait Write<W> {
    // 任意のエラー型を関連型`Error`として使うことができます
    type Error;
    fn write(&mut self, words: &[W]) -> Result<(), Self::Error>;
}
```

この`write()`の実装例は次のようになります。わかりやすいように、具体的な型を書きます。

```rust
/// SPIデバイスを表すstructです。
pub struct Spi {
    ...
}

/// SPIのエラー定義です。
#[derive(Debug)]
pub enum SpiError {
    /// オーバーラン
    Overrun,
    /// モードフォルト
    ModeFault
}

/// `Write` traitをSpi structに実装します。
impl Write<u8> for Spi {
    // Self::Errorは、上で定義している`SpiError`になります。
    type Error = SpiError;

    fn write(&mut self, bytes: &[u8]) -> Result<(), SpiError> {
        // ステータスレジスタを読み込みます
        let status = self.sr.read();

        // SPIデバイスがエラー状態であれば、エラーを返します
        if sr.ovr().bit_is_set() {
            return Error(SpiError::Overrun)
        } else if sr.modf().bit_is_set() {
            return Error(SpiError::ModeFault)
        }

        // エラー状態でなれば、データを送信します
        for byte in bytes.iter() {
            unsafe { ptr::write_volatile(&self.spi.addr as *const _ as *mut u8, byte) }
        }
        Ok(())
    }
}
```

この`Spi`を使用するアプリケーションは、次のようにエラーハンドリングできます。

```rust
match spi.write(&data) {
    Ok(_) => { /* 成功時の処理 */ }
    Error(e) => {
        match e {
            SpiError::Overrun => { /* オーバーランのエラー処理 */ }
            SpiError::ModeFault => { /* モードフォールドのエラー処理 */ }
        }
    }
}
```

この例では愚直に書いていますが、エラーを上位に伝播する方法や、他のエラーと合成する方法など、様々なエラー処理がうまく書けるようになっています。

このように、代数的データ型とパターンマッチを使うことで、C言語よりはるかに強力で安全な場合分け処理を実現することができます。

## Cargo

[Cargo](https://doc.rust-lang.org/cargo/)は、Rustのビルドシステム兼パッケージマネージャです。
何でも卒なくこなしてくれる頼もしい貨物です。
Rustによる快適開発ライフの3分の1くらいを担っていると思います。

### package manager

まず、パッケージマネージャとして、こなして欲しい仕事を全て面倒見てくれます。

- セマンティックバージョニングで依存関係を管理
- 依存関係を、crates.io, git, pathの形で指定可能
- Cargo.lockでビルドの再現性を確保
- srcディレクトリ以下のファイルを自動で解析してビルド

[crates.io](https://crates.io/)は、Rustのcrate (パッケージ) を登録する場所です。
ここにあるcrateであれば、cargoの設定ファイル`Cargo.toml`に次のように書くだけで、依存を解決してくれます。

```toml
[dependencies]
time = "0.1.12"
```

ローカルに`time` crateがない場合、ソースコードを取得して、最終バイナリまで一気にビルドしてくれます。
crateのバージョンは、完全に指定することもできますし、セマンティックバージョニングで互換性が崩れない範囲で最新のものを取得することもできます。

gitで取得可能なcrateや、自身のプロジェクトの中で小さなcrateを複数使っても、簡単に管理できます。

```
[dependencies]
rand = { git = "https://github.com/rust-lang-nursery/rand" }
my_utils = { path = "../my_utils" }
```

srcディレクトリ下のファイルで、トップ (`main.rs`か`lib.rs`) から辿って利用しているソースファイルは自動的にビルドします。
面倒な設定は必要ありません。

### project configuration

`.cargo/config`に様々な設定を記述することができます。組込みRustでは非常にお世話になります。

例えば、RISC-V向けのクロスコンパイラを使用し、カスタムリンカスクリプトを指定、実行時にはqemu-riscvを立ち上げるようにします。
これは、`.cargo/config`に次のように記述するだけです (ツールのインストールは別途必要です)。

```toml
# ビルドターゲットがRISC-Vの時の設定です
[target.riscv32imac-unknown-none-elf]
# ランナーとしてqemu-riscvを使います
runner = "qemu-system-riscv32 -nographic -machine sifive_u -kernel"
# 自作のリンカスクリプトでリンクします
rustflags = [
  "-C", "link-arg=-Tlinker.ld",
]

# デフォルトでRISC-Vをビルドターゲットにします
[build]
target = "riscv32imac-unknown-none-elf"
```

このように設定ファイルを用意すると、

```
cargo run
```

だけで、RISC-Vのバイナリが生成され、qemuでバイナリを実行します。

### build script

ビルド時にRustで記述したビルドスクリプトを実行することができます。
例えば、依存するC言語のライブラリをビルドしたり、アセンブリで書いたブートストラップをビルドした上で、Rustのコードをビルドすることができます。

`cc` crateを使うと、非常に容易です。

```rust
extern crate cc;

fn main() -> Result<(), Box<Error>> {
    Build::new()
        .file("boot.s")
        .flag("-mabi=ilp32")
        .compile("asm");

    Ok(())
}
```

これで、`boot.s`をコンパイルして、Rustのコードとリンクしてくれます。
ただし、クロスコンパイラを使う場合は、環境変数の設定が必要です。

```
$ env CC=riscv32-unknown-linux-gnu-gcc  cargo run
```

### external tools

cargoはプラグインシステムがあるため、サブコマンドを自由に作ることができます。
組込みで一番有用なのは間違いなく`binutils`でしょう。

次のコマンドでプラグインをインストールするだけで、cargoのサブコマンドとして`binutils`が利用可能です。

```
$ rustup component add llvm-tools-preview
$ cargo install cargo-binutils --vers 0.1.4
```

下のような感じで使うことができます。

```
$ cargo objdump --bin app -- -d -no-show-raw-insn
```

## LLVM

RustはLLVMをバックエンドに持つプログラミング言語であるため、クロスコンパイルがお手軽です。
ARMv7のクロスコンパイラをインストールするのは、コマンド1つです。

```
$ rustup target add thumbv7m-none-eabi
```

## コミュニティ

