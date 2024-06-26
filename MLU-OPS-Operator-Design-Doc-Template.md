# mluopR2C FFT 算子开发设计方案

- mluopR2C FFT 文档基本信息

| 算子名称    | mluopR2C FFT     |
| ----------- | ------------ |
| 编制人/日期 | zhaoxiang/2024-5-28 |

- #### 修改记录

| 版本号 | 修订人 | 修订日期  | 修订描述 |
| ------ | ------ | --------- | -------- |
| V1.0   | zhaoxiang    | 2024-5-28 | 首次提交 |

- #### 内容描述

本文档为`mluopR2C FFT`算子的设计文档，包括需求分析、接口设计、方案设计、性能优化记录。

- #### 算子需求 checklist

* 算子接口描述
* 功能描述
* 框架版本 + 对应源码路径
* 需求对应网络
* 网络中用到的规模
* 是否需要支持原位
* 是否需要支持 stride 机制
* 框架单元测试阈值指标（可选）

## 1 需求分析

### 1.1 算子需求分析

该需求分析为框架原生算子实现功能的需求分析，对于框架原生支持但 MLU-OPS™ 当前版本不支持的功能，需要在`1.4算子限制` 章节中显式注明。未明确注明不支持的功能，默认 MLU-OPS™ 全部支持。

example:

