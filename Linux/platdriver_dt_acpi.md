# Platform driver, device tree, ACPI

組込みLinuxの醍醐味は、ペリフェラルデバイスのカスタマイズ。
悲喜こもごもの、涙なしでは語れない苦労話がある(はず)。

世の中にはたくさんの組込みLinuxデバイスがあります。
それらのデバイスは、まず、SoCベンダーがリファレンスボードを作ります。TIが作っていたりNXPが作っていたりIntelが作っていたりするわけです。
それらの、リファレンスボードを、家電メーカーや車載機器メーカーが自社製品向けにカスタマイズして、独自の組込みLinuxプラットフォームを構築します。

まず、SoCベンダーが違えば、どのようなペリフェラルデバイスが、何個、搭載されているか、や、同種ペリフェラルデバイスでもメーカーや型番が違ってきます。
同じベンダーでも、デバイスの世代や、性能および消費電力のターゲットが異なれば、異なるデバイスが搭載されます。

ここで問題になるのが、それらの異なるデバイスをどのようにLinux kernelで扱うか、です。
例えば、あるボードでは0x10000000番地にUART16550が接続されていますが、別のボードにおいては、UARTは別アドレスに別デバイスが搭載されているでしょう。

また、SoCベンダーは、メーカーがボードをカスタマイズできるように、GPIOやSPIの口を用意しています。
GPIOやSPIはいろいろな入出力に使えるので、メーカーは自社製品の要件に適合するように、いい感じにカスタマイズするわけですね。

ここでGPIOを例にすると、GPIOはLEDはに使われたり、割り込みに使われたり、何らかのデバイスの入出力に使われたりします。
しかし、Linux kernelとしては、GPIOがLEDなのか割り込み線なのか何らかのデバイスの入出力なのか、は知りようがないわけです。
そのようなauto-negotiation processは存在していません。

そこで、GPIOは(memory mappedの場合)、ここのアドレスにあって、こういうデバイスだから、このdriverで動かしてね、ということをLinux kernelに伝える仕組みの登場です。

1. platform driverのplatform data
2. device tree
3. ACPI(のDSDT)

## Plug and Play

USBやPCI Expressなど、挿せばkernelに認識されるデバイスがあります。
それに対して、non-discoverableなplatform deviceが存在します(USBデバイス自体はPnPでもコントローラはnon-discoverableなplatform deviceです)。
これは、SoCやボードに組み込まれており、抜き差しできないデバイスです。

## Platform driver

### Platform drivers

現在は非推奨の方法です。
やむを得ず使わざるをえない場合もあります。

[omap1 serial](https://github.com/torvalds/linux/blob/master/arch/arm/mach-omap1/serial.c)

platform deviceを利用するためには、次の手順を踏む必要があります。

1. platform driverを登録する
2. deviceとdeviceが利用するリソースを登録する

### probe

deviceが登録された際、呼び出されるdriverの関数です。
deviceがどのdriverと紐付いているか、は`matching`という仕組みがあり、driverの名前でmatchするdriverを探します。

platform driverを登録するためには、init()ないで、platform_driver_register()を使います。

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