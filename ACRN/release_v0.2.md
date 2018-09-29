# ACRN v0.2 (Sep 2018)

ACRN version 0.2がリリースされました。

ACRNは組込み用途に特化した軽量なHypervisorです。OSSとして開発されており、組込みHypervisorのリファレンス実装を提供します。
特徴は次の通りです。
- Small Footprint (ソースコードが少ない)
- Real Time (低レイテンシ、高速起動)
- Built for Embedded IoT (グラフィック、オーディオ機能などの仮想化)
- Adaptability (guest OSに複数のOSをサポート)
- Open Source (BSDライセンス)
- Safety Criticality (Safety critical workloadの高優先度化および隔離)

詳細は[Introduction to Project ACRN](https://projectacrn.github.io/latest/introduction/index.html#introduction)を参照下さい。

ソースコードは[ACRN Hypervisor](https://github.com/projectacrn/acrn-hypervisor)です。

Intelが開発をサポートしており、Linux Foundationのプロジェクトになっています。
リファレンスのSoCはApollo Lakeです。
主なユースケースとして、車載、産業機器を挙げています。

ちなみにACRNはどんぐり (acron)が由来だそうです。ロゴも人の上半身に見えますがどんぐりのポリゴンです。

# Version 0.2 new features

えは追いかけきれていないので、間違っている可能性があります。正しい情報をご存知のかたはご指摘下さい。

## VT-x, VT-d

`パーティションモード`と呼ばれるモードが追加されたようです。
以前までのACRNは、Xenのような構成でした。つまり、管理用の特別なOS (ACRNではService OS、XenではDom0)と、通常のguest OS(ACRNではUser OS、XenではDomU)、という構成でした。
パーティションモードでは、コアやデバイスをパーティショニングして、複数のOSを起動できるようです。
マクロで静的に切り替える使い方のようです。

下のパッチは、起動処理への修正です。
[ff96453](https://github.com/projectacrn/acrn-hypervisor/commit/ff96453) hv: Boot multiple OS for Partitioning mode ACRN

非パーティションモードでは、VM起動時、bootstrap processorには、`prepare_vm0()`でvm0という特別なVMを用意しています。
```c
int prepare_vm0(void)
{
    // 省略
}

int prepare_vm(uint16_t pcpu_id)
{
    int err = 0;

    /* prepare vm0 if pcpu_id is BOOT_CPU_ID */
    if (pcpu_id == BOOT_CPU_ID) {
        err  = prepare_vm0();
    }
    return err;
}
```

一方、パーティションモードでは、起動するVMのbootstrap processorに対して、VMを準備するようになっています。
```c
#ifdef CONFIG_PARTITION_MODE
/* Create vm/vcpu for vm */
int prepare_vm(uint16_t pcpu_id)
{
	int ret = 0;
	uint16_t i;
	struct vm *vm = NULL;
	const struct vm_description *vm_desc = NULL;
	bool is_vm_bsp;

 	vm_desc = pcpu_vm_desc_map[pcpu_id].vm_desc_ptr;
	is_vm_bsp = pcpu_vm_desc_map[pcpu_id].is_bsp;

 	if (is_vm_bsp) {
		ret = create_vm(vm_desc, &vm);
		ASSERT(ret == 0, "VM creation failed!");

 		prepare_vcpu(vm, vm_desc->vm_pcpu_ids[0]);

 		/* Prepare the AP for vm */
		for (i = 1U; i < vm_desc->vm_hw_num_cores; i++)
			prepare_vcpu(vm, vm_desc->vm_pcpu_ids[i]);

 		/* start vm BSP automatically */
		start_vm(vm);

 		pr_acrnlog("Start VM%x", vm_desc->vm_id);
	}

 	return ret;
}
```

コミュニティミーティングの資料を見ると、車載用途を想定しているようです。
よりsafety criticalな処理を、より権限の強いService OSにさせたいわけですが、Service OSはUser OSのデバイスエミュレーションも担うため、負荷が過剰になってしまいます。
そうであるならば、コアやデバイスを、各guest OSに割り当ててしまって、お互いが独立して動作するほうが良い、となります。

VT-xで仮想CPUエミュレーションを、VT-dでデバイスの隔離を実現しているようです。

## PIC/IOAPIC/MSI/MSI-X/PCI/LAPIC

割り込み周りですね。
ACRN Hypervisorは、仮想化されたAPIC-V/EPT/IOAPIC/LAPIC機能をサポートするようです。

## Ethernet

virtio仕様の準仮想化Ethernet機能がサポートされたようです。
MediatorはサービスOSで実行され、物理デバイス（イーサネット、Wi-Fiなど）とUser OSの仮想デバイス間のパケット転送を提供します。

ただし、Ethernet AVB(Audio Video Bridging)はダメみたいです。車載での要望は強いので、今後に期待ですね。

## Storage (eMMC)

不揮発ストレージを、VMがprivateに使用したり、VM間で共有できるようです。

## USB (xDCI)

USB xHCIを仮想化して割り当てることができます。
仮想xHCIコントローラは、複数のUser OSで共有、もしくは、USBポートを、あるVMが専有するように割り当てることもできるようです。
Service OS上でDevice Modelが動作しており、仮想xHCIと物理USBポートのマッピングをしています。最終的に、User OSからのリクエストは、libusbを使ってUSBポートとやり取りします。

xDCIは、特定のUser OSにパススルーできる、とあるので、デバイスコントローラにつながっているUSBポートをまとめてパススルーする、ということだと推測しています。

規格としては、USB 3.0も使えるようです。
UVCはテストしていない、とのことです。

## USB Mediator (xHCI and DRD)

USB Mediatorをサポートします。

詳細がわかっていないですが、準仮想化ができる、ということかと思われます。

## CSME

CSME（Converged Security and Manageability Engine)?
いくつかのfirmwareを総称するもののようです。すみません、よくわからないので、詳しい方、ぜひ教えて下さい。

単一のguest OS (Service OS含む) にCSMEをサポートするようです。

## WiFi

WiFi subsystemのパススルーをサポートします。
なぜかパススルー対象がIVI(In-vehicle-infotainment)となっていますが、guest OSにパススルーで割当可能と考えて良さそうです。

## IPU (MIPI-CS2, HDMI-in)

IPU (Image Processing Unit)をGuest OSにパススルーで割り当てることができるようです。
現状は、Guest OS間での共有ができないです。ロードマップにはGuest OS間での共有が載っているので、今後、実装が期待されます。

## Bluetooth

単一のGuest OSにパススルーできます。

## GPU - Preemption

車載向けの機能です。異常状態での警告など、高優先度のワークロードがある場合に、GPUリソースを優先的に使えるようにします。

## GPU - display surface sharing via Hyper DMA

Hyper DMAを使ったサーフェスシェアリングでは、別Guest OSの個々の`サーフェス`にアクセスできます。
例えば、あるGuest OS上の、あるアプリケーションが描画しているサーフェスのみを共有できます。
車載用途では、ナビ画面を複製するような使い方を想定しているのだと思います。

Hyper DMA Buffer sharingは、LinuxのDMA buffer sharingを拡張した機能です。
おそらく、データをコピーしなくても、DMA領域から複数のGuest OS (driver)がデータにアクセスできる仕組みなのだと思われます。

## S3

電源管理です。
User OS/Service OSともにスリープ状態に入れるようです。
User OSのスリープ管理は、Service OSのDevice Modelが、Service OSのスリープ管理は、ACRN Hypervisorがエミュレーションするようです。

# 最後に

組込み向けのOSS Hypervisorということで、とても面白いです。バックにIntelのサポートがあるので、今後も発展していくことが期待できます。
反面、多様なデバイス技術を把握していないと追いかけるのが難しいです・・・。
一緒に追いかけてくれる人を絶賛某集中です。
