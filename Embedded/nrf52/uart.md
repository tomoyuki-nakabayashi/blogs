# UART

UARTでおしゃべりしたい。

## Zephyr調査

### Zephyr printk()の出力先は？

`hello world`サンプルは、printk()に出力しているが、果たしてUARTにつながっているのだろうか？

dtsが下のようになっているから、さすがに、uart0に出ていても良さそうだな。

```
/ {
	model = "Nordic PCA10059 Dev Kit";
	compatible = "nordic,pca10059-dk", "nordic,nrf52840-qiaa",
		     "nordic,nrf52840";

	chosen {
		zephyr,console = &uart0;
		zephyr,uart-mcumgr = &uart0;
		zephyr,sram = &sram0;
		zephyr,flash = &flash0;
	};
```

### PIN設定とGPIOとの紐付け

これは、実際どこのGPIOにつながっている？

```
&uart0 {
	compatible = "nordic,nrf-uart";
	current-speed = <115200>;
	status = "ok";
	tx-pin = <20>;
	rx-pin = <24>;
};
```

ちゃんと、GPIO 0.20につながっているっぽいことがわかった。
↓で動いたので。

defconfigの`# CONFIG_UART_CONSOLE is not set`はこれで良い？
https://docs.zephyrproject.org/1.3.0/reference/kconfig/CONFIG_UART_CONSOLE.html

上を見ると、有効にしないといけない気がする。

pca10056のdefconfigを見ると、`CONFIG_UART_CONSOLE`が有効になっている。

```
# enable console
CONFIG_CONSOLE=y
CONFIG_UART_CONSOLE=y
```

pca10059のdefconfigを見ると、`CONFIG_RTT_CONSOLE`が有効になっている。

```
# enable console
CONFIG_CONSOLE=y
CONFIG_RTT_CONSOLE=y
```

https://docs.zephyrproject.org/1.13.0/reference/kconfig/CONFIG_RTT_CONSOLE.html
を見ると、`CONFIG_RTT_CONSOLE`は、J-Linkに出力するためのものなので、これではダメだね。

ZephyrのmenuconfigでUART_CONSOLEを有効にする。
反応無し。

## オシロスコープ

あー、ハードがダメっぽいな。ピンを左右からしっかり挟まないとVBUSの波形が正常に出ない。
接触不良を解決しないといけないが、とりあえずピンを挟めばなんとかなる。

きた！！！

```
***** Booting Zephyr OS v1.13.99-ncs2 *****
Hello World! nrf52840_pca10059
```

## 手順

### dts修正

`zephyr/boards/arm/nrf52840_pca10059/nrf52840_pca10059.dts`のUARTが、UARTEを使うようになっているので、UARTを使うように、下の通り修正する。

```
&uart0 {
	compatible = "nordic,nrf-uart";
	current-speed = <115200>;
	status = "ok";
	tx-pin = <20>;
	rx-pin = <24>;
};
```

### hello_worldサンプルのビルド

サンプルプロジェクトのビルド準備をする。

```
cd <zephyr>/samples/hello_world
mkdir build && cd build
cmake -GNinja -DBOARD=nrf52840_pca10059 ..
```

menuconfigで少し修正する。TODO: これは、pca10059ボードのdefconfigに反映する。

```
ninja menuconfig
```

`Device Drivers -> Console drivers`の`Use UART for console`を有効化する。

`Device Drivers -> Console drivers -> Serial Drivers -> nRF UART nrfx drivers -> UART Port 0 Driver type`の`nRF UART 0`を有効化する。

```
ninja
```

いつもどおりリンカスクリプトを修正する。この手順も自動化しないと…。

### hello_worldサンプルの実行

USBシリアル変換ケーブルを接続して、minicomで受信を待ち受ける。

```
sudo minicom --device /dev/ttyUSB0
```

flashに書き込む。

```
nrfutil  pkg generate --hw-version 52 --sd-req=0x00 --application zephyr.hex --application-version 1 pkg.zip
nrfutil dfu usb-serial -pkg pkg.zip -p /dev/ttyACM0
```

minicomに次のログが表示される。

```
***** Booting Zephyr OS v1.13.99-ncs2 *****
Hello World! nrf52840_pca10059
```
