# MISRA-C Rust

## 信頼性

### 領域は初期化し、大きさに気を付けて使用する。

Rustでは、初期化していない変数を使うとコンパイルエラーになります。
スライスやコレクションをインデックスでアクセスする場合、最悪でも動的チェックが入り、定義されたパニックが発生します。

### 領域は、初期化してから使用する

Rustでは、初期化していない変数を使用すると、コンパイルエラーになる。

```rust
    let x: u32;
    println!("{}", x);
```

```
error[E0381]: borrow of possibly uninitialized variable: `x`
4 |     println!("{}", x);
  |                    ^ use of possibly uninitialized `x`
```

### 要素数を指定した配列の初期化では、初期値の数は、指定した要素数と一致させる。

Rustでは、スライスのサイズは型の一部であり、その型を満たさない初期化はコンパイルエラーとなる。

```rust
    let x: [u32; 4] = [1, 2, 3];
```

```
error[E0308]: mismatched types
2 |     let x: [u32; 4] = [1, 2, 3];
  |                       ^^^^^^^^^ expected an array with a fixed size of 4 elements, found one with 3 elements
  |
  = note: expected type `[u32; 4]`
             found type `[u32; 3]`
```

### ポインタの指す範囲に気を付ける

1. ポインタへの整数の加減算（++、－－も含む）は使用せず、確保した領域への参照・代入は[ ]を用いる配列形式で行う
2. ポインタへの整数の加減算（++、－－も含む）は、ポインタが配列を指している場合だけとし、結果は、配列の範囲内を指すようにする。

Rustでは、生ポインタの使用は、限られた状況でしか使いません。

先に、MISRA-Cの適合例を示します。

```c
#define N 10
int data[N];
int *p;
int i;
p = data;
i = 1;
（1）、（2）の適合例
data[i] = 10; /* OK */
data[i+3] = 20; /* OK */
（2）の適合例
*(p + 1) = 10;
```

1について、Rustにおけるインデックスアクセスは、静的および動的に境界チェックが行われるため、更に安全です。
下のような単純な例は、コンパイルエラーになります。

```rust
    let x: [u32; 4] = [1, 2, 3, 4];
    
    println!("{}", x[3]);
    println!("{}", x[3+1]);
```

```
error: index out of bounds: the len is 4 but the index is 4
  |
5 |     println!("{}", x[3+1]);
  |                    ^^^^^^
```

少し複雑なパターンでも、実行時、パニックになります。これは **定義された動作** です。

```rust
fn array_index(array: &[u32], index: usize) -> u32 {
    array[index]
}

fn main() {
    let x: [u32; 4] = [0, 1, 2, 3];
    array_index(&x, 4);
}
```

```
thread 'main' panicked at 'index out of bounds: the len is 4 but the index is 4', src/main.rs:2:5
```

2について、Rustでは生ポインタの参照外しは、`unsafe`ブロック内でしかできません。
あえて、面倒くさいコードを書かない限りは、ルールに反することはありません。

```rust
    let mut x: [u32; 4] = [0, 1, 2, 3];
    let p = &mut x[0] as *mut u32;
    unsafe { *p.offset(1) = 10 };
```

# MISRA-Rust

