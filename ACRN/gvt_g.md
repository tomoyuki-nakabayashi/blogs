# ACRN GVT-G

## GVT-G high-level design

[GVT-G high-level design](https://projectacrn.github.io/latest/developer-guides/APL_GVT-g-hld.html)

素晴らしいドキュメントが用意されているので、その理解から。

PlatformはApollo Lake SoC

The Programmer's Reference Manuals (PRM).

### Background

GVT-gはGPUの完全仮想化アプローチで、仲介者ありのパススルー技術に基づいている。
APLのGVT-gは、ACRN, KVM, Xenそれぞれの実装がある。今回の記事の対象はACRN-GVTである。

#### Targeted Usages

自動車アプリケーション。Instrument cluster + IVI + Rear seat entertainment or camera。

### Existing Techniques

#### Emulation

ハードウェアに依存しないので、互換性が高くなるが、CPUエミュレーションコストが高く、製品利用には耐えられない。

#### API Forwarding

virtioアプローチ。各VMのフロントエンドドライバが、GPUを制御するバックエンドドライバに、API呼び出し(DirectX, OpenGL)をforwardする。
メモリコピーを減らすために、hostとguestで共有メモリを利用する。
これで良い気がするが、次の問題がある。

- Lagging features: API更新への追従が遅れる。APIが追加されると、ドライバを更新しなければならない。
- Compatibility issues: GPUは複雑で、グラフィックAPIはhigh levelである。別プロトコルは巧妙な各APIごとに100%互換ではない。
- Maintenance burden: Graphics driverのプロトコルが変わったり、APIのバージョンが変わるごとにメンテナンスしないといけない。
- Performance overhead: API forwardingはかなりの性能差異が生まれる。

ピンと来なかったが、virtio的に考えると、VMにはGPUを仮想化した形でAPIを見せないといけないので、VMからは抽象化されたAPIしか見えない(あるGPU固有のAPIを見せることはできない)。この抽象化されたAPIを、DirectXとかOpenGL APIとして表現している、と思われる。中間表現のAPI使うことも考えられるが、内在する問題は一緒。
つまり、frontend→backendで使うAPIと、backend→Graphics driverで使うAPIはプロトコルが異なることになる(うーん日本語が良くない)。このAPI(プロトコル)変換が100%正しくできないよ、と言っていると考えられ、それはそうだ、という納得感。

#### Direct Pass-Through

性能では有利だが、VM間で共有できなくなる。

### Mediated Pass-Through

#### Concept

Mediated pass-throughでは、VMはほとんどの場合、hypervisorの介在なしで、performance criticalなリソースに直接アクセスできる。
特権オペレーションだけが、hypervisorにトラップされてエミュレーションされる。これは、VM間をsecureにisolationする。

hypervisorは、脆弱性がないことを保証しなければいけない。各VMにperformance-criticalなリソースを割り当てるときに。
performance-criticalなリソースが分割されていない場合、VMが時間ベースの共有をするために、HW/SWスケジューラが必要になる。この場合、デバイス(GPU)は、hypervisorからのコンテキストスイッチを許容する必要がある。

#### Virtualization Policies for GPU Resources

ドライバはCPUからコマンドバッファにコマンドを書き込む。GPUのRender Engineはそれらのコマンドをフェッチして、実行する。Display EngineはピクセルデータをFrame Bufferからフェッチして、外部モニタに送信する。
Intel Processor Graphicsは、システムメモリをgraphics memoryとして使う、つまりCPUとGPUでメモリを共有する。
システムメモリは、GPUページテーブルでGlobal graphics memoryとLocal graphics memoryにマッピングされる。
4GB(1つ)のGlobal graphics memoryは、Global Page Tableでマッピングされ、GPUとCPU、双方からアクセス可能である。Global graphics memoryは、Frame BufferとCommand Bufferとして利用される。
4GB(複数)のLocal graphics memoryは、Local Page Tableでマッピングされ、Render Engineからのみアクセスできる。
ハードウェアアクセラレーション中の大規模なメモリアクセスは、Local graphics memoryで行われる。

CPUはGPU固有のコマンドでGPUを制御する(producer-consumer model)。graphics driverはGPUコマンドをCommand Bufferへプログラムする。primary bufferとbatch bufferを含む。これは、OpenGLやDirectXといったhigh-level APIに基づいている。その後、GPUはコマンドをフェッチして実行する。
primary buffer (ring buffer)は、他のbatch bufferとchainしている場合がある。batch bufferは主要コマンドを運搬するのに使われる。

GPU-intensiveな3D負荷は、次の４つのインタフェースの利用に分類できる。
1. the Frame Buffer,
2. the Command Buffer,
3. the GPU Page Table Entries (PTEs), which carry the GPU page tables, and
4. the I/O registers, including Memory-Mapped I/O (MMIO) registers, Port I/O (PIO) registers, and PCI configuration space registers for internal state.

ロード時のFrame Bufferへのアクセス（CPUからの頂点と画素の書き込み)と、Run-timeのCommand Bufferへのアクセス(CPUからのGPU制御)頻度が支配的で、他の2つはそれほどでもない。

### High Level Architecture

guest VMは、リソース分割(後述)により、Frame BufferとCommand Bufferに直接アクセスできる。
特権リソース(I/OレジスタやPTE)は、Hypervisorにトラップされて、SOSのGVTデバイスモデルに転送される。
デバイスモデルはi915 interfaceを利用して、物理GPUにアクセスする。

デバイスモデルは、VM間で物理GPUのタイムスロットを共有するために、GPUスケジューラを実装している。
**GPUスケジューラの必要性がまだよく分かっていない。**

ACRN移植のために、ACRN-DM互換なMPT (Mediated Pass-Through) interfaceを実装した。

### Key Techniques

## TCM

## Source code

#＃ 参考文献

[Full GPU Virtualization Solution with Mediated Pass-Through](https://www.usenix.org/node/183932)
[Hardware Specification - PRMs](https://01.org/linuxgraphics/documentation/hardware-specification-prms)
[るくすの日記 Intel GVT-gにみるMediatedパススルーを用いたGPU仮想化](http://rkx1209.hatenablog.com/entry/2015/12/17/213528)
