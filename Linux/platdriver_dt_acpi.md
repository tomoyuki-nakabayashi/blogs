# Platform driver, device tree, ACPI

この記事は[Linux Advent Calendar 2018](https://qiita.com/advent-calendar/2018/linux)の11日目の記事として書かれました。

## はじめに

組込みLinuxの醍醐味は、ペリフェラルデバイスのカスタマイズだと考えています。
新しく取り付けられたデバイスを、ユーザーランドから利用可能にするエンジニアリングは、困難を伴う場合が多くあります。
様々な困難をくぐり抜け、デバイスが動作した瞬間の喜びだけが、組込みLinux屋さんの救いです (なお、個人差)。

世の中にはたくさんの組込みLinuxデバイスがあります。
それらのデバイスは、まず、SoCベンダーがリファレンスボードを作ります。TIが作っていたりNXPが作っていたりIntelが作っていたりするわけです。
それらの、リファレンスボードを、家電メーカーや車載機器メーカーが自社製品向けにカスタマイズして、独自の組込みLinuxプラットフォームを構築します。

まず、SoCベンダーが違えば、どのようなペリフェラルデバイスが何個搭載されているか、や、同種ペリフェラルデバイスでもメーカーや型番が違ってきます。
同じベンダーでも、デバイスの世代や、性能および消費電力のターゲットが異なれば、異なるデバイスが搭載されます。

ここで問題になるのが、それらの異なるデバイスをどのようにLinux kernelで扱うか、です。
例えば、あるボードでは0x10000000番地にUART16550が接続されていますが、別のボードにおいては、UARTは別アドレスに別デバイスが搭載されているでしょう。

また、SoCベンダーは、メーカーがボードをカスタマイズできるように、GPIOやSPIの口を用意しています。
GPIOやSPIはいろいろな入出力に使えるので、メーカーは自社製品の要件に適合するように、いい感じにデバイスを追加したりして、カスタマイズするわけです。

ここでGPIOを例にすると、GPIOはLEDに使われたり、割り込みに使われたり、何らかのデバイスの入出力に使われたりします。
しかし、Linux kernelとしては、あるGPIOがLEDなのか割り込み線なのか何らかのデバイスの入出力なのか、は知りようがないわけです。
そこで、GPIOは(memory mappedの場合)、ここのアドレスにあって、こういうデバイスだから、このdriverで動かしてね、ということをLinux kernelに伝える仕組みが必要になります。

このような目的で利用されている機能は、次の3つがあります。

1. platform driver/platform device
2. device tree
3. ACPI(のDSDT)

本記事では、それぞれの方法について、簡単に紹介します。
包括的もしくは体系的な説明ではなく、簡単なデバイス(GPIO)を例に、各方法でどのようにハードウェアを定義するか、見ていきます。

## 余談 Plug and Play

USBやPCI Expressなど、挿せばLinux kernelに認識されるデバイスがあります。
それに対して、non-discoverableなプラットフォーム組込みのデバイスが存在します。
例えば、USBメモリなどのUSB機器はPnPでも、USBコントローラはnon-discoverableなデバイスです。
本記事では、non-discoverableなデバイスを組み込む方法のみを説明します。

## platform driver & platform device

現在は非推奨の方法です。
やむを得ず使わざるをえない場合もあります。

大雑把にいうと、platform driverをLinux kernel起動時にロードさせて、プラットフォーム上に存在するplatform deviceを登録します。
これにより、Linux kernelはプラットフォーム上にどのようなデバイスが搭載されているのかを認識することができます。

[x86 platform](https://elinux.org/images/e/ec/Hart-x86-platform.pdf)
[minnowboard.c](https://kernel.googlesource.com/pub/scm/linux/kernel/git/horms/renesas-backport/+/v3.10.28-ltsi-rc1/drivers/platform/x86/minnowboard.c)

### platform drivers

[omap1 serial](https://github.com/torvalds/linux/blob/master/arch/arm/mach-omap1/serial.c)

platform deviceを利用するためには、次の手順を踏む必要があります。

1. platform driverを登録する
2. deviceを登録する

まず、`platform driverを登録する`では、独自のplatform driverを定義します。
`my_platform_driver`としておきます。

`deviceを登録する`では、登録したdriverに対応する名前で、deviceを登録します。。

### platform driverの登録

`module_platform_driver()`マクロを使います。
特定の用途に特化したマクロも定義されています。
`module_spi_driver(struct spi_driver)`や`module_i2c_driver(struct i2c_driver)`などが該当します。

これらのマクロは、platform driverを登録/登録解除するinit/exit関数を定義します。
マクロは次のように実装されています。

```c:device.h
#define module_driver(__driver, __register, __unregister, ...) \
static int __init __driver##_init(void) \
{ \
	return __register(&(__driver) , ##__VA_ARGS__); \
} \
module_init(__driver##_init); \
static void __exit __driver##_exit(void) \
{ \
	__unregister(&(__driver) , ##__VA_ARGS__); \
} \
module_exit(__driver##_exit);
```

例えば、次のように呼び出すと、

```c
static struct platform_driver my_platform_driver = {
...
}

module_platform_driver(my_platform_driver);
```

次のように展開されます。

```c
static int __init my_platform_driver_init(void)
{
    return platform_driver_register(&(my_platform_driver) , ##__VA_ARGS__);
}
module_init(my_platform_driver_init);

static void __exit my_platform_driver_exit(void)
{
    platform_driver_unregister(&(my_platform_driver) , ##__VA_ARGS__);
}
module_exit(my_platform_driver_exit);
```

driverを登録すると、対応するdeviceが登録されたときに、kernelが登録されているdriverの中から対応するdriverを探して、そのdriverのprobe関数が呼び出します。
`platform_driver`を次のように定義しておくと、`my_platform_driver`と関連づいているdeviceが登録された時に、`my_platform_driver_probe`関数が呼び出されます。

```c
static struct platform_driver my_platform_driver = {
    .probe = my_platform_driver_probe,
    .remove = my_platform_driver_remove,
    .driver = {
        .name = KBUILD_MODNAME,
        .owner = THIS_MODULE,
    },
};
```

Linux driverの初期化の流れは、去年のアドベントカレンダーの下記記事に解説があります。
[Linuxのドライバの初期化が呼ばれる流れ](https://qiita.com/rarul/items/308d4eef138b511aa233)

### Platform deviceの登録

#### platform device

`my_platform_driver`はkernelに登録済みのため、次のようなplatform deviceを登録すると、my_platform_driver_probe関数が呼び出されます。

```c
static struct platform_device my_platform_device = {
    .name = "my_platform_driver",
...
};
platform_device_register(&my_platform_device);
```

この`platform_device`には、どのようなデバイスが搭載されているか、というデータを定義することができます。

```c:include/linux/platform_device.h
struct platform_device {
    const char  *name;
    struct device   dev;
}
```

ここで重要なのが、`struct device`の`platform_data`メンバです。

```c:include/linux/device.h
/**
 * struct device - The basic device structure
...
 * @platform_data: Platform data specific to the device.
 *     Example: For devices on custom boards, as typical of embedded
 *     and SOC based hardware, Linux often uses platform_data to point
 *     to board-specific structures describing devices and how they
 *     are wired.  That can include what ports are available, chip
 *     variants, which GPIO pins act in what additional roles, and so
 *     on.  This shrinks the "Board Support Packages" (BSPs) and
 *     minimizes board-specific #ifdefs in drivers.
...
struct device {
    struct device       *parent;
...
    void        *platform_data; /* Platform specific data, device
                       core doesn't touch it */
...
};
```

コメントにある通り、`platform_data`には、カスタムボード固有のデバイス構造を記述します。
例えば、どのGPIOが、なんの役割を担っているか、ということを指定していきます。

さて、`platform_data`の型は`void*`です。これは、好きな構造体を定義して、好きに使って良い、ということです。
`platform_device`と`platform_driver`は強い結合があり、`platform_driver`は、この`platform_data`に定義されたデバイス構造データを利用して、デバイスを初期化していきます。

#### Specific platform device

全部が全部独自でデバイス構造を用意するわけではなく、一般的なデバイスについては、データ構造が用意されています。
例えば、SPIデバイスについては、`spi_board_info`という構造体が用意されています。この構造体には、多くのSPIデバイスが共通して持つパラメータが定義されています。
例えば、次のように使います。

```c
static struct spi_board_info spidev_on_my_platform[] = {
    {
        .modalias = "spidev",
        .max_speed_hz = 2000000,
        .bus_num = 1,
        .chip_select = 0,
        .mode = SPI_MODE_0,
    },
};

spi_register_board_info(&spidev_on_my_platform, 1);
```

この`spidev_on_my_platform`のような実体を定義し、`spi_register_board_info`でdeviceを登録することで、対応するSPIデバイスのdriverと紐付けることができます。
上記の例では、`.modalias`で指定されている`spidev`というdriverと紐付けがされます。
`spidev_on_my_platform`が登録された時点で、kernelは、spidev driverのprobe関数を呼び出します。
probe関数呼び出し時、kernelは`spi_device`というデータを作成し、spidev driver probeの引数として渡します。
これは、`.max_speed_hz`などのメンバから作成したSPIデバイスのパラメータ一式となります。
spidev driverのprobe関数では、spi_deviceのデータを参照して、デバイスの初期化を実行します。

このような共通のデータ構造を使って、SPIデバイスは、各driverのprobe関数で、デバイスを初期化します。
しかし、多くのSPIデバイスでは、driverのprobe内で、用意されたメンバ以外のデータが必要になります。
そのために、`spi_board_info`にも、`platform_data`メンバが用意されています。

```c:include/linux/spi/spi.h
/**
 * struct spi_board_info - board-specific template for a SPI device
 * @modalias: Initializes spi_device.modalias; identifies the driver.
 * @platform_data: Initializes spi_device.platform_data; the particular
 *	data stored there is driver-specific.
...
 */
struct spi_board_info {
	char		modalias[SPI_NAME_SIZE];
	const void	*platform_data;
...
}
```

共通のパラメータだけでは不足する場合には、独自のデータ構造体を定義し、driverで取得して利用します。

まとめると、

1. platform driverを登録する
2. platform deviceを登録する
3. 登録されたplatform deviceとplatform driverのマッチングを行い、対応するdriverのprobeを呼び出す
4. probe内では、platform deviceで定義されたパラメータデータを利用して、デバイスの初期化を実施する

## device tree

platform driverは一度修正すると、kernelを再度ビルドする必要があります。
また、boardごとに新しいdriverが追加され、kernelソースコードを肥大化を招いていました。
特定ボードの設定は、kernelに含まれるべきではない、という思想から、**device tree** が開発されることになりました。

device treeは、プラットフォーム上のハードウェアは、木構造に似た形式で記述します。
各デバイスは、ノードと呼ばれ、必要なパラメータやデータは、`property`という形で表現します。
device treeを修正した場合、device treeを再コンパイルするだけで済み、kernelは再ビルドする必要がありません。

device treeについては、下記記事にとても良くまとめられています。

[Device Tree についてのまとめ](https://qiita.com/koara-local/items/ed99a7b96a0ca252fc4e)

現在もまとまった日本語情報はないと思います。本記事では包括的、または、体系的な説明はしません(できません)。例をいくつか挙げ、それがどのような意味なのか、を解説していきます。

英語では、[Linux Device Drivers Development](https://www.amazon.co.jp/Linux-Device-Drivers-Development-Madieu/dp/1785280007)に解説があります。
私はこの本を購入してだいぶ救われました(なお、個人差があります)。
本記事を書く上でもかなり参考にしています。(誤植は多いですが)device treeに対応したdevice driverについて解説のある良い本ですので、device treeやdevice driver開発で悩まれている方は、購入を検討してみて下さい。

そうは言っても、最終的には、各driverのdevice tree bindingsを参照しながら、必要ならdriver自体のコードを解析しながら利用することになります。
bindings一覧は、`Documentation/devicetree/bingins`にまとめられています。
各ベンダー固有のforkにしか存在しないbindingsドキュメントもあるため、注意が必要です。

[device tree bindings](https://github.com/OpenChannelSSD/linux/tree/master/Documentation/devicetree/bindings)を覗いてみると、`board`, `gpio`, `leds`, `serial`, `spi`などデバイス種別ごとのディレクトリが存在します。
試しに、`leds/leds-gpio.txt`を見てみます。これは、GPIOで制御可能なLEDデバイスのノードを記述するための説明です。
propertyには、必須(shold/required)なものと、optionalなものがあります。下記の例では、`compatible`と`gpios`が必須のpropertyで、`default-state`はoptionalなpropertyです。


```:leds/leds-gpio.txt抜粋
LEDs connected to GPIO lines

Required properties:
- compatible : should be "gpio-leds".

LED sub-node properties:
- gpios :  Should specify the LED's GPIO, see "gpios property" in
  Documentation/devicetree/bindings/gpio/gpio.txt.  Active low LEDs should be
  indicated using flags in the GPIO specifier.

- default-state:  (optional) The initial state of the LED.
  see Documentation/devicetree/bindings/leds/common.txt
  
Examples:

run-control {
	compatible = "gpio-leds";
	red {
		gpios = <&mpc8572 6 GPIO_ACTIVE_HIGH>;
		default-state = "off";
	};
	green {
		gpios = <&mpc8572 7 GPIO_ACTIVE_HIGH>;
		default-state = "on";
	};
};
```

`compatible`は、deviceとdriverを紐付けるためのpropertyです。`leds-gpio` driverを見るとわかりやすいです。
`leds-gpio` driverは`platform_driver`として登録され、compatibleに`gpio-leds`という文字列が指定されたデバイスと紐付けされます。

[leds-gpio.c](https://github.com/OpenChannelSSD/linux/blob/master/drivers/leds/leds-gpio.c)

```c:/drivers/leds/leds-gpio.c
static const struct of_device_id of_gpio_leds_match[] = {
	{ .compatible = "gpio-leds", },
	{},
};
MODULE_DEVICE_TABLE(of, of_gpio_leds_match);

static struct platform_driver gpio_led_driver = {
	.probe		= gpio_led_probe,
	.shutdown	= gpio_led_shutdown,
	.driver		= {
		.name	= "leds-gpio",
		.of_match_table = of_gpio_leds_match,
	},
};
module_platform_driver(gpio_led_driver);
```

上記のデバイスツリーの例では、`run-control`ノードに、redとgreenという2つの`gpio-leds` compatibleなデバイスを定義しています。
結果として、2つのGPIOは、`leds-gpio` driverで制御されます。

https://dri.freedesktop.org/docs/drm/driver-api/gpio/board.html

Open Firmware (OF) matching.

## ACPI DSDT

例えば、device treeではなくACPIを利用するLinux kernelのコンフィギュレーションは次のようになる。

```
CONFIG_GPIOLIB=y
CONFIG_GPIO_ACPI=y
```

https://www.kernel.org/doc/Documentation/acpi/enumeration.txt

> In ACPI, the device identification object called _CID (Compatible ID) is used to
list the IDs of devices the given one is compatible with, but those IDs must
belong to one of the namespaces prescribed by the ACPI specification (see
Section 6.1.2 of ACPI 6.0 for details) and the DT namespace is not one of them.

```c:device.h
struct device_driver {
	const char		*name;
	struct bus_type		*bus;
...
	const struct of_device_id	*of_match_table;
	const struct acpi_device_id	*acpi_match_table;
...
	const struct dev_pm_ops *pm;
...
};
```

https://wiki.osdev.org/ACPI
https://wiki.osdev.org/AML
https://wiki.osdev.org/DSDT

https://github.com/westeri/meta-acpi/

## 参考

- [Linux Device Drivers Development](https://www.amazon.co.jp/Linux-Device-Drivers-Development-Madieu/dp/1785280007)
- [Linuxのドライバの初期化が呼ばれる流れ](https://qiita.com/rarul/items/308d4eef138b511aa233)