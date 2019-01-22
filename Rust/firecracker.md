# Rustで書かれたVMM firecrackerを読もう！(1)

[firecracker](https://firecracker-microvm.github.io)という最高の餌が与えられたので、少しずつ時間を取ってコード解析していこうと思います。

## firecracker build

何はともあれ、ビルドしてみましょう。
リリースビルドするときは、`--release`をつければ良いみたいです。
とりあえず、debugビルドします。

```
$ git clone https://github.com/firecracker-microvm/firecracker.git
$ cd firecracker/
r$ git checkout v0.11.0
$ tools/devtool build
[Firecracker devtool] About to pull docker image fcuvm/dev:latest
[Firecracker devtool] Continue? (y/n) y
[Firecracker devtool] Updating fcuvm/dev:latest ...
[Firecracker devtool] Starting build (debug) ...
    Updating crates.io index
 Downloading chrono v0.4.6                                                      
 Downloading backtrace v0.3.11                                                  
 Downloading serde_json v1.0.17
 Downloading clap v2.27.1                                                       
 Downloading num-integer v0.1.39                                                
 Downloading time v0.1.40                                                       
 Downloading num-traits v0.2.6                                                  
 Downloading libc v0.2.45                                                       
 Downloading regex v1.1.0                                                       
 Downloading serde_derive v1.0.27                                               
 Downloading serde v1.0.27                                                      
 Downloading thread_local v0.3.6                                                
 Downloading utf8-ranges v1.0.2                                                 
 Downloading regex-syntax v0.6.4                                                
 Downloading memchr v2.1.2                                                      
 Downloading aho-corasick v0.6.9                                                
 Downloading lazy_static v1.2.0                                                 
 Downloading ucd-util v0.1.3                                                    
 Downloading cfg-if v0.1.6                                                      
 Downloading version_check v0.1.5                                               
 Downloading itoa v0.4.3                                                        
 Downloading dtoa v0.4.3                                                        
 Downloading vec_map v0.8.1                                                     
 Downloading atty v0.2.11                                                       
 Downloading textwrap v0.9.0                                                    
 Downloading ansi_term v0.9.0                                                   
 Downloading unicode-width v0.1.5                                               
 Downloading bitflags v0.9.1                                                    
 Downloading strsim v0.6.0                                                      
 Downloading syn v0.11.11                                                       
 Downloading quote v0.3.15                                                      
 Downloading serde_derive_internals v0.19.0                                     
 Downloading synom v0.11.3                                                      
 Downloading unicode-xid v0.0.4                                                 
 Downloading rustc-demangle v0.1.9                                              
 Downloading backtrace-sys v0.1.24                                              
 Downloading cc v1.0.25                                                         
 Downloading tokio-uds v0.1.7                                                   
 Downloading tokio-core v0.1.12                                                 
 Downloading tokio-io v0.1.5                                                    
 Downloading hyper v0.11.16                                                     
 Downloading futures v0.1.18                                                    
 Downloading mio-uds v0.6.7                                                     
 Downloading bytes v0.4.11                                                      
 Downloading iovec v0.1.2                                                       
 Downloading log v0.3.9                                                         
 Downloading mio v0.6.16                                                        
 Downloading log v0.4.6                                                         
 Downloading slab v0.4.1                                                        
 Downloading net2 v0.2.33                                                       
 Downloading lazycell v1.2.1                                                    
 Downloading byteorder v1.2.1                                                   
 Downloading scoped-tls v0.1.2                                                  
 Downloading unicase v2.2.0                                                     
 Downloading httparse v1.3.3                                                    
 Downloading base64 v0.9.3                                                      
 Downloading language-tags v0.2.2                                               
 Downloading relay v0.1.1                                                       
 Downloading tokio-proto v0.1.1                                                 
 Downloading percent-encoding v1.0.1                                            
 Downloading tokio-service v0.1.0                                               
 Downloading futures-cpupool v0.1.8                                             
 Downloading mime v0.3.12                                                       
 Downloading safemem v0.3.0                                                     
 Downloading take v0.1.0                                                        
 Downloading slab v0.3.0                                                        
 Downloading rand v0.3.22                                                       
 Downloading smallvec v0.2.1                                                    
 Downloading rand v0.4.3                                                        
 Downloading num_cpus v1.9.0                                                    
 Downloading timerfd v1.0.0                                                     
 Downloading epoll v2.1.0                                                       
 Downloading rand v0.6.1                                                        
 Downloading bitflags v1.0.4                                                    
 Downloading rand_hc v0.1.0                                                     
 Downloading rand_core v0.3.0                                                   
 Downloading rand_chacha v0.1.0                                                 
 Downloading rand_isaac v0.1.1                                                  
 Downloading rand_xorshift v0.1.0                                               
 Downloading rand_pcg v0.1.1                                                    
 Downloading rustc_version v0.2.3                                               
 Downloading semver v0.9.0                                                      
 Downloading semver-parser v0.7.0                                               
 Downloading json-patch v0.2.1                                                  
 Downloading bitflags v0.7.0
...
    Finished dev [unoptimized + debuginfo] target(s) in 4m 48s                  
[Firecracker devtool] Build complete. Binaries placed under <build_dir>/firecracker/build/debug/
```

さすがに多くのcrate使ってますね。
何故かbitflagsが3 version存在しています。なんでしょうこれ。

```
 Downloading bitflags v0.7.0
 Downloading bitflags v0.9.1
 Downloading bitflags v1.0.4
```

firecrackerとjailerというバイナリが生成されています。

```
$ ls build/debug/
firecracker  jailer
```

`Jail a microVM`、なんかかっこいいですね。

```
$ ./build/debug/jailer -h
jailer 0.11.0
Amazon Firecracker team <firecracker-devel@amazon.com>
Jail a microVM.
```

## ソースコードを見る前に

最低限押さえておきましょう。
HTTP APIが生えていて、VM作成や管理ができるようです。モダンですね。
そのため、ソースコードは、起動周りを見て、イベント待ちっぽいところにたどり着いたら、HTTP API実装を見ていけば良さそうです。

## Cargo.toml

まずは、Cargoの設定ファイルから見てみましょう。
`.cargo/config`

```
[build]
target = "x86_64-unknown-linux-musl"
```

musl使ってますね。C言語ライブラリに依存しないバイナリにしているようです。ライブラリとの依存を持ちたくないでしょうし、妥当な感じですね。

Cargo.tomlは思ったより見るものなかったです。panic strategyが`abort`なことくらいでしょうか。

```toml
[profile.release]
lto = true
panic = "abort"
```

## src

では、さっそくsrcから見ていきましょう。
jailerもいますね。

```
$ tree src/
src/
├── bin
│   └── jailer.rs
└── main.rs
```

ちょっと寄り道をして、`jailer.rs`を覗き見してみます。

```rust
extern crate chrono;
extern crate clap;

extern crate fc_util;
extern crate jailer;

fn main() -> jailer::Result<()> {
    jailer::run(
        jailer::clap_app().get_matches(),
        (chrono::Utc::now().timestamp_nanos() / 1000) as u64,
        fc_util::now_cputime_us(),
    )
}
```

なんか本当に、`run`しているだけみたいですね。
jailerは追々見ていきまそう。

main.rsに移ります。まずは、ロガーの初期化をしています。

```rust
fn main() {
    LOGGER
        .init(&"", None, None)
        .expect("Failed to register logger");
...
```

`LOGGER`は、logger crateでlazy_staticを使って、定義しています。典型的なグローバルオブジェクトの初期化っぽいです。
ロガーは、複数スレッドからアクセスされるため、排他制御を行っているようですね。

```rust
lazy_static! {
    static ref _LOGGER_INNER: Logger = Logger::new();
}

lazy_static! {
    /// Static instance used for handling human-readable logs.
    ///
    pub static ref LOGGER: &'static Logger = {
        set_logger(_LOGGER_INNER.deref()).unwrap();
        _LOGGER_INNER.deref()
    };
}
```

`set_logger`は[log crate](https://docs.rs/log/0.4.6/log/)の関数です。

```rust
pub fn set_logger(logger: &'static Log) -> Result<(), SetLoggerError> {
    set_logger_inner(|| logger)
}

fn set_logger_inner<F>(make_logger: F) -> Result<(), SetLoggerError>
where
    F: FnOnce() -> &'static Log,
{
    unsafe {
        match STATE.compare_and_swap(UNINITIALIZED, INITIALIZING, Ordering::SeqCst) {
            UNINITIALIZED => {
                LOGGER = make_logger();
                STATE.store(INITIALIZED, Ordering::SeqCst);
                Ok(())
            }
            INITIALIZING => {
                while STATE.load(Ordering::SeqCst) == INITIALIZING {}
                Err(SetLoggerError(()))
            }
            _ => Err(SetLoggerError(())),
        }
    }
}
```

`set_logger_inner`では、正常ケースにおいて(`UNINITIALIZED`のアーム)、compare_and_swapを使って、アトミックにLOGGERの状態を初期化済みに変更します。
LOGGERの状態は次のように定義されています。

```rust
// The LOGGER static holds a pointer to the global logger. It is protected by
// the STATE static which determines whether LOGGER has been initialized yet.
static mut LOGGER: &'static Log = &NopLogger;
static STATE: AtomicUsize = ATOMIC_USIZE_INIT;

// There are three different states that we care about: the logger's
// uninitialized, the logger's initializing (set_logger's been called but
// LOGGER hasn't actually been set yet), or the logger's active.
const UNINITIALIZED: usize = 0;
const INITIALIZING: usize = 1;
const INITIALIZED: usize = 2;
```

今日は、ここまで！