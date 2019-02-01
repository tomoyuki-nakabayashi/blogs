# nRF52840 Dongle

## specification

- CPU: Cortex-M4
- Flash: 1MB
  - 0x0000_0000 - 0x000F_F000
- RAM: 256kB
  - 0x2000_0000 - 0x2003_8000

ボードの型番は`pca10059`。nRF52840 DKは`pca10056`なのでドキュメントや設定ファイルを触る際は注意が必要。

どちらも、nRF52840のSoCを使っている。

## Tutorial

nRF52840 Dongleを動かす手順は、下のチュートリアルが詳しい。

[nRF52840 Dongle Programming Tutorial](https://devzone.nordicsemi.com/tutorials/b/getting-started/posts/nrf52840-dongle-programming-tutorial)

> The MBR occupies the first flash page (address 0 - 0xFFF) and also reserves the lowest 8 bytes of RAM (0x20000000 - 0x20000007) for interrupt forwarding. The MBRs primary responsibility is to support safe upgrades of the bootloader. The MBR itself is never updated.

Flashメモリの先頭4KBはMBR領域となっている。

## Minimum requirements

[nRF Connect for Desktop](https://www.nordicsemi.com/Software-and-Tools/Development-Tools/nRF-Connect-for-desktop)

これの`Programmer`でGUIからFlashの書き込みができる。最初はこちらで書き込みを試すと良い。
コマンドラインツールもあり、慣れてくると便利だが、少しとっつきにくい。

## Getting started

### Minimum requirements

上記ページで、インストーラ兼ランチャがダウンロードできる。

```
$ chmod +x nrfconnect260x8664.AppImage
$ ./nrfconnect260x8664.AppImage
```

`Add/remove apps`で`Programmer`をインストールする。

`Launch app`で`Programmer`を起動する。

デバイスがないと、怒られる。そら、挿してないからね。
デバイスを挿すとエラーが発生した。
リロードしてみる。

```
07:41:37.847	Error while probing usb device at bus.address 1.14: LIBUSB_ERROR_ACCESS. Please check your udev rules concerning permissions for USB devices, see https://github.com/NordicSemiconductor/nrf-udev
07:41:37.847	Error while probing devices: Error occured when get serial numbers. Errorcode: CouldNotOpenDLL (0x7) Lowlevel error: JLINKARM_DLL_NOT_FOUND (ffffff9c)
```

udevを設定しろ、とのこと。
書かれている通り、[nrf-udev](https://github.com/NordicSemiconductor/nrf-udev)に行ってみる。
debianはパッケージが用意されている。

```
$ wget https://github.com/NordicSemiconductor/nrf-udev/releases/download/v1.0.1/nrf-udev_1.0.1-all.deb
$ sudo dpkg -i nrf-udev_1.0.1-all.deb
Selecting previously unselected package nrf-udev.
(Reading database ... 599239 files and directories currently installed.)
Preparing to unpack nrf-udev_1.0.1-all.deb ...
Unpacking nrf-udev (1.0.1) ...
Setting up nrf-udev (1.0.1) ...
Reloading udev rules...
```

[udevルール](https://github.com/NordicSemiconductor/nrf-udev/blob/master/nrf-udev_1.0.1-all/lib/udev/rules.d/71-nrf.rules)を確認してみる。

```
# 71-nrf.rules
ACTION!="add", SUBSYSTEM!="usb_device", GOTO="nrf_rules_end"

# Set /dev/bus/usb/*/* as read-write for all users (0666) for Nordic Semiconductor devices
SUBSYSTEM=="usb", ATTRS{idVendor}=="1915", MODE="0666"

# Flag USB CDC ACM devices, handled later in 99-mm-nrf-blacklist.rules
# Set USB CDC ACM devnodes as read-write for all users
KERNEL=="ttyACM[0-9]*", SUBSYSTEM=="tty", SUBSYSTEMS=="usb", ATTRS{idVendor}=="1915", MODE="0666", ENV{NRF_CDC_ACM}="1"

LABEL="nrf_rules_end"
```

どうも対象のUSBが挿入されると、`/dev/ttyACMx`が生えるみたい。

ドングルを挿しなおして、Programmerをリロードするとデバイスが選択できるようになる。

### Getting started

とりあえず何か動かしてみたいので、Getting Started Assistantに沿って、やってみる。

#### Install the toolchain

けっこうツールチェインのインストールに色々求められるな。

git/wgetは入っているから良い。
cmake、地味にそこそこ新しいversionを求める。なんかPATHのexport先がおかしいが必要なのかな？

> Instead of setting the environment variables every time you start a new session, define them in the``~/.zephyrrc`` file as described in Setting up the build environment.

ああ、なるほど。

~/.bashrcに追加して最低限使えるようにしておく。

```
mkdir ${HOME}/cmake && cd ${HOME}/cmake
wget https://cmake.org/files/v3.8/cmake-3.8.2-Linux-x86_64.sh
yes | sh cmake-3.8.2-Linux-x86_64.sh | cat
echo "export PATH=${PWD}/cmake-3.8.2-Linux-x86_64/bin:${PATH}" >> ${HOME}/.zephyrrc
cmake --version
```

```
$ tail -n 1 ~/.bashrc
export PATH=${PWD}/cmake-3.8.2-Linux-x86_64/bin:${PATH}
```

ninja

```
sudo apt-get install ninja-build
```

```
sudo apt-get install ccache
sudo apt-get install dfu-util
```

device tree compiler

```
[ $(apt-cache show device-tree-compiler | grep '^Version: .*$' | grep -Po '(\d.\d.\d+)' | sed s/\.//g) -ge '146' ] && sudo apt-get install device-tree-compiler || (wget http://mirrors.kernel.org/ubuntu/pool/main/d/device-tree-compiler/device-tree-compiler_1.4.7-1_amd64.deb && sudo dpkg -i device-tree-compiler_1.4.7-1_amd64.deb)
```

pythonは軒並み入っていた。

```
sudo apt-get install python3-pip
sudo apt-get install python3-setuptools
sudo apt-get install python3-wheel
```

下も軒並み入っていた。

```
sudo apt-get install xz-utils
sudo apt-get install file
sudo apt-get install make
sudo apt-get install gcc-multilib
```

armツールチェイン。

```
wget https://developer.arm.com/-/media/Files/downloads/gnu-rm/8-2018q4/gcc-arm-none-eabi-8-2018-q4-major-linux.tar.bz2?revision=d830f9dd-cd4f-406d-8672-cca9210dd220?product=GNU%20Arm%20Embedded%20Toolchain,64-bit,,Linux,8-2018-q4-major
```

何か手順を飛ばしてしまったのか、別PCでやると、次の2つも必要だった。

```
sudo apt install libusb-1.0.0-dev
sudo apt install libudev-dev
```

#### Clone the nRF Connect SDK

```
cd <sourcecode_root>
mkdir ncs
cd ncs
git clone https://github.com/NordicPlayground/fw-nrfconnect-zephyr.git zephyr
git clone https://github.com/NordicPlayground/fw-nrfconnect-mcuboot.git mcuboot
git clone https://github.com/NordicPlayground/fw-nrfconnect-nrf.git nrf
git clone https://github.com/NordicPlayground/nrfxlib.git nrfxlib
```

```
cd <sourcecode_root>/ncs/zephyr ; git checkout tags/v1.13.99-ncs2
cd <sourcecode_root>/ncs/mcuboot ; git checkout tags/v1.2.99-ncs2
cd <sourcecode_root>/ncs/nrf ; git checkout tags/v0.3.0
cd <sourcecode_root>/ncs/nrfxlib ; git checkout tags/v0.3.0
```

Zephyrのビルド準備をする。

```
cd <sourcecode_root>/ncs
pip3 install --user -r zephyr/scripts/requirements.txt
pip3 install --user -r nrf/scripts/requirements.txt
```

コマンドラインツールの`nrfutil`をインストールしておく。後々使用する。

https://infocenter.nordicsemi.com/index.jsp?topic=%2Fcom.nordic.infocenter.tools%2Fdita%2Ftools%2Fnrfutil%2Fnrfutil_installing_from_pypi.html

```
pip install nrfutil
```

helpを見てみる。

```
$ nrfutil dfu usb-serial --help
Usage: nrfutil dfu usb-serial [OPTIONS]

  Perform a Device Firmware Update on a device with a bootloader that
  supports USB serial DFU.

Options:
  -pkg, --package FILE            Filename of the DFU package.  [required]
  -p, --port TEXT                 Serial port address to which the device is
                                  connected. (e.g. COM1 in windows systems,
                                  /dev/ttyACM0 in linux/mac)  [required]
  -cd, --connect-delay INTEGER    Delay in seconds before each connection to
                                  the target device during DFU. Default is 3.
  -fc, --flow-control BOOLEAN     To enable flow control set this flag to 1
  -prn, --packet-receipt-notification INTEGER
                                  Set the packet receipt notification value
  -b, --baud-rate INTEGER         Set the baud rate
  --help                          Show this message and exit.
```

まず、packageを作って、dfuで書き込むらしい。

## nRF SDK example

SDKのサンプルを動かしてみる。

```
wget https://developer.nordicsemi.com/nRF5_SDK/nRF5_SDK_v15.x.x/nRF5_SDK_15.2.0_9412b96.zip
unzip nRF5_SDK_15.2.0_9412b96.zip
```

nRF Connectで`examples/peripheral/blinky/hex/blinky_pca10059_mbr.hex`を書いてあげると動いた。

MBRとアプリケーションだけが書かれているので、SoftDeviceはいらないみたい。
SoftDeviceは、firmware。

```
$ nrfutil  pkg generate --hw-version 52 --sd-req=0x00 --application blinky_pca10059_mbr.hex --application-version 1 pkg.zip
$ nrfutil dfu usb-serial -pkg pkg.zip -p /dev/ttyACM0
```

OK！
nRF52840 DongleのLEDのがチカチカするようになった。

## nRF SDK Exampl self build

プリビルドバイナリは動いたので、自分でコンパイルして、動かす。
下のファイルにtoolchainのパスを書けばOK。

`nRF5_SDK_15.2.0_9412b96/components/toolchain/gcc/Makefile.posix`

```
$ cat 
/home/nakabayashi/others/03.connectFree/nrf-connect/nRF5_SDK_15.2.0_9412b96/components/toolchain/gcc/Makefile.posix
GNU_INSTALL_ROOT ?= /home/nakabayashi/others/03.connectFree/nrf-connect/gnuarmemb/gcc-arm-none-eabi-8-2018-q4-major/bin/
GNU_VERSION ?= 8.2.1
GNU_PREFIX ?= arm-none-eabi
```

```
cd <SDK_ROOT>/nRF5_SDK_15.2.0_9412b96/examples/peripheral/blinky/pca10059/mbr/armgcc
$ make
mkdir _build
cd _build && mkdir nrf52840_xxaa
Assembling file: gcc_startup_nrf52840.S
Compiling file: main.c
Compiling file: nrf_log_frontend.c
Compiling file: nrf_log_str_formatter.c
Compiling file: boards.c
Compiling file: app_error.c
Compiling file: app_error_handler_gcc.c
Compiling file: app_error_weak.c
Compiling file: app_util_platform.c
Compiling file: nrf_assert.c
Compiling file: nrf_atomic.c
Compiling file: nrf_balloc.c
Compiling file: nrf_fprintf.c
Compiling file: nrf_fprintf_format.c
Compiling file: nrf_memobj.c
Compiling file: nrf_ringbuf.c
Compiling file: nrf_strerror.c
Compiling file: system_nrf52840.c
Linking target: _build/nrf52840_xxaa.out
   text	   data	    bss	    dec	    hex	filename
   1708	    108	     28	   1844	    734	_build/nrf52840_xxaa.out
Preparing: _build/nrf52840_xxaa.hex
Preparing: _build/nrf52840_xxaa.bin
DONE nrf52840_xxaa
```

```
$ cd _build
$ nrfutil  pkg generate --hw-version 52 --sd-req=0x00 --application blinky_pca10059_mbr.hex --application-version 1 pkg.zip
$ nrfutil dfu usb-serial -pkg pkg.zip -p /dev/ttyACM0
```

無事、プリビルドバイナリと同じ挙動になった。

バイナリハックしてみる。

```
$ readelf -l nrf52840_xxaa.out

Elf file type is EXEC (Executable file)
Entry point 0x12b5
There are 3 program headers, starting at offset 52

Program Headers:
  Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
  EXIDX          0x0016a4 0x000016a4 0x000016a4 0x00008 0x00008 R   0x4
  LOAD           0x000000 0x00000000 0x00000000 0x016ac 0x016ac R E 0x10000
  LOAD           0x010008 0x20000008 0x000016ac 0x0006c 0x00088 RW  0x10000

 Section to Segment mapping:
  Segment Sections...
   00     .ARM.exidx 
   01     .text .ARM.exidx 
   02     .data .bss
```

変な位置にエントリポイントがあるのは、MRBの4KBをスキップしているらしい。

## Zephyr Blinky

Blinky sample applicationならドングルでも動くらしい。

zephyr/samples/basic/blinky/build

```
source ~/others/03.connectFree/nrf-connect/srcs/ncs/zephyr/zephyr-env.sh
# On Linux/macOS
cd $ZEPHYR_BASE/samples/basic/blinky
mkdir build && cd build

# On Windows
cd %ZEPHYR_BASE%\samples\basic\blinky
mkdir build & cd build


# Use cmake to configure a Ninja-based build system:
cmake -GNinja -DBOARD=nrf52840_pca10059 ..

# Now run ninja on the generated build system:
ninja
```

`|> log`

```
$ cmake -GNinja -DBOARD=nrf52840_p
ca10059 ..
-- Found PythonInterp: /usr/bin/python3 (found suitable version "3.5.2", minimum required is "3.4") 
-- Selected BOARD nrf52840_pca10059
Zephyr version: 1.13.99
Parsing Kconfig tree in /home/nakabayashi/others/03.connectFree/nrf-connect/srcs/ncs/zephyr/Kconfig
Loading /home/nakabayashi/others/03.connectFree/nrf-connect/srcs/ncs/zephyr/boards/arm/nrf52840_pca10059/nrf52840_pca10059_defconfig as base
Merging /home/nakabayashi/others/03.connectFree/nrf-connect/srcs/ncs/zephyr/samples/basic/blinky/prj.conf

warning: RTT_CONSOLE (defined at drivers/console/Kconfig:117) was assigned the value 'y' but got the
value 'n'. You can check symbol information (including dependencies) in the 'menuconfig' interface
(see the Application Development Primer section of the manual), or in the Kconfig reference at
http://docs.zephyrproject.org/latest/reference/kconfig/CONFIG_RTT_CONSOLE.html (which is updated
regularly from the master branch). See the 'Setting configuration values' section of the Board
Porting Guide as well.

warning: UART_INTERRUPT_DRIVEN (defined at drivers/serial/Kconfig:31) was assigned the value 'y' but
got the value 'n'. You can check symbol information (including dependencies) in the 'menuconfig'
interface (see the Application Development Primer section of the manual), or in the Kconfig
reference at
http://docs.zephyrproject.org/latest/reference/kconfig/CONFIG_UART_INTERRUPT_DRIVEN.html (which is
updated regularly from the master branch). See the 'Setting configuration values' section of the
Board Porting Guide as well.

warning: UART_NRFX (defined at drivers/serial/Kconfig.nrfx:8) was assigned the value 'y' but got the
value 'n'. You can check symbol information (including dependencies) in the 'menuconfig' interface
(see the Application Development Primer section of the manual), or in the Kconfig reference at
http://docs.zephyrproject.org/latest/reference/kconfig/CONFIG_UART_NRFX.html (which is updated
regularly from the master branch). See the 'Setting configuration values' section of the Board
Porting Guide as well.

warning: the choice symbol UART_0_NRF_UARTE (defined at drivers/serial/Kconfig.nrfx:34) was selected
(set =y), but no symbol ended up as the choice selection.  You can check symbol information
(including dependencies) in the 'menuconfig' interface (see the Application Development Primer
section of the manual), or in the Kconfig reference at
http://docs.zephyrproject.org/latest/reference/kconfig/CONFIG_UART_0_NRF_UARTE.html (which is
updated regularly from the master branch). See the 'Setting configuration values' section of the
Board Porting Guide as well.
-- Loading /home/nakabayashi/others/03.connectFree/nrf-connect/srcs/ncs/zephyr/boards/arm/nrf52840_pca10059/nrf52840_pca10059.dts as base
-- Overlaying /home/nakabayashi/others/03.connectFree/nrf-connect/srcs/ncs/zephyr/dts/common/common.dts
nrf52840_pca10059.dts_compiled: Warning (unique_unit_address): /soc/i2c@40003000: duplicate unit-address (also used in node /soc/spi@40003000)
nrf52840_pca10059.dts_compiled: Warning (unique_unit_address): /soc/i2c@40004000: duplicate unit-address (also used in node /soc/spi@40004000)
-- Cache files will be written to: /home/nakabayashi/.cache/zephyr
-- The C compiler identification is GNU 8.2.1
-- The CXX compiler identification is GNU 8.2.1
-- The ASM compiler identification is GNU
-- Found assembler: /home/nakabayashi/others/03.connectFree/nrf-connect/gnuarmemb/gcc-arm-none-eabi-8-2018-q4-major/bin/arm-none-eabi-gcc
-- Performing Test toolchain_is_ok
-- Performing Test toolchain_is_ok - Success
-- Configuring done
-- Generating done
-- Build files have been written to: /home/nakabayashi/others/03.connectFree/nrf-connect/srcs/ncs/zephyr/samples/basic/blinky/build
```

ビルドできた。

```
nrfutil pkg generate --hw-version 52 --sd-req=0x00 \
        --application zephyr.hex --application-version 1 pkg.zip
nrfutil dfu usb_serial -pkg pkg.zip -p /dev/ttyACM0
```

書き込むとうまく動かない。

## nRF Zephyr

動かない原因を解析する。

SDKのサンプルと、随分セクション情報が違う。

```
$ readelf -l zephyr.elf

Elf file type is EXEC (Executable file)
Entry point 0x191d
There are 4 program headers, starting at offset 52

Program Headers:
  Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
  EXIDX          0x00a9a8 0x0000a8f0 0x0000a8f0 0x00008 0x00008 R   0x4
  LOAD           0x0000b8 0x00000000 0x00000000 0x0b0f0 0x0b0f0 RWE 0x8
  LOAD           0x00b1a8 0x20002838 0x0000b0f0 0x00334 0x00334 RW  0x8
  LOAD           0x00b4e0 0x20000000 0x20000000 0x00000 0x02838 RW  0x8

 Section to Segment mapping:
  Segment Sections...
   00     .ARM.exidx 
   01     text .ARM.exidx sw_isr_table devconfig rodata 
   02     datas initlevel _k_sem_area _k_mutex_area _k_queue_area _net_buf_pool_area 
   03     bss noinit
```

PCを0x191dにしていそうだから、良い気がするが。vector_tableが0x0000から始まっている。

```
00000000 <_vector_table>:
       0:       20001af8        strdcs  r1, [r0], -r8
       4:       0000191d        andeq   r1, r0, sp, lsl r9
```

nRF SDKのreset vectorは0x1000から始まっているな。MBRを飛ばす形になっている。

```
00001000 <__isr_vector>:
    1000:       20040000        andcs   r0, r4, r0
    1004:       000012b5                        ; <UNDEFINED> instruction: 0x000
012b5
```

hexファイルは[Intel Hex](https://ja.wikipedia.org/wiki/Intel_HEX)形式。

Zephyrのリンカスクリプトがずれている説？
最終的に、`linker_pass_final.cmd`でリンクするみたいだが、なぜか`linker.cmd`も修正しないと動くバイナリができない。
これは追って調査が必要。

`linker.cmd`と`linker_pass_final.cmd`を修正して、FLASHを`0x1000`から始めるようにしたらLEDが光った。

```
$ head -n 10 linker_pass_final.cmd
nal.cmd
 OUTPUT_FORMAT("elf32-littlearm")
_region_min_align = 32;
MEMORY
    {
    FLASH (rx) : ORIGIN = (0x1000 + 0), LENGTH = (1024*1K - 0x1000)
    SRAM (wx) : ORIGIN = 0x20000008, LENGTH = (256 * 1K - 8)
    IDT_LIST (wx) : ORIGIN = (0x20000000 + (256 * 1K)), LENGTH = 2K
    }
ENTRY("__start")
SECTIONS
```

## references

[datasheet](https://www.mouser.jp/datasheet/2/297/Nordic_06192018_PCA10059_Schematic_And_PCB-1372278.pdf)
[nRF52840 Dongle](http://infocenter.nordicsemi.com/index.jsp?topic=%2Fcom.nordic.infocenter.welcome%2Fdita%2Fwelcome%2Fstartpage.html&cp=0)
[specification](http://infocenter.nordicsemi.com/pdf/nRF52840_PS_v1.0.pdf)

[nRF Connect SDK](https://developer.nordicsemi.com/nRF_Connect_SDK/doc/0.3.0/nrf/index.html)
[Nordic Semiconductor Playground](https://github.com/NordicPlayground)

[nRF Documentation Library](https://www.nordicsemi.com/DocLib)

### メモ

Zephyrのメインラインでは、動作しない。Nordicのforkと何が違う？
