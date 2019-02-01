# nRF52840 Memory Map

[Memory](https://www.nordicsemi.com/DocLib/Content/Product_Spec/nRF52840/latest/memory?19#memorymap)

下あたりだよね。Zephyrでどうなっているか。

| ID | Base address | Peripheral | Instance | Description |
| ---| --- | --- | --- | --- |
| 13 |	0x4000D000 |	RNG |	RNG	| Random number generator |
| 14 |	0x4000E000 |	ECB |	ECB	| AES electronic code book (ECB) mode block encryption |
| 15 |	0x4000F000 |	AAR |	AAR	| Accelerated address resolver |
| 15 |	0x4000F000 |	CCM |	CCM	| AES counter with CBC-MAC (CCM) mode block encryption |
| 42 |	0x5002A000 |	CRYPTOCELL |	CRYPTOCELL	| CryptoCell subsystem control interface |

`\> zephyr/ext/hal/nordic`の下で、ECBやCCMといったワードでgrepするといろいろとひっかかる。
[nrfx](https://github.com/NordicSemiconductor/nrfx)というNordic SoC用のdriverが、Zephyrの下に丸々入っている。

# nRF52840の暗号化アクセラレータCryptoCell 310

## nRF52840 Product Specification

http://infocenter.nordicsemi.com/index.jsp?topic=%2Fcom.nordic.infocenter.nrf52840.ps%2Fcryptocell.html

ARM TrustZoneを使った暗号化アクセラレータ。

機能

- 乱数生成
- 擬似乱数生成
- RSA公開鍵暗号
    - 鍵サイズ2048-bitまで
    - PKCS#1 v2.1/v.15
    - CRTはオプション
- 楕円曲線暗号 (ECC; Elliptic curve cryptography)
- Secure remote password protocol (SRP)
    - 3072-bitまで
- ハッシュ
    - SHA-1, SHA-2, 256-bitまで
    - HMAC
- AES暗号
    - 鍵サイズ128-bitまで
- ChaCha20/Poly1305 symmetric encryption

`ChaCha20/Poly1305 symmetric encryption`ってなんだろうと思ったら、TLSで採用されている暗号化方式らしい。

[ぼちぼち日記 新しいTLSの暗号方式ChaCha20-Poly1305](https://jovi0608.hatenablog.com/entry/20160404/1459748671)

### 使い方

レジスタインタフェースを直接叩くことはできず、SDKのライブラリを使います。
有効にするためには、ENABLEレジスタを使います。

### Always-on (AO) power domain

CryptoCellが無効な時でも、デバイスの秘密情報を保持するために、常に電源オン状態のパワードメインを持っています。

### DMA

CryptoCellはSRAMへのDMAを実装しています。

### Registers

Base address: 0x5002_A000  
ENABLE: 0x500  

# Zephyr nrfx

Zephyrのtreeに、nrfxがポーティングされている。
`|> zephyr/ext/hal/nordic/README`に次のように書かれている。

```
Purpose:
   With added proper shims adapting it to Zephyr's APIs, nrfx will provide
   peripheral drivers for Nordic SoCs.

Description:
   nrfx is an extract from the nRF5 SDK that contains solely the drivers
   for peripherals present in Nordic SoCs, for convenience complemented with
   the MDK package containing required structures and bitfields definitions,
   startup files etc.

   Only relevant files from nrfx were imported, for instance, template files
   and the ones with startup code or strictly related to installation of IRQs
   in the vector table were omitted.
```

halは、Hardware access layerで、driverより下位層に位置する。Hardware abstraction layerとややこしいからやめて欲しい。

ECB (AES electronic code book mode block encryption)やRNGには、driverがある。他は、ハードウェアアクセスするくらいの使い方しかないってことかな。
AAR (Accelerated address resolver)もあっても良さそうだけど・・・？

うーん、device treeでCC310の設定だけは追加されているけど、実装されている気配は一切ないな。これは諦めたほうがよさげ。

# nRF SDK

`|> nRF5_SDK_15.2.0_9412b96/examples/crypto`の下に色々とある。

```
$ ls
aes  chacha_poly  cli  ecdh  ecdsa  eddsa  hash  hkdf  hmac  rng  test_app
```

試しに`rng`を見てみる。

```c
#include <stdbool.h>
#include <stdint.h>
#include "boards.h"
#include "nrf_log_default_backends.h"
#include "nrf_log.h"
#include "nrf_log_ctrl.h"
#include "nrf_crypto.h"
...
int main(void)
{
    ret_code_t ret_val;

    log_init();

    NRF_LOG_INFO("RNG example started.");

    ret_val = nrf_crypto_init();
    APP_ERROR_CHECK(ret_val);

    // The RNG module is not explicitly initialized in this example, as
    // NRF_CRYPTO_RNG_AUTO_INIT_ENABLED and NRF_CRYPTO_RNG_STATIC_MEMORY_BUFFERS_ENABLED
    // are enabled in sdk_config.h.

    NRF_LOG_INFO("Generate %u random vectors of length %u:", ITERATIONS, VECTOR_LENGTH);
    for (int i = 0; i < ITERATIONS; i++)
    {
        ret_val = nrf_crypto_rng_vector_generate(m_random_vector, VECTOR_LENGTH);
        APP_ERROR_CHECK(ret_val);
        NRF_LOG_HEXDUMP_INFO(m_random_vector, VECTOR_LENGTH)
    }
...
```

`nrf_crypto_init()`や`nrf_crypto_rng_vector_generate()`は、`components/libraries/cryptos`下にある。

このあたりの関数を追いかけて行くと、`backend`という概念に行き着く。`backend`というディレクトリがあり、実際に処理をじっこうするバックエンドによってリンクされるモジュールが違ってくるようだ。

```
$ ls backend/
cc310  cc310_bl  cifra  mbedtls  micro_ecc  nrf_hw  nrf_sw  oberon
```

cc310を追いかけると途中でバイナリに突入してしまう。Arm TrustZoneなのである意味当然か。`CRYS_RND_GenerateVector()`

```
external/nrf_cc310_bl/include/crys_rnd.h:   Full description can be found in ::CRYS_RND_GenerateVector function API. */
external/nrf_cc310_bl/include/crys_rnd.h:CIMPORT_C CRYSError_t CRYS_RND_GenerateVector(
external/nrf_cc310_bl/include/crys_rnd.h:CIMPORT_C CRYSError_t CRYS_RND_GenerateVectorInRange(
external/nrf_cc310_bl/include/crys_rnd.h:to be later used by the ::CRYS_RND_Instantiation/::CRYS_RND_Reseeding/::CRYS_RND_GenerateVector functions.
```

hw_backendを追ってみる。これはmbedtls。

```c
ret_code_t nrf_crypto_rng_backend_vector_generate(void      * const p_context,
                                                  uint8_t   * const p_target,
                                                  size_t            size,
                                                  bool              use_mutex)
{
    int                         mbedtls_ret_val;
    mbedtls_ctr_drbg_context  * p_mbedtls_context =
        &((nrf_crypto_backend_rng_context_t *)p_context)->mbedtls_context;

    UNUSED_PARAMETER(use_mutex);

    mbedtls_ret_val = mbedtls_ctr_drbg_random(p_mbedtls_context, p_target, size);

    return result_get(mbedtls_ret_val);
}
```

`external/mbedtls`に飛ぶ。ふーむ。

RNG Exampleの説明を見ると、CC310があるとCC310がデフォルトになるようだ。nRF52840にはRNGハードウェアも載っているわけだが、CC310の方がよいらしい。
CC310が載っていないデバイスでは、nRF HW RNGがmbedTLSと共に使われるらしい。

[RNG Example](https://infocenter.nordicsemi.com/index.jsp?topic=%2Fcom.nordic.infocenter.sdk5.v15.0.0%2Flib_crypto_rng.html)

> CC310 is the preferred backend on devices that support it

## メモ

OpenThread

[nrf52840 cc310 is not initialized in some cases](https://github.com/openthread/openthread/issues/2896)

```make
NRF52840_MBEDTLS_CPPFLAGS += -I$(AbsTopSourceDir)/third_party/NordicSemiconductor/libraries/crypto
```

ということで、なんとかしようと思ったら、libraryを持ってこないとダメっぽい。うーん。