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

# CCM AES CCM mode encryption