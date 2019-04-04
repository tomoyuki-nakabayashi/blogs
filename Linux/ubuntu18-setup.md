# Ubuntu 18 setup

## 日本語入力

[Ubuntu 18.04 LTSで日本語が入力できない！どうすればいい？](https://linuxfan.info/ubuntu-18-04-japanese-input)の手順どおり、language supportをインストールして、Mozcを有効化する。

## 開発ツール

```
apt install git curl vim
```

## Rust

### rustup

https://rustup.rs/

```
curl https://sh.rustup.rs -sSf | sh

```

```
$ tail -n 2 .bashrc 
export PATH="$HOME/.cargo/bin:$PATH"
```

## その他

- chrome
- vscode
