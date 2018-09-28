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

Intelが開発をサポートしており、リファレンスのSoCはApollo Lakeです。
主なユースケースとして、車載、産業機器を挙げています。

ちなみにACRNはどんぐり (acron)が由来だそうです。ロゴも人の上半身に見えますがどんぐりのポリゴンです。

# Version 0.2 new features

追いかけきれていないので、間違っている点があれば、ご指摘をお願いします。

## VT-x, VT-d

VT-xを使ってACRNが仮想CPUをエミュレーションできるようになったようです。
かなり前にコード見た時は、物理CPU1つに対して、仮想CPU1つを割り当てていたと思うので、そこが改善されたのでしょうか。

VT-dは、デバイスを管理するパーティションの所有者にデバイスアクセスを隔離し、制限するためのハードウェアサポートを提供します。
I/OデバイスをVMに割り当てたり、I/O操作用のVMの保護および分離プロパティを拡張することができます。

各guest OS間で、デバイスを隔離するための機能のようです。