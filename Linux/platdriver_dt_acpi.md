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

`my_platform_driver`はkernelに登録済みのため、次のようなplatform deviceを登録すると、my_platform_driver_probe関数が呼び出されます。

```c
static struct platform_device my_platform_device = {
    .name = "my_platform_driver",
...
};
platform_device_register(&my_platform_device);
```

この`platform_device`には、どのようなデバイスが搭載されているか、というデータを定義することができます。

Linux driverの初期化の流れは、去年のアドベントカレンダーの下記記事に解説があります。
[Linuxのドライバの初期化が呼ばれる流れ](https://qiita.com/rarul/items/308d4eef138b511aa233)

### Platform devices

#### Resources

start/end, nameで定義可能なデータを登録する。

```c
#define IORESOUCE_IO 0x00000100  /* PCI/ISA I/O ports */
```

probe()で扱うのが良いです。
platform_get_resource()で取り出します。

### Platform data

platform_get_platdata()で取り出す。

### DeviceとDriverのマッチング

DeviceとDriverのひも付けの仕組みです。文字列でマッチングします。
例えば、GPIOなら、GPIO用のDriverのリストがあり、Deviceが登録された時点で、マッチングするDriverがあるかどうかを調べる。
Driverがロードされていない場合このタイミングでDriverがロードされる。
その後、probe()が実行される。

device treeだと`compatible`、platform deviceだと、`type`と`name`でマッチングする。

```c
static int platform_match(struct device *dev, struct device_driver *drv) {
    struct platform_device *pdev = to_platform_device(dev);
    struct platform_driver *pdrv = to_platform_driver(drv);

    /* OF style match, first */
    if (of_driver_match_device(dev, drv))
        return 1;
    
    /* Then ACPI style match */
    if (acpi_driver_match_device(dev, drv))
        return 1;
    
    if (pdrv->id_table)
        
}
```

## device tree

platform driverは一度修正すると、kernelを再度ビルドする必要があります。
また、boardごとに新しいdriverが追加され、kernelソースコードを肥大化を招いていました。
乱立するplatform driverの惨状に業を煮やしたLinusにより、device treeが開発されることになりました。

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