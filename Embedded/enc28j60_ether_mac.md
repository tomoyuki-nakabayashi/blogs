# Ethernet MAC controller

## Zephyr driver

`zephyr/drivers/ethernet/enc28j60.c`にdriverがある。

`zephyr/samples/net/echo_server/`で利用例がありそう。

dtsのbindingsが見つからないな。Kconfigで全て設定するのかしら？
driver見ると、そうっぽいな。

```c
static const struct eth_enc28j60_config eth_enc28j60_0_config = {
	.gpio_port = CONFIG_ETH_ENC28J60_0_GPIO_PORT_NAME,
	.gpio_pin = CONFIG_ETH_ENC28J60_0_GPIO_PIN,
	.spi_port = CONFIG_ETH_ENC28J60_0_SPI_PORT_NAME,
	.spi_freq  = CONFIG_ETH_ENC28J60_0_SPI_BUS_FREQ,
	.spi_slave = CONFIG_ETH_ENC28J60_0_SLAVE,
#ifdef CONFIG_ETH_ENC28J60_0_GPIO_SPI_CS
	.spi_cs_port = CONFIG_ETH_ENC28J60_0_SPI_CS_PORT_NAME,
	.spi_cs_pin = CONFIG_ETH_ENC28J60_0_SPI_CS_PIN,
#endif /* CONFIG_ETH_ENC28J60_0_GPIO_SPI_CS */
	.full_duplex = CONFIG_ETH_EN28J60_0_FULL_DUPLEX,
	.timeout = CONFIG_ETH_EN28J60_TIMEOUT,
};
```

## pca10059ピン配置

`SPI_0`を使う想定で行こう。

```
		spi0: spi@40003000 {
			compatible = "nordic,nrf-spi";
			#address-cells = <1>;
			#size-cells = <0>;
			reg = <0x40003000 0x1000>;
			interrupts = <3 1>;
			status = "disabled";
			label = "SPI_0";
		};
```

GPIOが32ピンだから、0.27, 0.26, 1.10になるはず。

```
&spi0 {
	status = "ok";
	sck-pin = <27>;
	mosi-pin = <26>;
	miso-pin = <42>;
};
```

GPIOに引き出されてないし。何だったらI2Cとかぶってるやん。

```
&i2c0 {
	status = "ok";
	sda-pin = <26>;
	scl-pin = <27>;
};
};
```

回路図見ても、上のピンは引き出されていないから、別のものを使おう。

https://www.nordicsemi.com/DocLib/Content/Product_Spec/nRF52840/latest/pin#pin_assign
https://www.mouser.jp/datasheet/2/297/Nordic_06192018_PCA10059_Schematic_And_PCB-1372278.pdf

まず、必要なピン数は、

int, SPIの3ピン, csあたり？リセットは↓に含まれていないから不要かな。

```c
static const struct eth_enc28j60_config eth_enc28j60_0_config = {
	.gpio_port = CONFIG_ETH_ENC28J60_0_GPIO_PORT_NAME,
	.gpio_pin = CONFIG_ETH_ENC28J60_0_GPIO_PIN,
	.spi_port = CONFIG_ETH_ENC28J60_0_SPI_PORT_NAME,
	.spi_freq  = CONFIG_ETH_ENC28J60_0_SPI_BUS_FREQ,
	.spi_slave = CONFIG_ETH_ENC28J60_0_SLAVE,
#ifdef CONFIG_ETH_ENC28J60_0_GPIO_SPI_CS
	.spi_cs_port = CONFIG_ETH_ENC28J60_0_SPI_CS_PORT_NAME,
	.spi_cs_pin = CONFIG_ETH_ENC28J60_0_SPI_CS_PIN,
#endif /* CONFIG_ETH_ENC28J60_0_GPIO_SPI_CS */
	.full_duplex = CONFIG_ETH_EN28J60_0_FULL_DUPLEX,
	.timeout = CONFIG_ETH_EN28J60_TIMEOUT,
};
```

