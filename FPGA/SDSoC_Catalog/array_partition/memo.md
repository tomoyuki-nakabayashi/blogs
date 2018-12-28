# Array Partitioning

[Array Partitioning](https://github.com/Xilinx/SDSoC_Examples/tree/master/cpp/getting_started/array_partition)

> array_partition/	This example shows how to use array partitioning to improve performance of a hardware function

Key Concepts
- Hardware Function Optimization
- Array Partitioning
Keywords
- #pragma HLS ARRAY_PARTITION
- complete

## Makefile

### default parameters

Platformのデフォルトはzcu102。

```makefile
# FPGA Board Platform (Default ~ zcu102)
PLATFORM := zcu102
```

ユーティリティディレクトリから、ライブラリをインクルードしている。

```makefile
# Points to Utility Directory
COMMON_REPO = ../../../
ABS_COMMON_REPO = $(shell readlink -f $(COMMON_REPO))

# Include Libraries
include $(ABS_COMMON_REPO)/libs/sds_utils/sds_utils.mk
```

### Makefile Data

リソースの指定は普通。

```makefile
SRC_DIR := src
OBJECTS += \
$(pwd)/$(BUILD_DIR)/main.o \
$(pwd)/$(BUILD_DIR)/matmul.o
```

SDSのオプションは少し調べないとわからないな。

```makefile
# SDS Options
HW_FLAGS := 
ifneq ($(TARGET), cpu_emu)
	HW_FLAGS += -sds-hw matmul_partition_accel matmul.cpp -sds-end
endif
```

> -sds-hw と -sds-end オプションはペアで使用されます。
> -sds-hw オプションでは、単一の関数記述のハードウェアへの移動が開始されます。
> -sds-end オプションでは、その関数のコンフィギュレーションの詳細なリストが終了されます。

`matmul_partition_accel`は、関数名。

`cpp/getting_started/array_partition/matmul.h`

```c
// Pragma below Specifies sds++ Compiler to Generate a Programmable Logic Design
// Which has Direct Memory Interface with DDR and PL.  
#pragma SDS data zero_copy(in1[0:mat_dim*mat_dim], in2[0:mat_dim*mat_dim], out[0:mat_dim*mat_dim])
void matmul_partition_accel(int *in1,  // Read-Only Matrix 1
                            int *in2,  // Read-Only Matrix 2
                            int *out,  // Output Result
                            int mat_dim); //  Matrix Dim (assumed only square matrix)
```

コンパイルフラグの整備。

```makefile
# Compilation and Link Flags
IFLAGS := -I.
CFLAGS = -Wall -O3 -c
CFLAGS += -I$(sds_utils_HDRS)
CFLAGS += $(ADDL_FLAGS)
LFLAGS = "$@" "$<"
```

### SDS driver

まぁ、普通ですな。

```makefile
SDSFLAGS := -sds-pf $(PLATFORM) \
		-target-os $(TARGET_OS)
ifeq ($(VERBOSE), 1)
SDSFLAGS += -verbose

# SDS Compiler
CC := sds++ $(SDSFLAGS)
endif
```

コンパイラのドライブも普通な感じですな。

```makefile
.PHONY: all
all: $(BUILD_DIR)/$(EXECUTABLE)

$(BUILD_DIR)/$(EXECUTABLE): $(OBJECTS)
	mkdir -p $(BUILD_DIR)
	@echo 'Building Target: $@'
	@echo 'Trigerring: SDS++ Linker'
	cd $(BUILD_DIR) ; $(CC) -o $(EXECUTABLE) $(OBJECTS) $(EMU_FLAGS)
	@echo 'SDx Completed Building Target: $@'
	@echo ' '

$(pwd)/$(BUILD_DIR)/%.o: $(pwd)/$(SRC_DIR)/%.cpp
	@echo 'Building file: $<'
	@echo 'Invoking: SDS++ Compiler'
	mkdir -p $(BUILD_DIR)
	cd $(BUILD_DIR) ; $(CC) $(CFLAGS) -o $(LFLAGS) $(HW_FLAGS)
	@echo 'Finished building: $<'
	@echo ' '
ifeq ($(TARGET), cpu_emu) 
	@echo 'Ignore the warning which states that hw function is not a HW accelerator but has pragma applied for cpu_emu mode'
	@echo ' '
endif
```

`make check`すると、エミュレータが走る。

```makefile
# Check Rule Builds the Sources and Executes on Specified Target
check: all
ifneq ($(TARGET), hw)

    ifeq ($(TARGET_OS), linux)
	    cp $(ABS_COMMON_REPO)/utility/emu_run.sh $(BUILD_DIR)/
	    cd $(BUILD_DIR) ; ./emu_run.sh $(EXECUTABLE)
    else
	    cd $(BUILD_DIR) ; sdsoc_emulator -timeout 500
    endif

else
	$(info "This Release Doesn't Support Automated Hardware Execution")
endif
```

`utility/emu_run.sh`を見ると、sdsoc_emulatorを呼び出している。
sdsoc_emulatorは、qemuベースのエミュレータで、RTLのシミュレーションもできる(ボードがXilinx作成物の場合)。

```sh
#!/usr/bin/env bash
if [ -f "$PWD/_sds/init.sh" ] 
then
rm -rf $PWD/_sds/emulation/sd_card
sdsoc_emulator -no-reboot |tee emulator.log
else
cat > "init.sh" <<EOT
#!/bin/sh
/mnt/$1
reboot
EOT
echo $PWD/_sds/init.sh >> "_sds/emulation/sd_card.manifest"
mv init.sh _sds
sdsoc_emulator -no-reboot |tee emulator.log
fi
```

### sds_utils.mk

少し寄り道をして、`sds_utils.mk`の内容を確認する。

`libs/sds_utils/sds_utils.mk`

```makefile
sds_utils_HDRS:=$(ABS_COMMON_REPO)/libs/sds_utils/
```

ヘッダファイルの場所を定義しているだけみたい。ヘッダの中身を大したこと書いていないな。

`libs/sds_utils/sds_utils.h`

```cpp
#ifndef SDS_UTILS_H_
#define SDS_UTILS_H_
#include <stdint.h>
#include "sds_lib.h"
namespace sds_utils {
    class perf_counter
    {

    private:
        uint64_t tot, cnt, calls;
    
    public:
        perf_counter() : tot(0), cnt(0), calls(0) {};
        inline void reset() { tot = cnt = calls = 0; }
        inline void start() { cnt = sds_clock_counter(); calls++; };
        inline void stop() { tot += (sds_clock_counter() - cnt); };
        inline uint64_t avg_cpu_cycles() {return (tot / calls); };
    };
}
#endif 
```

## C++ソースコード解析

まずは、動作確認用に作られているソフトウェア実行の関数を確認する。
`matrix[M][M]`のデータを乗算しているだけ。

```cpp
// Software Matrix Multiplication
void matmul(int *C, int *A, int *B, int M) {
    for (int k = 0; k < M; k++) {
        for (int j = 0; j < M; j++) {
            for (int i = 0; i < M; i++) {
                C[k * M + j] += A[k * M + i] * B[i * M + j];
            }
        }
    }
}
```

同様の関数を高位合成の対象としたのが、下記の`matmul_partition_accel`になる。
なんでも良いけど、API合わせておいて欲しい…。引数の順番が違うんですが。

```cpp
// Pragma below Specifies sds++ Compiler to Generate a Programmable Logic Design
// Which has Direct Memory Interface with DDR and PL.  
#pragma SDS data zero_copy(in1[0:mat_dim*mat_dim], in2[0:mat_dim*mat_dim], out[0:mat_dim*mat_dim])
void matmul_partition_accel(int *in1,  // Read-Only Matrix 1
                            int *in2,  // Read-Only Matrix 2
                            int *out,  // Output Result
                            int mat_dim); //  Matrix Dim (assumed only square matrix)
```

`cpp/getting_started/array_partition.cpp`

```cpp
int main(int argc, char **argv) {
    bool test_passed = true;
    static const int columns = MAX_SIZE;
    static const int rows = MAX_SIZE;

    // Allocate buffers using sds_alloc
    int *A    = (int *) sds_alloc(sizeof(int) * columns * rows);
    int *B    = (int *) sds_alloc(sizeof(int) * columns * rows);
    int *C    = (int *) sds_alloc(sizeof(int) * columns * rows);
```

まずは、`sds_alloc`でメモリ空間を確保する。

[SDSoC 開発環境ヘルプ メモリ割り当て](https://japan.xilinx.com/html_docs/xilinx2018_2/sdsoc_doc/osz1524548239815.html)

> sds_alloc/sds_free を使用してアクセラレータ専用のメモリを割り当てた方がデータが物理的に隣接したメモリに割り当てられて格納されるので、メモリへの読み出しおよび書き込みが速くなり、パフォーマンスを向上できます。

なるほど、連続領域にメモリが確保される、と。

データの初期化をして、ソフトウェア実装、ハードウェア実装を順番に呼び出す。
関数の実行にかかった時間を計測している。

```cpp
    for (int i = 0; i < NUM_TIMES; i++) 
    {
         // Data Initialization
        for(int i = 0; i < columns * rows; ++i) {
            A[i] = i;
            B[i] = i + i;
            gold[i] = 0;
            C[i] = 0;
        }

        sw_ctr.start();
        //Launch the Software Solution
        matmul(gold, A, B, columns);
        sw_ctr.stop();

        hw_ctr.start();
        //Launch the Hardware Solution
        matmul_partition_accel(A, B, C, columns); 
        hw_ctr.stop();

        // Compare the results of hardware to the simulation
        test_passed = verify(gold, C, columns * rows);
    }
```

`matmul_partition_accel`についているpragmaの意味を調べる。

```cpp
#pragma SDS data zero_copy(in1[0:mat_dim*mat_dim], in2[0:mat_dim*mat_dim], out[0:mat_dim*mat_dim])
void matmul_partition_accel(int *in1,  // Read-Only Matrix 1
                            int *in2,  // Read-Only Matrix 2
                            int *out,  // Output Result
                            int mat_dim); //  Matrix Dim (assumed only square matrix)
```

[pragma SDS data copy](https://japan.xilinx.com/html_docs/xilinx2018_2_xdf/sdaccel_doc/pragma-sds-data-copy-csh1504034362862.html)

> COPY プラグマを使用すると、データがホスト プロセッサ メモリからハードウェア関数にコピーされます。最適なデータ ムーバーによりデータ転送が実行されます。詳細は、『SDSoC 環境プロファイリングおよび最適化ガイド』 (UG1235) の「システム パフォーマンスの向上」を参照してください。
> ZERO_COPY を使用すると、ハードウェア関数が AXI4 マスター バス インターフェイスを介して共有メモリからデータに直接アクセスします。

なるほど。データムーバーを使うかどうか、が変わってくるということ。

`COPY`を利用すると、データ転送するコードに置き換えられる。

```cpp
#pragma SDS data copy(A[0:size*size], B[0:size*size])
void foo(int *A, int *B, int size) {
...
}

void _p0_foo_0(int *A, int *B, int size)
{
    ...
    cf_send_i(&(_p0_swinst_foo_0.A), A, (size*size) * 4, &_p0_request_0);
    cf_receive_i(&(_p0_swinst_foo_0.B), B, (size*size) * 4, &_p0_request_1);
    ...
}
```

`ZERO_COPY`を利用すると、 **参照** を送信するコードに置き換えられる。

```cpp
#pragma SDS data zero_copy(A[0:size*size], B[0:size*size])
void foo(int *A, int *B, int size) {
...
}

void _p0_foo_0(int *A, int *B, int size)
{
    ...
    cf_send_ref_i(&(_p0_swinst_foo_0.A), A, (size*size) * 4, &_p0_request_0);
    cf_receive_ref_i(&(_p0_swinst_foo_0.B), B, (size*size) * 4, &_p0_request_1);
    ...
}
```

> cf_send_ref_i および cf_receive_ref_i 関数は配列の参照またはポインターのみをアクセラレータに転送し、アクセラレータは直接メモリにアクセスします。

高位合成関数の本体は次のような感じ。

```cpp
void matmul_partition_accel(int *in1,  // Read-Only Matrix 1
                            int *in2,  // Read-Only Matrix 2
                            int *out,  // Output Result
                            int mat_dim)  // Matrix Dimension 
{               
    // Local memory is implemented as BRAM memory blocks
    int A[MAX_SIZE][MAX_SIZE];
    int B[MAX_SIZE][MAX_SIZE];
    int C[MAX_SIZE][MAX_SIZE];
    //partitioning Array A and B  
    #pragma HLS ARRAY_PARTITION variable=A dim=2 complete
    #pragma HLS ARRAY_PARTITION variable=B dim=1 complete
    
    // Burst reads on input matrices from DDR memory
    // Burst read for matrix A and B
    // Multiple memory interfaces are supported by default in SDSoC
    // It is possible to fetch both A and B concurrently. 
    readA:
    for (int itr = 0, i = 0, j = 0; itr < mat_dim * mat_dim; itr++, j++) {
    #pragma HLS PIPELINE
    #pragma HLS LOOP_TRIPCOUNT min=c_size*c_size max=c_size*c_size
        if (j == mat_dim) { j = 0; i++; }
        A[i][j] = in1[itr];
        B[i][j] = in2[itr];
    }

    // By Default VHLS create single Memory with two ports for each local buffer
    // which allows maximum two read/write from buffer per clock.
    // Due to this restriction, lowest loop of mmmult can be unroll by 2 times only.
    // 
    // However Partition gives instruction to VHLS Complier to split a large array
    // into small-small memory which allow user to get multiple concurrent accesses.
    //
    // To completely unroll the lowest loop of Mmult, A buffer is completely 
    // partitioned for 2nd dimension, and B buffer is completely partitioned
    // for 1st dimension. Which eventually will improve the overall performance of 
    // matrix multiplication. 
    arraypart1: for (int i = 0; i < mat_dim; i++) {
    #pragma HLS LOOP_TRIPCOUNT min=c_size max=c_size
        arraypart2: for (int j = 0; j < mat_dim; j++) {
        #pragma HLS LOOP_TRIPCOUNT min=c_size max=c_size
        #pragma HLS PIPELINE
            int result = 0;
            arraypart3: for (int k = 0; k < MAX_SIZE; k++) {
                result += A[i][k] * B[k][j];
            }
            C[i][j] = result;
        }
    }

    // Burst write from output matrices to DDR memory
    // Burst write from matrix C
    writeC:
    for (int itr = 0, i = 0, j = 0; itr < mat_dim * mat_dim; itr++, j++) {
    #pragma HLS PIPELINE
    #pragma HLS LOOP_TRIPCOUNT min=c_size*c_size max=c_size*c_size
        if (j == mat_dim) { j = 0; i++; }
        out[itr] = C[i][j];
    }
}
```

## build

ビルドして、どのような回路が生成されるか確認する。

```
$ make all TARGET=hw
```

ビルドログ

```
```

## 参考

- [SDSoC 開発環境ヘルプ ハードウェア関数オプション](https://japan.xilinx.com/html_docs/xilinx2018_2/sdsoc_doc/hardware-function-options-icf1504034400618.html)
- [sdscc/sds++ コンパイラのコマンドおよびオプション](https://japan.xilinx.com/support/documentation/sw_manuals_j/xilinx2015_2_1/sdsoc_doc/topics/compiler-linker/concept_compiler_linker_intro.html)