[MISRA-Rust](https://github.com/PolySync/misra-rust)

## デフォルトでコンパイルエラーにならない例

[MISRA-Rules](https://github.com/PolySync/misra-rust/blob/master/MISRA-Rules.md)

MISRAのルールは下から引用しています。

http://www.c-lang.org/detail/misra_c.html

## コメント

### 3.1 「/*」や「//」という文字の並びをコメント内で使用してはならない

どうでも良さそうですね。

```rust
fn main() {
    /* /* Nested comment */ // Nested comment */
    //~^ ERROR nested comments
}
```

## 識別子

MISRAの基本方針が、「識別子を再利用しない」なので、Rustがこのあたりのルールに準拠しないのは当然です。
Rustはシャドーイングを許す言語ですが、これは次のような利点があります。

- 可変性の制御
- 中間変数

可変性の制御では、ある変数が変更不要になったことをコンパイラに教えることができます。

```rust
    let mut x = 10;
    /* We can modify x here. */

    let x = x;
    /* We no longer modify x! */
```

また、処理が連鎖するような場合、シャドーイングにより、誤って中間結果を使うことがなくなります。

```rust
    let data = 0;

    let data = do_something(data);
    let data = do_another(data);

    let data = data.unwrap();
```

シャドーイングする度に、その前の`data`にはアクセスできなくなります。
これは、下手に一時変数を作るより安全ではないでしょうか。

### 5.1 外部識別子は異なったものにしなければならない

テストコードがおかしい気がします。外部に公開する識別子が一意になれば良い気がしますが…？

```rust
const ABC: i32 = 0;

fn main() {
    let abc: i32 = 1;
    //~^ ERROR Non-compliant - variable name shadows ABC
    let _ = abc + ABC;
}
```

### 5.2 同じスコープと名前空間で宣言された識別子は異なったものにしなければならない

当然シャドーイングが発生します。

```rust
fn main() {
    let engine_exhaust_gas_temperature_raw: i32 = 0;
    let engine_exhaust_gas_temperature_raw: i32 = 1;
    //~^ ERROR Non-compliant - variable name shadows engine_exhaust_gas_temperature_raw
}
```

### 5.3 内側のスコープで宣言された識別子は、外側のスコープで宣言された識別子を隠してはならない

はい。

```rust
fn main() {
    let i: i16 = 1;
    if true {
        let i: i16 = 0; //~ ERROR Non-compliant - `i` shadows outer scope.
        let _ = i;
    }
    let _ = i;
}
```

しかし、`static`や`const`のシャドーイングは、コンパイルエラーになります。

```rust
    static abc: i32 = 0;
    {
        let abc: i32 = 1;
    }
```

```
error[E0530]: let bindings cannot shadow statics
  |
4 |     static abc: i32 = 0;
  |     -------------------- the static `abc` is defined here
5 |     {
6 |         let abc: i32 = 1;
  |             ^^^ cannot be named the same as a static
```

そのため、グローバル領域に宣言したstatic変数や定数については、シャドーイングするとコンパイルエラーになります。
関数ローカルでしかシャドーイング起きない（？）ので、けっこう安全な気がします。

### 5.4 マクロ識別子は異なったものにしなければならない

通るんですね。意外。ちゃんと`unused macro definition`の警告は出ます。

```rust
macro_rules! engine_exhaust_gas_temperature_raw {
    () => {
        3;
    };
}

macro_rules! engine_exhaust_gas_temperature_raw {
    //~^ ERROR Non-compliant - variable name shadows engine_exhaust_gas_temperature_raw
    () => {
        4;
    };
}

fn main() {
    let _ = engine_exhaust_gas_temperature_raw!();
    let _ = engine_exhaust_gas_temperature_raw!();
}
```

### 5.6 typedef名は一意の識別子でなければならない

ルール的には、関連型とか全部ダメですよね…。

```rust
fn main() {
    type U8 = bool;
    {
        type U8 = u8;
        //~^ ERROR Non-compliant - type name shadows U8
    }
}
```

## リテラルと定数

### 7.2 符号なしの型で表現されているすべての整数定数には「u」または「U」接尾語を適用しなければならない

```rust
fn main() {
    let compliant_unsigned: u32 = 1u32;
    let unsigned: u32 = 1;
    //~^ ERROR Non-compliant - suffix specifying unsigned type required on integer constants
}
```

C言語と違って、下はちゃんとエラーになります。

```rust
    let unsigned: u32 = -1;
```

```
error[E0600]: cannot apply unary operator `-` to type `u32`
  |
2 |     let unsigned: u32 = -1;
  |                         ^^ cannot apply unary operator `-`
  |
  = note: unsigned values cannot be negated
```

C言語は、下のコードはコンパイルが通っちゃうので大変です。

```c
    uint32_t u = -1;
```

### 7.4 オブジェクトの型が「const修飾文字へのポインタ」でない限り、文字列リテラルをオブジェクトに代入してはならない

普通にできちゃいますわな。

```rust
fn main() {
    let mut _l = "string literal";
    //~^ ERROR Non-compliant - string literal not const-qualified.
}
```

## 宣言と定義

### 8.1 型は明示的に指定されなければならない

まぁ、型推論ありますからね…。

```rust
fn main() {
    let x;
    //~^ ERROR Non-compliant - type not explicitly specified
    x = 1;
```

### 8.7 翻訳単位内での参照がただ1つである場合、関数やオブジェクトは外部リンケージを使用して定義してはいけない

これは、clippyさんでも警告が出なかったです。

```rust
pub const LIBRARY_GLOBAL: u32 = 0;
//~^ ERROR Non-compliant - public but not used
```

### 8.9 識別子が単一の関数内にのみ出現する場合、そのオブジェクトはブロックスコープで定義されなければならない

これも、clippyさんでも警告出ないですね。

```rust
const GLOBAL: u32 = 0;
fn main() {
    let _x = GLOBAL + 1;
    //~^ ERROR Non-compliant - global only used at block scope
}
```

### 8.13 ポインタは可能な限りconst修飾型を指さなければならない

これは、残りのコードで使っていなければ、`mut`外せるよ、の警告が出ます。

```rust
    let mut x: Box<u8> = Box::new(8);
    //~^ ERROR Non-compliant - "pointer" is not const-qualified.
```

## 実質的な型モデル

### 10.1 オペランドが不適切な実質的な型であってはならない

テストコードが変な気がします。MISRAは暗黙変換のことを言っていると思うのですが…。

```rust
    let x: i32 = 0xFF;
    let y = x << 2;
    //~^ ERROR Non-compliant - inappropriate essential type
```

CERT Cの似た項目にある、下がちゃんと動けば良いのでは？
C言語では、`0x0a`が期待値ですが、最初に暗黙変換されてint32_tあたりになる結果、`0xfa`になります。

```c
    uint8_t port = 0x5a;
    uint8_t result_8 = ( ~port ) >> 4;
```

Rustでは、これはちゃんと`0x0a`になります。`4`が`u8`に型推論されるため(？)ですかね。

```rust
    let x: u8 = 0x5a;
    let y = ( !x ) >> 4;
```

### 10.8 複合式の値は異なる実質的な型分類やより広い実質的な型にキャストしてはならない

これもテストケースがおかしい気がしますね…。複合式の中で最終結果より大きい型にキャストされてはいけない、というルールっぽいのですが。

```rust
    let x: u16 = 1;
    let y: u16 = 2;
    let _: u32 = (x + y) as u32;
    //~^ ERROR Non-compliant - composite cast to wider type
```

下のようなコードが通らなければ良い？下はコンパイルエラーになります。

```rust
    let x: u8 = 255;
    let y: u8 = (x + 1u16) as u8;
```

## 式

### 12.1 式の中の演算子の優先順位は明白でなければならない

下はコンパイルエラーにはなりません。clippyさんは`()`を付けなさい、と言ってくれます。

```rust
    let x: usize = 1;
    if x >= 2 << 2 + 1 as usize {
        //~^ ERROR Non-compliant - operator precedence can trip the unwary
    }
```

### 12.4 定数式の評価は、符号なし整数のラップアラウンドを引き起こしてはならない

へー、これ、コンパイル時はダメなんですね。

```rust
    let u8a: u8 = 0;
    let _x = u8a - 10;
    //~^ERROR Non-compliant - attempt to subtract with overflow
```

皆さんご存知の通り、実行時は定義されたパニックが発生します。

```
thread 'main' panicked at 'attempt to subtract with overflow', src/main.rs:3:14
```

## 副作用

### 13.2 式の値とその永続的な副作用は、すべての許可された評価順で同じでなければならない

Rustは式の評価順決まっていそうですが…？
明示的に書いてあるドキュメントは見つかっていませんが、[Document function argument evaluation order (or lack thereof)](https://github.com/rust-lang-nursery/reference/issues/332)などを見ても、左から右へ評価される模様です。

そうでないと、ボローチェッカがうまくチェックできない気がします。

```rust
/// This function has a side effect.
fn increment(x: &mut u8) -> &mut u8 {
    *x += 1;
    x
}

/// This function does not have a side effect.
fn add(x: u8, y: u8) -> u8 {
    x + y
}

fn main() {
    let mut x: u8 = 0;
    let _ = add(*increment(&mut x), x);
    //~ ERROR Non-compliant - evaluation order effects result
}
```

### 13.5 &&や||の論理演算子の右側のオペランドは、永続的な副作用を含んではならない

&&や||は、短絡評価されるので、これはいけません。clippyさんも叱ってくれないですね。

```rust
/// This function has a side effect.
fn not(x: &mut bool) -> &mut bool {
    *x = !*x;
    x
}

fn main() {
    let mut x: bool = true;

    if *not(&mut x) || *not(&mut x) {
        //~^ ERROR Non-compliant - right hand operand contains persistent side-effects
    }
}
```

## 制御文の式

### 14.1 ループカウンタは、実質的に浮動小数点型を持ってはいけない

ループカウンタ扱い...なのかな？

```rust
    let mut f: f64 = 0_f64;

    while f < 1.0 {
        f += 0.001_f64;
        //^ ERROR Non-compliant - loop counter with floating point type
    }
```

### 14.2 forループは適正に定義されなければならない

コンパイルエラーにはなりませんが、この書き方の場合、**ループの回数自体は変化しません**。100回きっちり周ります。

```rust
    let mut bound: u32 = 100;

    for i in 0..bound {
        bound -= i;
        //~^ ERROR Non-compliant - attempt to mutate range bound within loop
    }
```

clippyさんも以下の苦言を呈してくれます。

```
warning: attempt to mutate range bound within loop; note that the range of the loop is unchanged
  |
6 |         bound -= i;
  |         ^^^^^^^^^^
```

ただし、`bound`が途中でoverflowするので、実行時に定義されたパニックが発生します。

```
thread 'main' panicked at 'attempt to subtract with overflow', src/main.rs:7:9
```

### 14.3 制御式は不変であってはならない

意外とこれ叱ってもらえないんですね…。

```rust
    let a: i32 = 0;
    if (a < 10) && (a > 20) {
        //~^ ERROR Non-compliant - always true
    }
```

## 制御フロー

### 15.5 関数は、その最後に1つだけの出口を持たなけらばならない

まぁ、これはね…。必須ルールではありませんし。リソース解放忘れが通常は発生しないので、Rustではガンガン早期リターンすれば良いと思います。

```rust
    let x = 1;

    if x > 1 {
        return;
    }

    if x < 1 {
        return;
        //~^ ERROR Non-compliant - more than one exit point from function
    }
```

### 15.7 すべてのif ... else if構文は、else文で終了しなければならない

clippyさんも反応なし。

```rust
fn main() {
    let x = 1;
    if x == 2 {
        let _ = 1;
    } else if x == 3 {
        let _ = 2;
    }
    //~^ ERROR Non-compliant - no terminating `else`
}
```

## switch文

### 16.6 すべてのswitch文は、少なくとも2つのスイッチ節を持たなければならない

clippyさんも反応なし。

```rust
    let i = 1;
    match i {
        _ => {} //~ ERROR Non-compliant - less that two clauses
    }
```

### 16.7 スイッチ式は実質的にブール型を持ってはいけない

clippyさんから、if/else式にしなよ、とお叱りを受けます。

```rust
    let i = true;

    match i as bool {
        //~^ ERROR Non-compliant - match on a boolean expression
        false => {
            let _ = 1;
        }
        _ => {
            let _ = 2;
        }
    }
```

## 関数

### 17.2 関数は、直接的または間接的に、自分自身を呼び出しはいけない

そうだった。再帰呼出し禁止なんですよね…。

### 17.7 void以外の戻り値の型を持つ関数が返す値は使用されなければならない

`_`で捨てるコードを書けば良いだけなのですが、それに意味があるか、という話ではありますね。
Result型は使わないと警告出ます。

```rust
fn func(para1: u16) -> u16 {
    para1
}

fn discarded(para2: u16) {
    func(para2);
    //~^ ERROR Non-compliant - `func` return discarded
}

fn main() {
    discarded(1);
}
```

### 17.8 関数パラメータを変更してはいけない

Rustでこんなコード書く人居るのかな…という気持ちになりながらも、コンパイルエラーにはなりません。

```rust
fn paramod(mut para: u16) -> u16 {
    para += 1; //~ ERROR parameter modified without persistent effect
    let _ = para;
    1
}

fn main() {
    paramod(1);
}
```

## ポインタと配列

### 18.5 2段階を超える入れ子のポインタを宣言してはいけない

これ、clippyさんの警告が面白いです。

```rust
fn nesting(p: &&&&&&[u8; 10]) {
    let _ = ****p;
}

fn main() {
    let a = [5; 10];
    nesting(&&&&&&a);
    //~^ ERROR Non-compliant - reference nesting exceeds maximum allowed
}
```

```
warning: this argument is passed by reference, but would be more efficient if passed by value
  |
1 | fn nesting(p: &&&&&&[u8; 10]) {
  |               ^^^^^^^^^^^^^^ help: consider passing by value instead: `&&&&&[u8; 10]`
```

1個減らせ、とのことです。ちなみに`&`が1個になるまで、永遠に1個減らせ、と言われます。ウケる。

## 重なり合う記憶域

### 19.2 unionキーワードを使用してはならない

使えますが、`unsafe`です。

```rust
union UnionA {
    f1: i16,
    f2: i32,
}

fn main() {
    let mut u = UnionA { f2: 0 };
    unsafe { u.f1 = u.f2 as i16 };
    //~^ ERROR mismatched types
}
```

## 前処理指令

### 20.1 #include指令に対しては、他の前処理指令やコメントのみが先行しうる

`#include`と違って、`use`は順番に依存しないので、ちょっと話が別な気がします。

```rust
struct MyStruct {
    a: u32,
}

fn func(_: MyStruct) {}

use std::fmt;
//~ ERROR Non-compliant - `include` directive preceeded by something other than macros or comments

impl fmt::Display for MyStruct {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "MyStruct{{ {} }}", self.a)
    }
}

fn main() {
    let s = MyStruct { a: 10 };
    func(s);
}
```

### 20.2 「'」、「"」または「\」文字、「/*」または「//」文字列がヘッダファイル名に存在してはならない

これ処理できるんですね。そもそもヘッダファイルがないので…。

```rust
include!("../../include/_'_.rs");
//~ ERROR invalid character `'` in crate name: `Rule_20_2_'`

fn main() {}
```

## 標準ライブラリ

### 21.1 予約済み識別子や予約済みマクロ名に対して#defineや#undefを使用してはならない

`println`って予約済みマクロなんでしょうか…？

```rust
macro_rules! println {
    //~ ERROR Non-compliant - redefinition of macro name
    () => {
        3;
    };
}

fn main() {
    println!();
}
```

### 21.2 予約済み識別子またはマクロ名を宣言してはならない

21.1と同じです。

## 参考

[ESCR Ver. 2.0](https://www.ipa.go.jp/files/000045164.pdf)