`CONFIG_ETH_ENC28J60_0_GPIO_PIN`が`INT`に使われるピン。
https://docs.zephyrproject.org/1.12.0/reference/kconfig/CONFIG_ETH_ENC28J60_0_GPIO_PIN.html

ということで、遅いやつ専用のGPIO以外が5ピンあれば、それでいこう。

SPIは、13, 15, 17を使うか。

```
AD8	    P0.13	Digital I/O	General purpose I/O	
AD10	P0.15	Digital I/O	General purpose I/O	
AD12	P0.17	Digital I/O	General purpose I/O
```

INTとCSで、0.22と1.00とを使えば、一応全部速いピンになる。
1.00のお勧めが、`Serial wire output`になっているから、SPIのCKかMOSIに使うかな。

CK -> 1.00  
MOSI -> 0.13  
MISO -> 0.15  
CS -> 0.17  
INT -> 0.22

うん、これで行こう。

## Device Tree

まずは、いらないi2c0/1とspi1を削除する。

```
&spi0 {
	status = "ok";
	sck-pin = <32>;
	mosi-pin = <13>;
	miso-pin = <15>;
};
```

とりあえずこれで。

## SPI単体動作確認

spidev相当のものはあるか、探してみよう。
それっぽいものが見つからないな。

## network subsystem

network subsystemを有効化すると、shellでpingなどが打てるはずだ！

https://docs.zephyrproject.org/1.13.0/samples/net/echo_server/README.html

> overlay-enc28j60.conf This overlay config enables support for enc28j60 ethernet board. This add-on board can be used for example with Arduino 101 board.

Arduino 101ボードのdtsを見れば良さそうか。あ、add-onボードだから見てもダメだわ。

## echo_server sample

### ビルド方法

```
cd $ZEPHYR_BASE/samples/net/echo_server
mkdir build && cd build
cmake -GNinja -DBOARD=<board to use> -DCONF_FILE=<config file to use> ..
ninja
```

例：

```
cd $ZEPHYR_BASE/samples/net/echo_server
mkdir build && cd build
cmake -GNinja -DBOARD=frdm_k64f -DCONF_FILE="prj.conf overlay-frdm_k64f_cc2520.conf" ..
ninja run
```

なので、下のような感じで良いはず。
問題は、これでconfig足りるのか？INTとCSが紐付かない気がするが。

```
cd $ZEPHYR_BASE/samples/net/echo_server
mkdir build && cd build
cmake -GNinja -DBOARD=nrf52840_pca10059 -DCONF_FILE="prj.conf overlay-enc28j60.conf" ..
ninja run
```

### 修正箇所

overlay-enc28j60.conf

```
CONFIG_NET_L2_ETHERNET=y

# ENC28J60 Ethernet Device
CONFIG_ETH_ENC28J60=y
CONFIG_ETH_ENC28J60_0=y
CONFIG_ETH_ENC28J60_0_SPI_PORT_NAME="SPI_0"
CONFIG_ETH_ENC28J60_0_MAC3=0x2D
CONFIG_ETH_ENC28J60_0_MAC4=0x30
CONFIG_ETH_ENC28J60_0_MAC5=0x36

CONFIG_ETH_ENC28J60_0_GPIO_PORT_NAME="GPIO_0"
CONFIG_ETH_ENC28J60_0_GPIO_PIN=22

CONFIG_ETH_ENC28J60_0_GPIO_SPI_CS=y
CONFIG_ETH_ENC28J60_0_SPI_CS_PORT_NAME="GPIO_0"
CONFIG_ETH_ENC28J60_0_SPI_CS_PIN=17
```

`boards/nrf52840_pca10056.conf`をコピーして`boards/nrf52840_pca10059.conf`を作成した。

```
mkdir build && cd build
cmake -GNinja -DBOARD=nrf52840_pca10059 -DCONF_FILE="boards/nrf52840_pca10059.conf prj.conf overlay-enc28j60.conf" ..
ninja run
```

