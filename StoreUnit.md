# StoreUnit模块验证文档
## 文档概述

本文档描述了StoreUnit模块的结构与功能，并根据功能给出测试点参考，方便测试的参与者理解测试需求，编写相关测试用例。StoreUnit包括其五级流水线处理流程、支持的三种类型store指令（标量、向量、非对齐）、接口设计与信号交互逻辑。该模块用于执行Store类指令的地址生成与处理，是Load/Store流水线中的关键组成部分。文档面向有一定CPU微架构基础的读者，建议理解TLB、PMP、RAW违例、StoreQueue、LSQ等相关概念。


## 术语说明

| 名称 | 定义 |
| ------- | ---|
| TLB（Translation Lookaside Buffer） | 地址转换旁路缓冲器，用于虚拟地址到物理地址的快速转换 |
| PMP（Physical Memory Protection）	| 物理内存访问权限检查机制 |
| RAW（Read After Write）违例	| 写后读违例，表示一个load指令读取尚未写入的store数据 |
| LSQ（Load Store Queue） | 处理Load/Store指令顺序及依赖检查的数据结构 |
| StoreQueue | 专门用于跟踪Store指令的FIFO队列 |

## 前置知识

在阅读本模块文档前，建议掌握以下知识：
1. RISC-V中的Store指令地址流水线与语义；
2. 虚拟地址与物理地址的映射机制；
3. 异常与中断系统，尤其是地址不对齐异常、页面异常；
4. 向量指令中store语义与mask机制。


<mrs-functions>

## 功能说明

### 1. 支持标量Store指令

#### 1.1 地址计算与对齐检查（S0）
- 输入：来自store issue队列的指令
- 处理：
  - 计算虚拟地址（VA）
  - 检查地址是否对齐，并标记`storeAddrMisaligned`
  - 发起TLB请求（`io.tlb.req`）
  - 生成mask，输出到StoreQueue
- 输出：`st_mask_out`

#### 1.2 TLB与RAW违例处理（S1）
- 输入：TLB响应（`io.tlb.resp`）
- 处理：
  - 将TLB响应写入StoreQueue
  - 向LoadQueue发出RAW检测
  - 如果TLB命中，则向后端发出issue信息（`io.issue`）

#### 1.3 异常与反馈更新（S2）
- 输入：PMP检查结果（`io.pmp`）
- 处理：
  - 异常信号更新至ROB
  - 将TLB miss反馈发送至RS（`feedback_slow`）
  - 将其他信息写入LSQ

#### 1.4 同步一拍（S3）
- 处理：用于与RAW违例检测对齐同步

#### 1.5 发起写回（S4）
- 输出：
  - 普通store结果写回后端（`stout`）

### 2. 支持向量Store指令

#### 2.1 接收vsSplit请求（S0）
- 来自VecStIn通道（`io.vecstin`），高优先级，无需地址计算

#### 2.2 计算向量偏移（S1）
- 计算vecVaddrOffset、vecTriggerMask
- 向LSQ写入store数据

#### 2.3 忽略feedback（S2）
- 不需要向后端RS发送反馈信息

#### 2.4 发起向量写回（S4）
- 输出：向量store结果写回（`vecstout`）

### 3. 支持非对齐Store指令

#### 3.1 非对齐请求处理（S0）
- 来自MisalignBuffer（`misalign_stin`），优先级最高

#### 3.2 判断是否进入MisalignBuffer（S2）
- 如果是新请求且未对齐但不跨16B边界，则进入MisalignBuffer（`misalign_enq`）
- 如果来自MisalignBuffer：
  - TLB miss则重发（`misalign_stout`）
  - 否则发起写回

</mrs-functions>


## 常量说明 【可选项】 需列出模块中所有可配置参数及其物理意义


| 常量名 | 常量值 | 解释 |
| ------ | ------ | ---- |
| VAddrBits | 39 | 虚拟地址位宽 |
| XLEN | 64 | 数据位宽 |
| VLEN | 512 | 向量长度 |
| RAWTotalDelayCycles | 1 | RAW违例处理延迟周期 |


## 接口说明 【必填项】 详细解释各种接口的含义、来源

