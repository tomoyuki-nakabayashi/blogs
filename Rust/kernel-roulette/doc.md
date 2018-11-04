# shim

さすがにC言語の層は必要。
でもかなり薄い。
と思ったらsystem callをC言語で受けている。

```c
/*
 * The entry points in C
 */
static int _mod_init(void) {

  // Register a character device
  rl_dev_major_num = register_chrdev(0 /* allocate a major number */, rl_device_name, &rl_driver_fops);

  if (rl_dev_major_num < 0) {
    printk(KERN_ALERT "failed to register character device: got major number %d\n", rl_dev_major_num);
    return rl_dev_major_num;
  }

  printk(KERN_INFO "Registered %s with major device number %d\n", rl_device_name, rl_dev_major_num);
  printk(KERN_INFO "Run /bin/mknod /dev/%s c %d 0\n", rl_device_name, rl_dev_major_num);

  return rust_mod_init();
}

module_init(_mod_init);
```

# Rust

```c
/*
 * Functions defined in Rust
 */
extern u8 sample(void);
extern void set_chance(u8 chance);
extern u8 get_chance(void);
```

print!()が呼ばれると…？

```rust
// Entry points
#[no_mangle]
pub extern "C" fn rust_mod_init() -> i32 {
    print!("Panic probability: {}/{}\n", CONFIG.lock().chance, MAX_RAND);
    0
}
```

io::print()が呼ばれる。
この中では、writerをlock()して、write_fmt()して、flush()している。

```rust
/// Like the `print!` macro in the standard library, but calls printk
#[allow(unused_macros)]
macro_rules! print {
    ($($arg:tt)*) => ($crate::io::print(format_args!($($arg)*)));
}

/// Like the `print!` macro in the standard library, but calls printk
#[allow(unused_macros)]
macro_rules! println {
    ($fmt:expr) => (print!(concat!($fmt, "\n")));
    ($fmt:expr, $($arg:tt)*) => (print!(concat!($fmt, "\n"), $($arg)*));
}

pub fn print(args: fmt::Arguments) {
    use core::fmt::Write;
    let mut writer = PRINTK_WRITER.lock();
    writer.write_fmt(args).unwrap();
    writer.flush();
}
```

KernelDebugWriterは、bufferをVec(そう！Vec！)で保持している。
`self.buffer.extend(s.bytes());`といったヒープ領域使う操作も問題なく動作する。

`flush()`は、C言語のputs_c()を呼んでいる。
これは想像通り、printk()を呼び出す。

```rust
#[derive(Default)]
pub struct KernelDebugWriter {
    buffer: Vec<u8>,
}

extern "C" {
    fn puts_c(len: u64, c: *const u8);
}

impl KernelDebugWriter {
    fn flush(&mut self) {
        // Call c_puts with the string length and a pointer to its contents
        unsafe { puts_c(self.buffer.len() as u64, self.buffer.as_ptr() as *const u8) };
        self.buffer.clear();
    }
}

impl fmt::Write for KernelDebugWriter {
    fn write_str(&mut self, s: &str) -> fmt::Result {
        // Add the bytes from `s` to the buffer
        self.buffer.extend(s.bytes());
        Ok(())
    }
}
```

```c
/*
 * A utility function to print a string.
 * NOTE: `str` is not necessarily NULL-terminated.
 */
void puts_c(u64 length, char* str) {
  printk(KERN_DEBUG "%.*s", (int)length, str);
}
```

# 参考

https://qiita.com/termoshtt/items/7d97bfb5d7e31e7b341b
https://forge.rust-lang.org/platform-support.html