とりあえず、通っちゃったな。

```
nrfutil  pkg generate --hw-version 52 --sd-req=0x00 --application zephyr.hex --application-version 1 pkg.zip
nrfutil dfu usb-serial -pkg pkg.zip -p /dev/ttyACM0
```

なんらか動こうとはしているっぽい？

```
***** Booting Zephyr OS v1.13.99-ncs2 *****                             
                                                                                
                                                                                
[00:00:00.004,699] <err> net_ctx.net_context_bind: Cannot bind to ::            
[00:00:00.004,699] <err> net_app._net_app_set_net_ctx: Cannot bind context (-49)
[00:00:00.004,730] <err> net_echo_server_tcp.start_tcp: Cannot wait connection )
[00:00:00.005,950] <err> net_ctx.net_context_bind: Cannot bind to 0.0.0.0       
[00:00:00.005,981] <err> net_app._net_app_set_net_ctx: Cannot bind context (-49)
[00:00:00.006,011] <err> net_ctx.net_context_bind: Cannot bind to ::            
[00:00:00.006,011] <err> net_app._net_app_set_net_ctx: Cannot bind context (-49)
[00:00:00.006,011] <err> net_echo_server_udp.start_udp: Cannot wait connection )
```

ま、焦らずに、SPI出てるかどうか、からやるか。

### SPIプローブ

波形出ていない。

.configを確認する。

```
# CONFIG_SPI is not set
```

（あかんやつや）

menuconfigで関係ありそうなものをガーッと。

```
CONFIG_SPI=y
...
CONFIG_SPI_INIT_PRIORITY=70
# CONFIG_SPI_LOG_LEVEL_OFF is not set
# CONFIG_SPI_LOG_LEVEL_ERR is not set
# CONFIG_SPI_LOG_LEVEL_WRN is not set
CONFIG_SPI_LOG_LEVEL_INF=y
# CONFIG_SPI_LOG_LEVEL_DBG is not set
CONFIG_SPI_LOG_LEVEL=3
CONFIG_SPI_0=y
CONFIG_SPI_0_OP_MODES=1
# CONFIG_SPI_1 is not set
# CONFIG_SPI_2 is not set
# CONFIG_SPI_3 is not set
# CONFIG_SPI_4 is not set
# CONFIG_SPI_5 is not set
CONFIG_SPI_NRFX=y
CONFIG_SPI_0_NRF_SPI=y
# CONFIG_SPI_0_NRF_SPIM is not set
```

うーん？

```
[00:00:00.004,821] <err> net_ctx.net_context_bind: Cannot bind to ::            
[00:00:00.004,821] <err> net_app._net_app_set_net_ctx: Cannot bind context (-49)
[00:00:00.004,852] <err> net_echo_server_tcp.start_tcp: Cannot wait connection )
[00:00:00.006,042] <err> net_ctx.net_context_bind: Cannot bind to 0.0.0.0       
[00:00:00.006,042] <err> net_app._net_app_set_net_ctx: Cannot bind context (-49)
[00:00:00.006,103] <err> net_ctx.net_context_bind: Cannot bind to ::            
[00:00:00.006,103] <err> net_app._net_app_set_net_ctx: Cannot bind context (-49)
[00:00:00.006,103] <err> net_echo_server_udp.start_udp: Cannot wait connection )
```

SPIのエラーが出てないんだよなぁ。

```
3 0x40003000 SPI SPI0 SPI master 0 Deprecated
3 0x40003000 SPIM SPIM0 SPI master 0
3 0x40003000 SPIS SPIS0 SPI slave 0
3 0x40003000 TWI TWI0 Two-wire interface master 0 Deprecated
3 0x40003000 TWIM TWIM0 Two-wire interface master 0
3 0x40003000 TWIS TWIS0 Two-wire interface slave 0
```

SPIMのdriverが必要？

```
# CONFIG_ETH_ENC28J60 is not set
```

あれ？

一通りmenuconfigで有効した。