### 输入接口
| 信号名 | 方向 | 位宽 | 描述 |
|--------|------|------|------|
| clock | input | 1 | 时钟信号 |
| reset | input | 1 | 复位信号 |
| io.redirect | input | 11 | 异常重定向信息（含robIdx等） |
| io.csrCtrl | input | CustomCSRCtrlIO | CSR控制信号集 |
| io.stin | input | 100 | 普通标量store请求输入 |
| io.vecstin | input | Decoupled(VecPipeBundle) | 向量store请求输入 |
| io.misalign_stin | input | Decoupled(LsPipelineBundle) | 非对齐store请求输入 |
| io.prefetch_req | input | Decoupled(StorePrefetchReq) | 预取store请求 |
| io.vec_isFirstIssue | input | 1 | 向量store是否第一次发射 |
| io.fromCsrTrigger | input | CsrTriggerBundle | CSR触发器调试控制输入 |

### 输出接口
| 信号名 | 方向 | 位宽 | 描述 |
|--------|------|------|------|
| io.issue | output | Valid(MemExuInput) | 发出到后端的issue信息 |
| io.misalign_stout | output | Valid(SqWriteBundle) | 非对齐store请求写回响应 |
| io.tlb | mixed | TlbRequestIO | 地址转换请求与响应接口（含req/resp） |
| io.dcache | mixed | DCacheStoreIO | DCache store接口（含req/resp/kill等） |
| io.pmp | input | PMPRespBundle | PMP权限检查反馈 |
| io.lsq | output | Valid(LsPipelineBundle) | 向LoadStoreQueue发送普通store结果 |
| io.lsq_replenish | output | LsPipelineBundle | 向LSQ发送异常或补充信息 |
| io.feedback_slow | output | Valid(RSFeedback) | 向调度器发送TLB miss等信息 |
| io.prefetch_train | output | Valid(LsPrefetchTrainBundle) | 向SMS训练路径反馈 |
| io.s1_prefetch_spec | output | 1 | Stage1是否为预取投机路径 |
| io.s2_prefetch_spec | output | 1 | Stage2是否为预取投机路径 |
| io.stld_nuke_query | output | Valid(StoreNukeQueryBundle) | RAW违例检测接口 |
| io.stout | output | Decoupled(MemExuOutput) | 写回结果（标量store） |
| io.vecstout | output | Decoupled(VecPipelineFeedbackIO) | 写回结果（向量store） |
| io.st_mask_out | output | Valid(StoreMaskBundle) | 输出store掩码mask给StoreQueue使用 |
| io.debug_ls | output | DebugLsInfoBundle | debug信息输出 |
| io.misalign_enq | output | MisalignBufferEnqIO | 向MisalignBuffer发起入队或撤销请求 |
| io.s0_s1_valid | output | 1 | Stage0或Stage1是否活跃，用于RAW检测同步 |

## 接口时序 【可选项】 对复杂接口，提供波形图的案例

### 案例1

请在这里填充时序案例1

### 案例2

请在这里填充时序案例2

## 测试点总表 (【必填项】针对细分的测试点，列出表格)

实际使用下面的表格时，请用有意义的英文大写的功能名称和测试点名称替换下面表格中的名称

<mrs-testpoints>

| 序号 |  功能名称 | 测试点名称      | 描述                  |
| ----- |-----------------|---------------------|------------------------------------|
| 1.1.1 | SCALAR_STORE | TEST_ADDR_ALIGN | 检查地址对齐时是否正确发出TLB请求 |
| 1.2.1 | SCALAR_STORE | TEST_TLB_HIT | 检查DTLB命中时是否向后端发起issue |
| 1.3.1 | SCALAR_STORE | TEST_FEEDBACK | 检查PMP检查结果是否正确反馈至RS |
| 2.1.1 | VECTOR_STORE | TEST_VEC_OFFSET | 检查向量指令计算偏移是否正确 |
| 2.4.1 | VECTOR_STORE | TEST_VEC_WRITEBACK | 检查vecstout是否正确写回 |
| 3.2.1 | MISALIGN_STORE | TEST_MISALIGN_ENTRY | 检查未对齐请求是否进入MisalignBuffer |
| 3.2.2 | MISALIGN_STORE | TEST_MISALIGN_REPLAY | 检查TLB miss后是否正确重发 |

</mrs-testpoints>

## 附录【可选项】

此部分用于存放正文的补充内容，以便进行扩展和详细说明，旨在使文档格式更加清晰，排版更加合理。