| 算子功能简介                                  | R2C FFT                 |
| ----------------------------------------------| ----------------------------------------------------------- |
| 需求来源                                      | PyTorch/TensorFlow/                                          |
| 应用网络                                      | Conformer                                                    |
| 输入数据类型                                  | half, float                                                  |
| 输入标量参数                                  | 标准数据类型                                                  |
| 输入 Shape                                    | input1: [batches, n0]; input2: [batches, 4]                 |
| 输入 Layout                                   | input: ARRAY                                                |
| 输出数据类型                                  | complex_half, complex_float                                  |
| 输出 Shape                                    | complex_half, complex_float                                 |
| 输出 Layout                                   | ARRAY                                                       |
| 模式(可选）                                   |  无                                                          |
| 是否含有 dim/axis 等类似语义的参数且该参数支持负数/其他特殊处理| 有 dim/axis 参数且需要支持负数 / 不含带 dim/axis 语义的参数等<br>(例如 scatter 算子接口中的 dim 参数支持为负，当 dim=-1 时，实际在最低维上做计算) |
| 是否含有 labels/index 等类似语义的参数且该参数支持负数/界外情况/其他特殊处理 | 有 labels 参数，labels 取值在 dim 范围之外时输出为 Nan / 有 index 参数，要求 index 支持负数 / 不含带 labels/index 语义的参数等<br>(例如 sparse_softmax_ce_logits 算子支持 label 取值在 dim 范围之外，advanced_index 算子支持负数 index 等) |
| 是否需要支持原位                               | 是                                                         |
| 是否需要支持 stride 机制                       | 是                                                         |
| 是否需要支持广播                               | 否                                                         |
| 0 元素检查是否直接返回                         |                                                            |
| 其他特殊需求(在线量化，融合，转数提前等，可选)   |  无                                                        |
| 本次开发优先支持的规模/模式                    | (90, 180, 768)                                              |

### 1.2 算子功能和应用场景描述

详细描述算子功能和应用场景，若有公式，详细描述公式。最好能给一个简单的 example。

备注：

1、需要根据算子算法的定义，考虑对边界值进行处理，比如 sign 算子，计算输出输入数据的符号，就需要考虑当输入是 0 的时候怎么处理，是否存在+0，-0 等。

2、index/axis 类的算子，可能存在 index 越界/负数 dim/axis 的情况，调研的时候需要注意。

3、需要说明对 nan/inf 的特殊处理，输入存在 nan/inf 的，分为输入当中只包含 nan、只包含 inf、同时包含 nan 和 inf 的情况。

### 1.3 算子输入输出参数要求

| 参数        | 语义 | 类型（输入/输出） | 支持类型    | 物理布局 | 规模限制 |
| ----------- | ---- | ----------------- | ----------- | -------- | -------- |
| handle      |      | 输入              |             | /        | 无       |
| input1_desc |      | 输入              |             | /        | 无       |
| input1      |      | 输入              | half, float | ARRAY    | 无       |
| input2_desc |      | 输入              |             | /        | 无       |
| input2      |      | 输入              | half, float | ARRAY    | 无       |
| mode        |      | 输入              |             | /        | 无       |
| output_desc |      | 输入              |             | /        | 无       |
| output      |      | 输出              |             | ARRAY    | 无       |

### 1.4 算子限制

详细描述与框架需求相比，算子尚有哪些功能/模式/范围/数据类型/xxx 不支持。
使用表格或分段列出均可。

`注意`：凡是没有在此处列出，但最终被框架检测到的算子限制，均会被视为算子 bug。
在此处列出的限制，算子内做好防呆。

example:

| 限制类型     | 详细说明                                                                                                        |
| ------------ | --------------------------------------------------------------------------------------------------------------- |
| 数据类型限制 | 不支持 input 和 output 同时为 half                                                                              |
| 布局限制     | 不支持 NCHW 的 Layout, 需要明确感知 layout 的算子才需要说明限制, 对于不需要感知 layout 的算子, 不要加额外的防呆 | 
| 规模限制     | output_size <= 384KB                                                                                            |
| 功能限制     | 不支持 xxx 模式/不支持 xx 和 xxx 的模式组合                                                                     |
| 数据范围限制 | half 类型下，数据需满足[xx, xx]范围，否则有精度问题                                                             |
| 原位限制     | 不支持原位                                                                                                      |
| stride 限制  | 不支持 stride 机制                                                                                              |
| 广播限制     | xxx 参数不支持广播                                                                                              |
| xx 限制      | xxx                                                                                                             |

### 1.5 验收标准

#### 1.5.1 精度验收标准

按照[精度验收标准](../MLU-OPS-Accuracy-Acceptance-Standard.md)的要求明确本算子的精度标准。

例如：本算子属于纯 IO 类算子，验收标准为 diff3=0。

#### 1.5.2 性能验收标准

见 [MLU-OPS™ 性能验收标准](../MLU-OPS-Performance-Acceptance-Standard.md)。

## 2 算子接口设计

### 2.1 参考接口

- TensorFlow

```c++
// 给出TensorFlow接口
```

- PyTorch

```c++
// 给出PyTorch接口
```

- cuDNN

```c++
// 给出cuDNN接口
```

### 2.2 接口设计

```c++
1.FFT plan描述符定义，用于描述执行过程所需的信息:
typedef struct mluopFFTStruct *mluopFFTPlan_t;

2.创建FFT plan的接口函数
mluopStatus_t mluopCreateFFTPlan(mluopFFTPlan_t *fft_plan);

3.plan初始化接口，根据输入描述符input_desc、输出描述符output_desc、变换维度rank，变换规模n[], reservespace大小reservespace_size, 以及workspace大小 workspace_size分配所需内存以及提前计算kernel中所需的常数
mluopStatus_t mluopMakeFFTPlanMany(mluopHandle_t handle,
                                 mluopFFTPlan_t fft_plan,
                                 const mluopTensorDescriptor_t input_desc,
                                 const mluopTensorDescriptor_t output_desc,
                                 const int rank,
                                 const int n[],
                                 size_t *reservespace_size,
                                 size_t *workspace_size);
4.给plan绑定reservespace指针
mluopStatus_t mluopSetFFTReserveArea(mluopHandle_t handle,
                                   mluopFFTPlan_t fft_plan,
                                   void *reservespace);
5.执行接口，可利用创建后的plan多次执行相应FFT
mluopStatus_t mluopExecFFT(mluopHandle_t handle,
                         const mluopFFTPlan_t fft_plan,
                         const void *input,
                         const float scale_factor,
                         void *workspace,
                         void *output,
                         int direction);
6.Destroy接口，负责释放plan与reservespace
mluopStatus_t mluopDestroyFFTPlan(mluopFFTPlan_t fft_plan);

```

## 3 实现方案设计

### 3.1 R2C FFT算法介绍

快速傅里叶变换是离散傅里叶变换的快速实现方式，为高性能计算领域最重要的基础算法之一，被广泛应用于信号处理、偏微分方程求解等领域。快速傅里叶变换的使用，使得处理大量数据和复杂算法成为可能，如气象学中，用于分析天气模式和大气数据等。目前复数FFT 算法的实现已经足够成熟，但对于在音频处理、图像处理等需要实数序列进行离散傅里叶变换计算的场景不能完全进行替换。

离散傅里叶变换是傅里叶变换在时域和频域都呈现离散的形式，其定义为公式（1）。

$$X(k)=\sum_{n=0}^{N-1} x(n) W_N^{n k}$$

根据FFT算法原理可以将公式可以利用旋转因子的周期性、对称性以及可约性进行多次递归分解，分解成若干较小规模的DFT进行计算。从而将算法复杂度从![alt text](image.png)降低到O(nlogn)。

如等式（2），是经过一次基-R大小的FFT分解结果。

公式2

在R2C离散傅里叶变换中，除了复数离散傅里叶变换所共有的性质外，还具有复共轭定理的性质。其表示输入均为实数序列时，输出结果具有X(k) 与X(N-k) 是互为共轭复数的关系（实部相等，虚部相反）。因此，为了省略后半部分的冗余数据，R2C FFT中，当输入为N个实数的序列时，输出![alt text](image-1.png)个复数，其中输出序列中第零个复数的虚部为0，若N 为偶数，第N/2个复数的虚部也为0。基于以上的这些内容，可将R2C FFT 蝶形计算kernel 分为了以下三种情况：

1）当k = 0时，其输出序列为X(0)、![alt text](image-2.png)、... 、X(![alt text](image-3.png))，其中![alt text](image-4.png)与![alt text](image-5.png)、![alt text](image-6.png)与X(![alt text](image-7.png))等均满足互为共轭复数的关系，满足实数傅里叶变换的要求，因此只需要计算N/2及其之前的数据即可。并且其输入序列是![alt text](image-8.png)，均为虚部为零的实数，同时![alt text](image-9.png)与输入序列相乘值不变。对此，面对这种输入实数输出为R/2+1个数的情况，我们将进行特殊处理。由于有奇数基和偶数基，因此为了便于统一，当R为奇数时，令R'为R-1，偶数时为R-1。将k=0 带入到上述FFT分解公式即C2C蝶形公式中，可以得到等式（3），该公式被称为为R2C_I蝶形kernel。

2）当k = 1~$$'\frac{N-R}{2R}'$$时，该部分的输出序列为X(k)、X(k+$$'\frac{N}{R}'$$)、X(k+$$'\frac{2N}{R}'$$)、... 、X(k+$$'\frac{(R-1)N}{R}'$$)。其数据在输出序列中不存在互为共轭复数的情况满足复数FFT中的C2C蝶形情况，因此每个数据都是有效的，但在输出序列中只需要X(N/2+1) 及其之前的数据即可。因此，该部分依旧采用复数FFT的蝶形进行计算，然后将X(N/2) 后面的数据进行虚部取反，转换为前半部分未计算的输出数据。



拆分建议分几个维度来写，job间拆分，cluster间拆分，cluster内拆分。

一些基本的拆分原则：

1、拆分逻辑尽量保持均匀，避免出现负载不均衡，避免出现多核芯片单核工作，多核围观的情况出现。

2、尽可能保证拆分不会产生性能很差的 IO pattern，比如 stride 过大的访存，datasize 特别小的访存，非对齐的访存等等。

3、尽量保证拆分的时候不会造成重复的 IO，比如对 conv，如果对 HW 做拆分，由于有 kernel 的存在，hw 的 overlap 部分就会有重复的 IO。

4、拆分一定是和算子整体方案密切相关的，虽然模板把方案分成了几部分，但是这只是提醒大家关注这些重要的指标，并不是一部分一部分分开考虑的，最终方案肯定是拆分，资源分配，指令流水综合权衡得到的结果。

### 3.4 性能优化设计

1、资源分配

| 表项            | 分配策略                                                                       |
| --------------- | ------------------------------------------------------------------------------ |
| NRAM            | 保存 node 数据，分 ping pong 两部分，ping 计算的时候 pong IO             |
| WRAM            | 举例: 保存卷积滤波张量数据                                                     |
| SRAM            | 举例：暂时存储 IO 的输入输出数据，卷积滤波张量最先 load 在 SM 上然后 broadcast |
| DRAM(workspace) | 举例：存储计算中的临时数据，比如 ci 拆分情况下的部分和                         |

2、流水设计
建议这一部分细致的排布之后，能完全理清楚整个实现细节，写代码仅仅将方案用代码的形式进行描述，无论是测试用例设计，还是功能调试，性能分析优化都会十分方便，以下是几个建议：

对计算部分，建议写用的是哪种指令，比如如果是加的话，使用 pool 还是用 atomic_add 还是 cycle_add，一定要写清楚，这样可以暴露一些由于硬件指令不熟悉而影响性能的情况。

对 IO 部分，建议写清楚 IO pattern，因为不同的 IO pattern 是影响性能的关键因素，比如连续 load 数据，读跳 stride 进行访存，写跳 stride 进行对齐。

对流水部分，建议按照模板，可以看到每个时间片都在做什么事情，硬件提供的各个流的实际操作，目的是可以看到是否有些没有数据依赖的操作可以藏起来。

### 3.5 可维护性设计

1、bangc 代码中加入必要的 log 信息，比如输入的规模、数据类型、layout 这些，以及如果出错会导致程序 core dump 的变量，比如 IO 指令的 data_size、dim xyz 的值等，这些信息都是有利于快速定位问题；

2、对每一个函数命名变量命名都有充分的注释；

3、避免魔鬼数字，对于确定的数字尽量使用公共宏来替代。

### 3.6 测试用例设计

- 算子在网络中用到的规模：
  xxx

- 边界 case：
  xxx

其他可根据需要进行补充。算子开发完毕后，补充测试报告链接。

### 3.7 算子防呆检查

- 列出算子需要做的防呆，比如

1、指针为空防呆；

2、0 元素检查防呆，VLOG(5)打印信息，是否返回与框架沟通；

3、涉及 workspace 算子对于 workspace_size 的检查防呆；

4、是否需要对输入输出支持的 dtype、layout 以及 shape 进行防呆；

5、elementwise 算子则需要保证输入输出的每一个维度都要一样；

6、算子存在的自身的相关参数防呆。

主要是列出 4,5,6 防呆内容，方便 review。

## 4 算子性能/精度问题 & 优化记录

### 4.1 当前存在问题的规模说明

只需列出在测试过程中发现的性能/精度异常的规模。

example:

| 提交日期  | 问题规模 | 问题描述 | 是否已修复 |
| --------- | -------- | -------- | ---------- |
| 2022-8-26 | N >= 128 | 性能问题 | 是         |
|           | H >= 128 | 性能问题 | 否         |
|           | W >= 128 | 精度问题 | 否         |
| 2022-8-26 | C >= 128 | 精度问题 | 否         |

### 4.2 已经过优化的规模说明

此项仅填写未在 4.1 中列出的规模，否则填入 4.1.

example:

| 提交日期  | 修复规模 | 修复问题 |
| --------- | -------- | -------- |
| 2022-8-26 | C == 1   | 性能提升 |
| 2022-8-26 | C 不对齐 | 精度修复 |