```
uart:~$                                                                         
                                                                                
[00:00:00.000,000] <err> spi_nrfx_spi.init_spi: Function: init_spi.
```

変なエラーは出てない。SPIドライバもロードされている。
どうかな？

```
$ net iface show                                                          
                                                                                
Interface 0x2000cee0 (Ethernet) [0]                                             
===================================                                             
Link addr : 00:04:A3:00:00:00                                                   
MTU       : 1500                                                                
Ethernet capabilities supported:                                                
        10 Mbits                                                                
IPv6 unicast addresses (max 3):                                                 
        fe80::204:a3ff:fe00:0 autoconf preferred infinite                       
        2001:db8::1 manual preferred infinite                                   
IPv6 multicast addresses (max 4):                                               
        <none>                                                                  
IPv6 prefixes (max 2):                                                          
        <none>                                                                  
IPv6 hop limit           : 64                                                   
IPv6 base reachable time : 30000                                                
IPv6 reachable time      : 43209                                                
IPv6 retransmit timer    : 0                                                    
IPv4 unicast addresses (max 1):                                                 
        192.0.2.1 manual preferred infinite                                     
IPv4 multicast addresses (max 1):                                               
        <none>                                                                  
IPv4 gateway : 0.0.0.0                                                          
IPv4 netmask : 255.255.255.0
```

インタフェース見えているね？

自分宛てにpingを送ると帰ってきているっぽい。

```
$ net ping 192.0.2.1                                                      
Received echo reply from 192.0.2.1 to 192.0.2.1                                 
Sent a ping to 192.0.2.1
```

Ethernet MAC拡張ボードの電源供給を止めると、interfaceがdownしている。

```
$ net iface show

Interface 0x2000cee0 (Ethernet) [0]
===================================
Interface is down.
```

```
$ net ping 192.168.200.100                                                
Sent a ping to 192.168.200.100                                                  
Ping timeout
```

pingは通らないな。
あれ、ホスト側でarpコマンド打つと、なんか見えてる。MACアドレスも合っているし。

```
$ arp
Address                  HWtype  HWaddress           Flags Mask            Iface
...
192.168.200.101          ether   00:04:a3:00:00:00   C                     enx4ce67655b31f
```

data bufferサイズの問題？

```
$ net ping 192.168.200.100
Sent a ping to 192.168.200.100
Ping timeout
[00:01:37.736,785] <err> eth_enc28j60.eth_enc28j60_rx: Could not allocate data buffer
```

エラーの発生箇所を調べると割り込みハンドラ内だぞ？

```c
static int eth_enc28j60_rx(struct device *dev)
{
...
		/* Get the frame from the buffer */
		pkt = net_pkt_get_reserve_rx(0, config->timeout);
		if (!pkt) {
			LOG_ERR("Could not allocate rx buffer");
			goto done;
		}
...
```

メモリ不足？

`|> zephyr/subsys/net/ip/net_pkt.c`

```c
struct net_pkt *net_pkt_get_reserve(struct k_mem_slab *slab,
				    u16_t reserve_head,
				    s32_t timeout)
{
	struct net_pkt *pkt;
	int ret;

	if (k_is_in_isr()) {
		ret = k_mem_slab_alloc(slab, (void **)&pkt, K_NO_WAIT);
	} else {
		ret = k_mem_slab_alloc(slab, (void **)&pkt, timeout);
	}

	if (ret) {
		return NULL;
	}
...
```

ここか？普通にslab_allocatorからメモリが取れていない？

ENC28J60C Ethernet Controllerの設定にstack sizeがある。これを増やしてみるか。

```
(800) Stack size for internal incoming packet handler
```

```
$ net ping 192.168.200.100
Sent a ping to 192.168.200.100
Ping timeout
```

普通にタイムアウトするだけやし。

