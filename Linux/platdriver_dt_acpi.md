# 組込みLinuxでプラットフォーム上のデバイスを記述する3つの方法()

この記事は[Linux Advent Calendar 2018](https://qiita.com/advent-calendar/2018/linux)の11日目の記事として書かれました。

## はじめに

組込みLinuxの醍醐味は、ペリフェラルデバイスのカスタマイズだと考えています。
新しく取り付けられたデバイスを、ユーザーランドから利用可能にするエンジニアリングは、困難を伴う場合が多くあります。
様々な困難をくぐり抜け、デバイスが動作した瞬間の喜びだけが、組込みLinux屋さんの救いです (なお、個人差があります)。

世の中にはたくさんの組込みLinuxプラットフォームがあります。
それらのプラットフォームは、まずSoCベンダーなどがリファレンスボードを作ります。TIが作っていたりNXPが作っていたりIntelが作っていたりするわけです。
それらの、リファレンスボードを、家電メーカーや車載機器メーカーが自社製品向けにカスタマイズして、独自の組込みLinuxプラットフォームを構築します。

まず、SoCベンダーが違えば、どのようなペリフェラルデバイスが何個搭載されているか、や、同種ペリフェラルデバイスでもメーカーや型番が違ってきます。
同じベンダーでも、デバイスの世代や、性能および消費電力のターゲットが異なれば、異なるデバイスが搭載されます。

ここで問題になるのが、それらの異なるデバイスをどのようにLinux kernelで扱うか、です。
例えば、あるボードでは0x10000000番地にUART16550が接続されていますが、別のボードにおいては、別アドレスに別のUARTデバイスが搭載されているでしょう。

また、SoCベンダーは、メーカーがボードをカスタマイズできるように、GPIOやSPIといったIOを用意しています。
GPIOやSPIはいろいろな入出力に使えるので、メーカーは自社製品の要件に適合するように、いい感じにデバイスを追加したりして、カスタマイズするわけです。

ここでGPIOを例にすると、GPIOはLEDに使われたり、割り込みに使われたり、何らかのデバイスの入出力に使われたりします。
しかし、Linux kernelとしては、あるGPIOがLEDなのか割り込み線なのか何らかのデバイスの入出力なのか、は知りようがないわけです。
そこで、GPIOは(memory mappedの場合)、ここのアドレスにあって、こういうデバイスだから、このdriverで動かしてね、ということをLinux kernelに伝える仕組みが必要になります。

このような目的で利用されている機能は、次の3つがあります。

1. board-specific platform device driver (board file)
2. device tree
3. ACPI(のDSDT)

本記事では、それぞれの方法について、簡単に紹介します。
包括的もしくは体系的な説明ではなく、簡単なデバイス(GPIO)を例に、各方法でどのようにハードウェアを定義するか、見ていきます。

特にACPIについては、情報がなくて苦労しているので、本記事をきっかけに少しでもACPIでプラットフォームデバイスを記述するための情報が出回れば、と願っています。

### 用語定義

本記事内で使用する用語について定義します。

| 用語 | 定義 |
| --- | --- |
| non-discoverable device | kernelが自動で検出できないデバイスです。 |
| platform device | non-discoverableなデバイスで、本記事内で対象とするデバイスです。 |
| platform driver | platform device用のdevice driverです。 |
| GPIO | 汎用入出力ピンです。[wikipedia GPIO](https://ja.wikipedia.org/wiki/GPIO)。ソフトウェアで入出力を切り替え可能で、入力ピンとしても出力ピンとしても利用できます。 |

## 共通事項

3つの方法で共通となることについてまとめます。

いずれの方法にしても、プラットフォーム上のデバイス構成を`何らかの方法で`記述する必要があります。
そのplatform device記述が存在した上で、次のようにデバイスの初期化が実行されます。

1. platform driverをkernelに登録する
2. platform deviceに対し、対応するdriverがbindされる
3. bindされたdriverのprobe関数が呼ばれ、デバイスの初期化を行う

3つの方法では、それぞれ、プラットフォーム上のデバイス構成を記述する方法が異なります。  
また、Linux kernelとの関係性も違ってきます。

### board-specific platform device driver

driverにプラットフォーム上のデバイス構成を記述します。driverとしてLinux kernelに組み込まれます。  
正式にはなんと呼べば良いのか、よくわかりません。
過去のやり方で、現在は非推奨です。

### device tree

device treeと呼ばれる形式でデバイスの構成をツリー状に記述します。`firmware`という扱いで、Linux kernelとは独立したblobを形成します。  
組込みLinuxで用いられる多くのdriverが対応しています。

### ACPI

ACPI Source Language (ASL)でデバイスの構成を記述します。ACPIテーブルの一部(DSDT)として、Linux kernelの外部に置かれます。  
組込みLinuxで用いられるdriverで対応しているものは一部(という印象)です。私もあまり馴染みがないため、間違っている部分があれば、編集リクエスト下さい。

それでは、3つの方法について、個別に見ていきましょう。

## board-specific platform device driver

現在は非推奨の方法です。
やむを得ず使わざるをえない場合もあります。

platform deviceを、kernel driverとして記述します。
これにより、Linux kernelはプラットフォーム上にどのようなデバイスが搭載されているのかを認識することができます。
(kernel内に情報が取り込まれているので、ある意味当然ですが)

下は[MinnowBoard](https://minnowboard.org/)というIntelの組込みプラットフォームのplatform device記述です。  
偶然見つけたのですが、内容がシンプルなため、こちらを例に解説していきます。

[minnowboard.c](https://kernel.googlesource.com/pub/scm/linux/kernel/git/horms/renesas-backport/+/v3.10.28-ltsi-rc1/drivers/platform/x86/minnowboard.c)

platform deviceとして、GPIOで制御できるLEDを登録しています。

### MinnowBoard platform driver

platform deviceを利用するために、次の手順を踏んでいます。

1. board-specific platform device driverをロードする
2. deviceを登録する

なお、Linux driverの初期化の流れは、去年のアドベントカレンダーの下記記事に解説があります。必要に応じて参照下さい。
[Linuxのドライバの初期化が呼ばれる流れ](https://qiita.com/rarul/items/308d4eef138b511aa233)

#### board-specific platform device driverをロードする

Minnowboard用のdriverを実装して、driverをロードします。
お決まりの`module_init`を使うパターンです。
長いので説明に必要な箇所以外は省略しています。
driverロード時に、`minnow_gpio_leds`というplatform deviceを登録しています。

```c:minnowboard.c
static int __init minnow_module_init(void)
{
...
	err = platform_device_register(&minnow_gpio_leds);
...
}
module_init(minnow_module_init);
```

#### deviceを登録する

それでは、platform deviceである`minnow_gpio_leds`の定義を見てみましょう。

```c
static struct platform_device minnow_gpio_leds = {
	.name =	"leds-gpio",
	.id = -1,
	.dev = {
		.platform_data = &minnow_leds_platform_data,
	},
};
```

重要なポイントは、`name`と`dev`メンバです。
この`platform_device`構造体には、どのようなデバイスが搭載されているか、というデータを定義することができます。

```c:include/linux/platform_device.h
struct platform_device {
    const char  *name;
    struct device   dev;
}
```

`name`は、`leds-gpio`という名前のdriverを、このデバイスにbindするために使います。
[Documentation/driver-model/platform.txt](https://github.com/torvalds/linux/blob/master/Documentation/driver-model/platform.txt)

>     * platform_device.name ... which is also used to for driver matching.





platform deviceである`minnow_gpio_leds`がkernelに登録されると、`leds-gpio`という名前のdriverを探します。`leds-gpio` driverは次のように登録されているため、無事、driverが見つかり、そのdriverの`probe`関数を呼び出します。

```c
static struct platform_driver gpio_led_driver = {
	.probe		= gpio_led_probe,
	.driver		= {
		.name	= "leds-gpio",
	},
};
module_platform_driver(gpio_led_driver);
```

probe関数内でデバイスを初期化する際に参照するのが、`dev`メンバに定義したデータになります。
`dev`は、`device`構造体で、いくつかのメンバが存在しますが、ここでは、`platform_data`メンバに`minnow_leds_platform_data`のポインタを代入しています。

```c
	.dev = {
		.platform_data = &minnow_leds_platform_data,
	},
```

`device`構造体の`platform_data`メンバは、次のように定義されています。

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

コメントにある通り、`platform_data`には、デバイス固有のデータを記述します。
さて、`platform_data`の型は`void*`です。これは、好きな構造体を定義して、好きに使ってくれや、ということです。

`leds-gpio` driverでは、`gpio_led_platform_data`を使用します。

```c
static struct gpio_led_platform_data minnow_leds_platform_data = {
	.num_leds = ARRAY_SIZE(minnow_leds),
	.leds = (void *) minnow_leds,
};
```

`gpio_led_platform_data`では、複数のLEDを定義することができます(なので、led`s`-gpioという名前なのだと思われます)。
MinnowBoardでは、次のように2つのLEDを定義しています。

```c
/* leds-gpio platform device structures */
static const struct gpio_led minnow_leds[] = {
	{
      .name = "minnow_led0",
      .gpio = GPIO_LED0,
      .active_low = 0,
	  .retain_state_suspended = 1,
      .default_state = LEDS_GPIO_DEFSTATE_ON,
	  .default_trigger = "heartbeat"
    },
	{
      .name = "minnow_led1",
      .gpio = GPIO_LED1,
      .active_low = 0,
	  .retain_state_suspended = 1,
      .default_state = LEDS_GPIO_DEFSTATE_ON,
	  .default_trigger = "mmc0"
    },
};
```

細かいところは少し置いておいて、`gpio`メンバでGPIOのピン番号を指定しています。
GPIO_LED0/GPIO_LED1は、GPIOの10/11番に割り当てられているようです。

```c:drivers/platform/x86/minnowboard-gpio.h
#define GPIO_LED0 10
#define GPIO_LED1 11
```

このように、MinnowBoard固有のデバイス情報をdriverに定義し、デバイスを登録することで、`leds-gpio` driverの`probe`関数が呼ばれます。
この時、上記で定義したデバイス情報が引数として渡されます。

少し、[leds-gpio.c](https://github.com/torvalds/linux/blob/master/drivers/leds/leds-gpio.c)の実装を確認してみましょう。

```c
static int gpio_led_probe(struct platform_device *pdev)
{
	struct gpio_led_platform_data *pdata = dev_get_platdata(&pdev->dev);
	struct gpio_leds_priv *priv;
...
    // privにメモリ領域を割り当て
		priv->num_leds = pdata->num_leds;
		for (i = 0; i < priv->num_leds; i++) {
			const struct gpio_led *template = &pdata->leds[i];
			struct gpio_led_data *led_dat = &priv->leds[i];
            // GPIOを1つ初期化
...
        }
...
}
```

引数`pdev`は、`platform_device`のポインタ、という形で渡されます。
`pdev->dev`を、自身のデータ構造体である`gpio_led_platform_data`型のポインタにキャストします。
再掲載しますが、下記のようなplatform deviceを定義していましたね。

```c
static struct gpio_led_platform_data minnow_leds_platform_data = {
	.num_leds = ARRAY_SIZE(minnow_leds),
	.leds = (void *) minnow_leds,
};
```

ここから、`num_leds`の回数分、`leds`のデータを使って、デバイスを初期化します。
今回は、2つのLEDデバイスを定義しているので、2つのGPIOが初期化されます。

このように、**ボードに固有のplatform deviceをdriverに定義**して、デバイスを初期化しています。
これが、board-specific platform device driverのplatform device定義方法です。

### マクロを利用したplatform driverの登録 (余談)

ここは、飛ばしても問題ありません。

MinnowBoardでは、`module_init`と`module_exit`を明示的に定義していましたが、マクロを利用する方法もあります。

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

## device tree

platform driverは一度修正すると、kernelを再度ビルドする必要があります。
また、boardごとに新しいdriverが追加され、kernelソースコードを肥大化を招いていました。
[arch/arm](https://github.com/torvalds/linux/tree/master/arch/arm)を見ると、`mach-`や`plat-`から始まるディレクトリが大量に存在します。これらが、board-specific platform device driverです。

特定ボードの設定は、kernelに含まれるべきではない、という思想から、組込みボードでは**device tree** が広く使われるようになりました。

device treeは、プラットフォーム上のハードウェアは、木構造に似た形式で記述します。
各デバイスは、ノードと呼ばれ、必要なパラメータやデータは、`property`という形で表現します。
device treeを修正した場合、device treeを再コンパイルするだけで済み、kernelは再ビルドする必要がありません。
現在、組込みLinuxで利用されるペリフェラルデバイス用driverの多くは、device treeに対応しています。

device treeについては、下記記事にとても良くまとめられています。

[Device Tree についてのまとめ](https://qiita.com/koara-local/items/ed99a7b96a0ca252fc4e)

現在もまとまった日本語情報はないと思います。本記事では包括的、または、体系的な説明はしません(できません)。GPIOを例に、それがどのような意味なのか、を解説していきます。

英語では、[Linux Device Drivers Development](https://www.amazon.co.jp/Linux-Device-Drivers-Development-Madieu/dp/1785280007)に解説があります。
私はこの本を購入してだいぶ救われました(なお、個人差があります)。
本記事を書く上でもかなり参考にしています。(誤植は多いですが)device treeに対応したdevice driverについて解説のある良い本ですので、device treeやdevice driver開発で悩まれている方は、購入を検討してみて下さい。

そうは言っても、最終的には、各driverのdevice tree bindingsを参照しながら、必要ならdriver自体のコードを解析しながら利用することになります。
bindings一覧は、`Documentation/devicetree/bingins`にまとめられています。
各ベンダー固有のforkにしか存在しないbindingsドキュメントもあるため、注意が必要です。

[device tree bindings](https://github.com/OpenChannelSSD/linux/tree/master/Documentation/devicetree/bindings)を覗いてみると、`board`, `gpio`, `leds`, `serial`, `spi`などデバイス種別ごとのディレクトリが存在します。
試しに、`leds/leds-gpio.txt`を見てみます。これは、上記MinnowBoardで解説したGPIOで制御可能なLEDデバイスのノードを記述するための説明です。
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

`compatible`は、deviceとdriverをbindするためのpropertyです。`leds-gpio` driverを見るとわかりやすいです。
`leds-gpio` driverは`platform_driver`として登録され、compatibleに`gpio-leds`という文字列が指定されたデバイスと紐付けされます。
(`platform_device`構造体を使う時と異なり、name (driverの名前)ではなく、compatibleが一致するdriverとbindされます)

[leds-gpio.c](https://github.com/OpenChannelSSD/linux/blob/master/drivers/leds/leds-gpio.c)

```c:/drivers/leds/leds-gpio.c
static const struct of_device_id of_gpio_leds_match[] = {
	{ .compatible = "gpio-leds", },
	{},
};
MODULE_DEVICE_TABLE(of, of_gpio_leds_match);
```

上記のデバイスツリーの例では、`run-control`ノードに、redとgreenという2つのLEDを持つ`gpio-leds` compatibleなデバイスを定義しています。
結果として、2つのGPIOは、`leds-gpio` driverで制御されます。

driverがbindされたあとは、board-specific platform device driverと同様となります。

## ACPI DSDT

http://www.uefi.org/sites/default/files/resources/ACPI_6_2.pdf

ACPI自体は規格化されているため、仕様書は存在するわけです
しかしながら、組込み系の周辺ペリフェラルを記述するための例は乏しい状況です。

Yoctoのレイヤとして、下記のmeta-acpiが公開されています。

[meta-acpi](https://github.com/westeri/meta-acpi)

こちらにあるsampleを見ながら、理解を進めていきましょう。
`recipes-bsp/acip-tables/samples/minnowboard`に、`leds.asl`と`buttons.asl`というファイルがあります。
`leds.asl`がGPIOを使ったLEDのようなので、これを見ていきましょう。

ちなみに、筆者自身もACPIでのdevice定義があまりわかっていません。解説されていない部分は、わかってないんだなぁ、と思ってスルーするか、むしろコメントなどで教えて下さい。

```
DefinitionBlock ("leds.aml", "SSDT", 5, "", "LEDS", 1)
{
    External (_SB_.PCI0.LPC, DeviceObj)

    Scope (\_SB.PCI0.LPC)
    {
        Device (LEDS)
        {
            Name (_HID, "PRP0001")
            Name (_DDN, "GPIO LEDs device")

            Name (_CRS, ResourceTemplate () {
                GpioIo (
                    Exclusive,                  // Not shared
                    PullNone,                   // No need for pulls
                    0,                          // Debounce timeout
                    0,                          // Drive strength
                    IoRestrictionOutputOnly,    // Only used as output
                    "\\_SB.PCI0.LPC",           // GPIO controller
                    0)                          // Must be 0
                {
                    10,                         // E6XX_GPIO_SUS5
                    11,                         // E6XX_GPIO_SUS6
                }
            })

            Name (_DSD, Package () {
                ToUUID("daffd814-6eba-4d8c-8a91-bc9bbf4aa301"),
                Package () {
                    Package () {"compatible", "gpio-leds"},
                },
                ToUUID("dbb8e3e6-5886-4ba6-8795-1319f52a966b"),
                Package () {
                    Package () {"led-0", "LED0"},
                    Package () {"led-1", "LED1"},
                }
            })

            /*
             * For more information about these bindings see:
             * Documentation/devicetree/bindings/leds/leds-gpio.txt and
             * Documentation/acpi/gpio-properties.txt.
             */
            Name (LED0, Package () {
                ToUUID("daffd814-6eba-4d8c-8a91-bc9bbf4aa301"),
                Package () {
                    Package () {"label", "heartbeat"},
                    Package () {"gpios", Package () {^LEDS, 0, 0, 0}},
                    Package () {"linux,default-state", "off"},
                    Package () {"linux,default-trigger", "heartbeat"},
                    Package () {"linux,retain-state-suspended", 1},
                }
            })

            Name (LED1, Package () {
                ToUUID("daffd814-6eba-4d8c-8a91-bc9bbf4aa301"),
                Package () {
                    Package () {"label", "sd-card"},
                    Package () {"gpios", Package () {^LEDS, 0, 1, 0}},
                    Package () {"linux,default-state", "off"},
                    Package () {"linux,default-trigger", "mmc0"},
                    Package () {"linux,retain-state-suspended", 1},
                }
            })
        }
    }
}
```

device treeと比較すると、随分大げさに見えます。
これは _心_ 骨が折れそうですが、1つずつ見ていきましょう。

device treeと共通する点としては、deviceをツリー上に表現する、ということです。

```
    Scope (\_SB.PCI0.LPC)
    {
        Device (LEDS)
        {
```

`_SB`はSystem Busを意味するようです。最も上位のバスだと認識しておけば良さそうです。
`PCI0`はそのまま、PCIバスです。
`LPC`は[Low Pin Count](https://ja.wikipedia.org/wiki/Low_Pin_Count)でしょうか？自信がありません。
[Accessing Intel ICH/PCH GPIOs](https://lab.whitequark.org/notes/2017-11-08/accessing-intel-ich-pch-gpios/)を見ると、歴史的に、Intelは、LPC下にGPIOコントローラを置いているようです？Intelわからん。

いずれにしても、LPCの下に、`LEDS`というラベルのデバイスを定義しています。

次に行きましょう。

```
            Name (_HID, "PRP0001")
            Name (_DDN, "GPIO LEDs device")
```

_HIDは、Hardware IDです。
どうやら、`RPR0001`は特殊なIDであり、device tree compatibleなdevice識別子とのことです。
https://patchwork.kernel.org/patch/6593351/
結局、device treeの仕組みに頼るのかい！と思いながら先に進みます。

_DDNは、Dos Device Nameだそうです。
さらに読み進めます。

`GpioIo()`は、`Operator`と呼ばれるもので、IO Resource Descriptor Macro、だそうです。
マクロでdeviceのpropertyを設定している、という理解で良さそうです。

```:leds.asl
            Name (_CRS, ResourceTemplate () {
                GpioIo (
                    Exclusive,                  // Not shared
                    PullNone,                   // No need for pulls
                    0,                          // Debounce timeout
                    0,                          // Drive strength
                    IoRestrictionOutputOnly,    // Only used as output
                    "\\_SB.PCI0.LPC",           // GPIO controller
                    0)                          // Must be 0
                {
                    10,                         // E6XX_GPIO_SUS5
                    11,                         // E6XX_GPIO_SUS6
                }
            })
```

GpioIo()マクロへの引数は、次のように、仕様書に掲載されています。

```
GpioIo (Shared, PinConfig, DebounceTimeout, DriveStrength, IORestriction, ResourceSource,
ResourceSourceIndex, ResourceUsage, DescriptorName, VendorData) {PinList}
```

上の記述と照らし合わせます。

- `PinList`から、対象となるGPIOピンは10/11番のGPIO
- 各ピンは専有される
- pull up/pull downはしない
- Outputポートとして利用する

となります。これらのポート設定は、GPIOコントローラによってなされるので、設定を依頼するGPIOコントローラが指定されているのだと考えられます。

GPIOピン、10/11にLEDのが接続されているようです。
そういえば、Minnowboardのplatform device driverでも、ヘッダファイルで次のようにマクロが定義されていました。

```c:minnowboard-gpio.h
#define GPIO_LED0 10
#define GPIO_LED1 11
```

```
            Name (_DSD, Package () {
                ToUUID("daffd814-6eba-4d8c-8a91-bc9bbf4aa301"),
                Package () {
                    Package () {"compatible", "gpio-leds"},
                },
                ToUUID("dbb8e3e6-5886-4ba6-8795-1319f52a966b"),
                Package () {
                    Package () {"led-0", "LED0"},
                    Package () {"led-1", "LED1"},
                }
            })

            /*
             * For more information about these bindings see:
             * Documentation/devicetree/bindings/leds/leds-gpio.txt and
             * Documentation/acpi/gpio-properties.txt.
             */
            Name (LED0, Package () {
                ToUUID("daffd814-6eba-4d8c-8a91-bc9bbf4aa301"),
                Package () {
                    Package () {"label", "heartbeat"},
                    Package () {"gpios", Package () {^LEDS, 0, 0, 0}},
                    Package () {"linux,default-state", "off"},
                    Package () {"linux,default-trigger", "heartbeat"},
                    Package () {"linux,retain-state-suspended", 1},
                }
            })

            Name (LED1, Package () {
                ToUUID("daffd814-6eba-4d8c-8a91-bc9bbf4aa301"),
                Package () {
                    Package () {"label", "sd-card"},
                    Package () {"gpios", Package () {^LEDS, 0, 1, 0}},
                    Package () {"linux,default-state", "off"},
                    Package () {"linux,default-trigger", "mmc0"},
                    Package () {"linux,retain-state-suspended", 1},
                }
            })
```

続いて、`_DSD`です。kernelにドキュメントがあるので、少し見てみます。

[DSD-properties-rules](https://github.com/torvalds/linux/blob/master/Documentation/acpi/DSD-properties-rules.txt)

> The _DSD (Device Specific Data) configuration object, introduced in ACPI 5.1, allows any type of device configuration data to be provided via the ACPI namespace.

おそらく、ACPI規格で網羅しきれない、deviceのpropertyを記述するためのものだと推測しています。
DSDを使うことで、device treeと同様のproperty記述ができるようになる、ということでしょうか。

各DSDは、識別子としてUUIDを持つ必要があるため、各要素にUUIDが設定されています。
下の記述で、device treeと同様に、`compatible`を設定しています。

```
                Package () {
                    Package () {"compatible", "gpio-leds"},
                },
```

gpio-ledsのデバイスを2つ定義しています。

```
                    Package () {"led-0", "LED0"},
                    Package () {"led-1", "LED1"},
```

それぞれの詳細なpropertyは、その下で記述されています。ここでは、LED0のみ掲載します。

```
            Name (LED0, Package () {
                ToUUID("daffd814-6eba-4d8c-8a91-bc9bbf4aa301"),
                Package () {
                    Package () {"label", "heartbeat"},
                    Package () {"gpios", Package () {^LEDS, 0, 0, 0}},
                    Package () {"linux,default-state", "off"},
                    Package () {"linux,default-trigger", "heartbeat"},
                    Package () {"linux,retain-state-suspended", 1},
                }
            })
```

[leds-gpio.txt](https://github.com/torvalds/linux/blob/master/Documentation/devicetree/bindings/leds/leds-gpio.txt)を再び参照すると、各propertyがleds-gpioのpropertyとして存在していることがわかります。

```
LED sub-node properties:
- gpios :  Should specify the LED's GPIO, see "gpios property" in
  Documentation/devicetree/bindings/gpio/gpio.txt.  Active low LEDs should be
  indicated using flags in the GPIO specifier.
- label :  (optional)
  see Documentation/devicetree/bindings/leds/common.txt
- linux,default-trigger :  (optional)
  see Documentation/devicetree/bindings/leds/common.txt
- default-state:  (optional) The initial state of the LED.
  see Documentation/devicetree/bindings/leds/common.txt
- retain-state-suspended: (optional) The suspend state can be retained.Such
  as charge-led gpio.
```

ただ、`gpios`については、ぱっと見で、自明ではありません。

```
                    Package () {"gpios", Package () {^LEDS, 0, 0, 0}},
```

device treeの例では、下のようになっていため、これと同様の要素になっているはずです。

```
		gpios = <&mpc8572 6 GPIO_ACTIVE_HIGH>;
```

[gpio.txt](https://github.com/torvalds/linux/blob/master/Documentation/devicetree/bindings/gpio/gpio.txt)によると、gpiosの1つ目の項目はGPIOコントローラ、2つ目はGPIOコントローラ内のローカルオフセット、3つ目はピンのフラグになっています。

> In the above example, &gpio1 uses 2 cells to specify a gpio. The first cell is a local offset to the GPIO line and the second cell represent consumer flags, such as if the consumer desire the line to be active low (inverted) or open drain.

GPIOピンのフラグは、次の通り定義されています。

```c:include/dt-bindings/gpio/gpio.h
/* Bit 0 express polarity */
#define GPIO_ACTIVE_HIGH 0
#define GPIO_ACTIVE_LOW 1

/* Bit 1 express single-endedness */
#define GPIO_PUSH_PULL 0
#define GPIO_SINGLE_ENDED 2

/*
 * Open Drain/Collector is the combination of single-ended active low,
 * Open Source/Emitter is the combination of single-ended active high.
 */
#define GPIO_OPEN_DRAIN (GPIO_SINGLE_ENDED | GPIO_ACTIVE_LOW)
#define GPIO_OPEN_SOURCE (GPIO_SINGLE_ENDED | GPIO_ACTIVE_HIGH)
```

これに沿って解釈してみましょう。

```
                    Package () {"gpios", Package () {^LEDS, 0, 0, 0}},
```

はい、なんか項目が1つ多いですね。
一番後ろの`0`は、GPIOピンのフラグでしょう。`0`ということは、`GPIO_ACTIVE_HIGH`です。
残りの部分ですが、GPIOコントローラと、ローカルオフセットである`10`を意味するはずです。
どうにかこじつけてこの対応を説明しないと記事の収まりが悪いです。


```
        Device (LEDS)
        {
...
            Name (_CRS, ResourceTemplate () {
                GpioIo (
...
                    "\\_SB.PCI0.LPC", // ←GPIOコントローラ
                    0)
                {
                    10, // ←ローカルオフセット
                    11,
                }
```

答えは、[gpio-properties.txt](https://github.com/torvalds/linux/blob/master/Documentation/acpi/gpio-properties.txt)にあります。

> The format of the supported GPIO property is:
>
>  Package () { "name", Package () { ref, index, pin, active_low }}
>
>  ref - The device that has _CRS containing GpioIo()/GpioInt() resources,
>        typically this is the device itself (BTH in our case).
>  index - Index of the GpioIo()/GpioInt() resource in _CRS starting from zero.
>  pin - Pin in the GpioIo()/GpioInt() resource. Typically this is zero.
>  active_low - If 1 the GPIO is marked as active_low.

つまり、デバイスの定義である`LEDS`からの相対位置で、ピン番号を指定しています。
`Package () {^LEDS, 0, 0, 0}`が意味するところは、次のようになります。

```
        Device (LEDS) // ←`^LEDS`でここへの参照
        {
...
            Name (_CRS, ResourceTemplate () {
                GpioIo (  // ←_CRSの`0`番目
...
                    "\\_SB.PCI0.LPC",
                    0)
                {
                    10, // ←_CRS0番の、`0`番目のピン
                    11,
                }
```

_わかるか！こんなもん！_ 一通り理解できたので、すっきりしましたね！

ACPIでプラットフォームデバイスを定義するにあたり、辛いのは、具体例の少なさです。
私のような、具体例を足がかりに、体系へと学習を進めるタイプの人間にとって、具体例が少ないのは危機的状況です。
というわけで、少しでも情報が出まわるきっかけになれば、と思いACPIの具体例を取りあげて解説をしてみました。

## おまけ

### device treeとACPIの使い分け

device driver構造体は、device tree、ACPI両方に対応しています。
多く(?)の(組込み向け)driverは、device treeのみ対応していますが、device tree、ACPI両方に対応しているdriverも存在しています。そのようなdriverは、両方のmatch_tableを定義してます。

[spi-pxa2xx](https://github.com/torvalds/linux/blob/master/drivers/spi/spi-pxa2xx.c)

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

GPIOくらい一般的なdriverだと、device treeとACPI両方に対応しています。
例えば、下のような実装になっており、device treeを使う場合と、ACPIを使う場合とを、マクロレベルでラップしてくれています。

```c:drivers/gpio/gpiolib.c
struct gpio_desc *__must_check gpiod_get_index(struct device *dev,
					       const char *con_id,
					       unsigned int idx,
					       enum gpiod_flags flags)
{
...
		/* Using device tree? */
		if (IS_ENABLED(CONFIG_OF) && dev->of_node) {
			dev_dbg(dev, "using device tree for GPIO lookup\n");
			desc = of_find_gpio(dev, con_id, idx, &lookupflags);
		} else if (ACPI_COMPANION(dev)) {
			dev_dbg(dev, "using ACPI for GPIO lookup\n");
			desc = acpi_find_gpio(dev, con_id, idx, &flags, &lookupflags);
		}
...
```

## 参考

- [Linux Device Drivers Development](https://www.amazon.co.jp/Linux-Device-Drivers-Development-Madieu/dp/1785280007)
- [Linuxのドライバの初期化が呼ばれる流れ](https://qiita.com/rarul/items/308d4eef138b511aa233)