# Zephyr build process

Zephyrのビルドプロセスを解析する。

## nRF52840 linker script

Zephyrのデフォルトでビルドすると、下のリンカスクリプトになっている。

```
$ head -n 10 zephyr/linker.cmd
 OUTPUT_FORMAT("elf32-littlearm")
_region_min_align = 32;
MEMORY
    {
    FLASH (rx) : ORIGIN = (0x0 + 0), LENGTH = (1024*1K - 0)
    SRAM (wx) : ORIGIN = 0x20000000, LENGTH = (256 * 1K)
    IDT_LIST (wx) : ORIGIN = (0x20000000 + (256 * 1K)), LENGTH = 2K
    }
ENTRY("__start")
SECTIONS
```

nRF52840 donle (pca10059)のドキュメントを見ると、先頭4KBはMBRで書き換えしない、と書いてある。
実際にこのリンカスクリプトを使ったバイナリは動作せず、FLASHのORIGINを4KB後ろにずらしてあげると動くようになる。
今までは手動でリンカスクリプトを修正していたので、そうしなくてよいようにする。

- ビルド時に自分が修正したリンカスクリプトを使うように修正する。

### nRF SDKのlinker script

`<nRF SDK>/examples/peripheral/blinky/pca10059/mbr/armgcc/blinky_gcc_nrf52.ld`

```
MEMORY
{
  FLASH (rx) : ORIGIN = 0x1000, LENGTH = 0xff000
  RAM (rwx) :  ORIGIN = 0x20000008, LENGTH = 0x3fff8
}
```

pca10056には、mbrとblankという2種類が用意されている。
blankの方は、ORIGINが0x0000になっている。

### linker scriptの出処

`zephyr/soc/arm/nordic_nrf/nrf52/linker.ld`

```
/* linker.ld - Linker command/script file */

/*
 * Copyright (c) 2014 Wind River Systems, Inc.
 *
 * SPDX-License-Identifier: Apache-2.0
 */

#include <arch/arm/cortex_m/scripts/linker.ld>
```

cortex_mのlinker scriptをincludeしているだけ。

`zephyr/include/arch/arm/cortex_m/scripts/linker.ld`
ここだな。

```
MEMORY
    {
    FLASH                 (rx) : ORIGIN = ROM_ADDR, LENGTH = ROM_SIZE
...
    SRAM                  (wx) : ORIGIN = RAM_ADDR, LENGTH = RAM_SIZE

    /* Used by and documented in include/linker/intlist.ld */
    IDT_LIST  (wx)      : ORIGIN = (RAM_ADDR + RAM_SIZE), LENGTH = 2K
    }
```

`ROM_ADDR`や`ROM_SIZE`はマクロになっており、これらのマクロは、同ファイルで下記のように定義されている。

```
#define ROM_ADDR (CONFIG_FLASH_BASE_ADDRESS + CONFIG_FLASH_LOAD_OFFSET)
#ifdef CONFIG_TI_CCFG_PRESENT
  #define CCFG_SIZE 88
  #define ROM_SIZE (CONFIG_FLASH_SIZE*1K - CONFIG_FLASH_LOAD_OFFSET - \
		    CCFG_SIZE)
  #define CCFG_ADDR (ROM_ADDR + ROM_SIZE)
#else
#if CONFIG_FLASH_LOAD_SIZE > 0
  #define ROM_SIZE CONFIG_FLASH_LOAD_SIZE
#else
  #define ROM_SIZE (CONFIG_FLASH_SIZE*1K - CONFIG_FLASH_LOAD_OFFSET)
#endif
#endif
```

`CONFIG_FLASH_LOAD_OFFSET`を設定できれば良さそう。
`Kconfig.zephyr`で次のようになっている。

```
config HAS_FLASH_LOAD_OFFSET
	bool
	help
	  This option is selected by targets having a FLASH_LOAD_OFFSET
	  and FLASH_LOAD_SIZE.

if !HAS_DTS
config FLASH_LOAD_OFFSET
	hex "Kernel load offset"
	default 0
	depends on HAS_FLASH_LOAD_OFFSET
	help
	  This option specifies the byte offset from the beginning of flash that
	  the kernel should be loaded into. Changing this value from zero will
	  affect the Zephyr image's link, and will decrease the total amount of
	  flash available for use by application code.

	  If unsure, leave at the default value 0.
```

device treeで設定できるみたい。
余談、Custom linker scripts providedという設定がある。ファイルパスを指定することができるようだ。

[Zephyr Doc Device Tree Flash Partitions](https://docs.zephyrproject.org/1.13.0/devices/dts/flash_partitions.html)

`zephyr/boards/arm/nrf52840_pca10059/nrf52840_pca10059.dts`を見ると、`slot0_partition`が0x1000から始まっている。

```
&flash0 {
	/*
	 * For more information, see:
	 * http://docs.zephyrproject.org/latest/devices/dts/flash_partitions.html
	 */
	partitions {
		compatible = "fixed-partitions";
		#address-cells = <1>;
		#size-cells = <1>;

		boot_partition: partition@e0000 {
			label = "mcuboot";
			reg = <0x0000e0000 0x0001a000>;
		};
		slot0_partition: partition@1000 {
			label = "image-0";
			reg = <0x00001000 0x000060000>;
		};
		slot1_partition: partition@61000 {
			label = "image-1";
			reg = <0x0061000 0x000060000>;
		};
		scratch_partition: partition@c1000 {
			label = "image-scratch";
			reg = <0x000c1000 0x0001f000>;
		};
		storage_partition: partition@fa000 {
			label = "storage";
			reg = <0x000fa000 0x00004000>;
		};
	};
};
```

chosenで`zephyr,code-partition`をこのパーティションにしてあげると、linker.cmdが修正される。

```
	chosen {
		zephyr,console = &uart0;
		zephyr,uart-mcumgr = &uart0;
		zephyr,sram = &sram0;
		zephyr,flash = &flash0;
		zephyr,code-partition = &slot0_partition;
	};
```

`linker.cmd`

```
 OUTPUT_FORMAT("elf32-littlearm")
_region_min_align = 32;
MEMORY
    {
    FLASH (rx) : ORIGIN = (0x0 + 4096), LENGTH = 393216
    SRAM (wx) : ORIGIN = 0x20000000, LENGTH = (256 * 1K)
    IDT_LIST (wx) : ORIGIN = (0x20000000 + (256 * 1K)), LENGTH = 2K
    }
...
```