```
enx4ce67655b31f Link encap:Ethernet  HWaddr 4c:e6:76:55:b3:1f  
          inet addr:192.168.200.100  Bcast:192.168.200.255  Mask:255.255.255.0
          RX packets:243 errors:0 dropped:0 overruns:0 frame:0
          TX packets:570 errors:0 dropped:0 overruns:0 carrier:0
          RX bytes:54200 (54.2 KB)  TX bytes:43832 (43.8 KB)

enx4ce67655b31f Link encap:Ethernet  HWaddr 4c:e6:76:55:b3:1f  
          inet addr:192.168.200.100  Bcast:192.168.200.255  Mask:255.255.255.0
          RX packets:246 errors:0 dropped:0 overruns:0 frame:0
          TX packets:573 errors:0 dropped:0 overruns:0 carrier:0
          RX bytes:54358 (54.3 KB)  TX bytes:44106 (44.1 KB)

enx4ce67655b31f Link encap:Ethernet  HWaddr 4c:e6:76:55:b3:1f  
          inet addr:192.168.200.100  Bcast:192.168.200.255  Mask:255.255.255.0
          RX packets:247 errors:0 dropped:0 overruns:0 frame:0
          TX packets:574 errors:0 dropped:0 overruns:0 carrier:0
          RX bytes:54404 (54.4 KB)  TX bytes:44152 (44.1 KB)
```

pingを打つ度、packetは増える！
tcpdumpさんの出番か。
やっぱりお喋りしようとはしているな。

```
$ sudo tcpdump -i enx4ce67655b31f -vv
tcpdump: listening on enx4ce67655b31f, link-type EN10MB (Ethernet), capture size 262144 bytes
09:54:49.261768 IP6 (hlim 1, next-header Options (0) payload length: 56) fe80::4ee6:76ff:fe55:b31f > ff02::16: HBH (rtalert: 0x0000) (padn) [icmp6 sum ok] ICMP6, multicast listener report v2, 2 group record(s) [gaddr ff02::fb to_ex { }] [gaddr ff02::1:ff55:b31f to_ex { }]
09:54:49.261946 IP6 (hlim 255, next-header ICMPv6 (58) payload length: 16) fe80::204:a3ff:fe00:0 > ip6-allnodes: [icmp6 sum ok] ICMP6, router solicitation, length 16
	  source link-address option (1), length 8 (1): 00:04:a3:00:00:00
	    0x0000:  0004 a300 0000
09:54:49.261975 IP6 (hlim 1, next-header Options (0) payload length: 56) fe80::4ee6:76ff:fe55:b31f > ff02::16: HBH (rtalert: 0x0000) (padn) [icmp6 sum ok] ICMP6, multicast listener report v2, 2 group record(s) [gaddr ff02::fb to_ex { }] [gaddr ff02::1:ff55:b31f to_ex { }]
09:54:49.549833 IP6 (hlim 1, next-header Options (0) payload length: 56) fe80::4ee6:76ff:fe55:b31f > ff02::16: HBH (rtalert: 0x0000) (padn) [icmp6 sum ok] ICMP6, multicast listener report v2, 2 group record(s) [gaddr ff02::fb to_ex { }] [gaddr ff02::1:ff55:b31f to_ex { }]
09:54:49.550106 IP6 (hlim 1, next-header Options (0) payload length: 56) fe80::4ee6:76ff:fe55:b31f > ff02::16: HBH (rtalert: 0x0000) (padn) [icmp6 sum ok] ICMP6, multicast listener report v2, 2 group record(s) [gaddr ff02::fb to_ex { }] [gaddr ff02::1:ff55:b31f to_ex { }]
09:54:50.257631 IP6 (hlim 255, next-header ICMPv6 (58) payload length: 16) fe80::204:a3ff:fe00:0 > ip6-allnodes: [icmp6 sum ok] ICMP6, router solicitation, length 16
	  source link-address option (1), length 8 (1): 00:04:a3:00:00:00
	    0x0000:  0004 a300 0000

09:55:02.497410 ARP, Ethernet (len 6), IPv4 (len 4), Request who-has 192.168.200.100 tell 0.0.0.0, length 46
09:55:02.497429 ARP, Ethernet (len 6), IPv4 (len 4), Reply 192.168.200.100 is-at 4c:e6:76:55:b3:1f (oui Unknown), length 28
09:55:02.497583 ARP, Ethernet (len 6), IPv4 (len 4), Reply 192.168.200.100 is-at 4c:e6:76:55:b3:1f (oui Unknown), length 46
```

ARPを返してくれているけど、それに反応できていないのか。受信割り込み周りか？
ていうか、これIPv6だったら通るとかあるかね。割り込み周りだったら、通らないけど。

オシロスコープで確認したが、Zephyrでpingを打ったあと、割り込みは入ってきている。

IPv6の方はICMPで会話しようとしているのか？
そっか、IPv6だとarpじゃないんだな。

けっこうそれっぽく動いているよ？

```
$ net stats

Interface 0x2000d760 (Ethernet) [0]
===================================
IPv6 recv      2        sent    5       drop    0       forwarded       0
IPv6 ND recv   0        sent    5       drop    0
IPv6 MLD recv  0        sent    0       drop    0
IPv4 recv      0        sent    1       drop    0       forwarded       0
IP vhlerr      0        hblener 0       lblener 0
IP fragerr     0        chkerr  0       protoer 0
ICMP recv      2        sent    1       drop    2
ICMP typeer    0        chkerr  0
UDP recv       0        sent    0       drop    0
UDP chkerr     0
TCP bytes recv 0        sent    0
TCP seg recv   0        sent    0       drop    0
TCP seg resent 0        chkerr  0       ackerr  0
TCP seg rsterr 0        rst     0       re-xmit 0
TCP conn drop  0        connrst 0
Bytes received 220
Bytes sent     316
Processing err 0
```

## ログ強化

(top menu) → Networking → Link layer options → Enable Ethernet support → Log level for Ethernet L2 layer

(top menu) → Networking → IP stack

関係ありそうなのをざっとDebugレベルにする。

## IPv6 ping

```
$ net ping fe80::4ee6:76ff:fe55:b31f                                                                                        
Sent a ping to fe80::4ee6:76ff:fe55:b31f                                                                                          
Received echo reply from fe80::4ee6:76ff:fe55:b31f to fe80::204:a3ff:fe2d:3036
[00:01:42.711,364] <dbg> eth_enc28j60.eth_enc28j60_tx: pkt 0x20007d90 (len 62)                                                    
[00:01:42.718,536] <dbg> eth_enc28j60.eth_enc28j60_tx: Tx successful                                                              
[00:01:42.725,555] <dbg> eth_enc28j60.eth_enc28j60_rx: Received packet of length 62
```

IPv6だと動いているっぽい！

変更点をまとめよう。

## GPIOまとめ

| pin | description | Recommended usage |
| --- | --- |
| 0.13 | General purpose I/O |  |
| 0.15 | General purpose I/O |  |
| 0.17 | General purpose I/O |  |
| 0.20 | General purpose I/O |  |
| 0.22 | General purpose I/O |  |
| 0.24 | General purpose I/O |  |
| 1.00 | General purpose I/O. Trace buffer TRACEDATA[0]. Serial wire output (SWO). | QSPI |
| 0.09 | General purpose I/O. NFC antenna connection. | Standard drive, low frequency I/O only |
| 0.10 | General purpose I/O. NFC antenna connection. | Standard drive, low frequency I/O only |
| 0.31 | General purpose I/O. Analog input. | Standard drive, low frequency I/O only |
| 0.29 | General purpose I/O. Analog input. | Standard drive, low frequency I/O only |
| 0.02 | General purpose I/O. Analog input. | Standard drive, low frequency I/O only |
| 1.15 | General purpose I/O | Standard drive, low frequency I/O only |
| 1.13 | General purpose I/O | Standard drive, low frequency I/O only |
| 1.10 | General purpose I/O | Standard drive, low frequency I/O only |
