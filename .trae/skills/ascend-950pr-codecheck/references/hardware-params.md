# 950PR 硬件参数真值表

> **来源**:CANNBot-Skills `ops/npu-arch/references/npu-hardware-params.md`(官方 INI 真值表)
> **架构代号**:DAV_3510
> **产品系列**:Ascend950DT / Ascend950PR

> ⚠️ **核心原则**:**运行时必须用 `PlatformAscendC` 接口获取硬件参数,禁止硬编码**。本表仅供审查代码中是否有硬编码错误。

---

## 950PR 通用参数(各子型号一致)

| 参数 | INI 字段 | 值 |
|------|---------|:---:|
| NpuArch | `NpuArch` | 3510 |
| L1 | `l1_size` | 512 KB (524288) |
| L0C | `l0_c_size` | 256 KB (262144) |
| **UB** | `ub_size` | **248 KB (253952)** |
| BT | `bt_size` | 4 KB (4096) |
| **稀疏 4:2** | `sparsity` | **0(不再支持 4:2)** |
| 核心类型 | `core_type_list` | `CubeCore,VectorCore` |
| **核间关系** | — | **CubeCore : VectorCore = 1 : 2** |

---

## 子型号变化参数(典型 SKU)

| 参数 | Ascend950PR PCIE | Ascend950PR Server |
|------|------------------|--------------------|
| Cube 核数 | 28 | **32** |
| 频率 | 1.65 GHz | 1.65 GHz |
| L2 | 112 MB | 128 MB |
| Memory | 112 GB | 128 GB |

---

## 950PR 与 910B 关键差异对比

| 参数 | 950PR (DAV_3510) | 910B (DAV_2201) | 差异 |
|---|---|---|---|
| **UB** | **248 KB** | 192 KB | **+56 KB(+29%)** |
| **L0C** | **256 KB** | 128 KB | **翻倍(+100%)** |
| **L1** | **512 KB** | — | **新增** |
| **BT** | **4 KB** | 1 KB | +3 KB |
| **稀疏 4:2** | **不支持** | 支持 | **移除(重大)** |
| Cube 核数(Server) | **32** | 24 | +8 |
| 频率 | 1.65 GHz | 1.8 GHz | -0.15 GHz |
| L2(Server) | 128 MB | 192 MB | -64 MB |
| Memory(Server) | 128 GB | 64 GB | 翻倍 |

---

## [HW-1] 硬编码 UB 大小

**严重度**:high

**问题描述**:代码中硬编码 UB 大小为 256KB 是 910B 时代的常见错误,950PR 真值是 **248KB**。

**错误示例**:
```cpp
// BAD: 硬编码 256KB(910B 时代的值)
uint64_t ubSize = 256 * 1024;
if (totalBytes > ubSize) {
    return;
}
```

**正确做法**:
```cpp
// GOOD: 通过 PlatformAscendC 获取
auto platform = platform_ascendc::PlatformAscendC(context->GetPlatformInfo());
uint64_t ubSize = platform.GetUbSize();  // 950PR 返回 253952
if (totalBytes > ubSize) {
    return;
}
```

---

## [HW-2] 使用 4:2 稀疏支持

**严重度**:high

**问题描述**:950PR 已**移除 4:2 稀疏支持**(`sparsity = 0`),代码中如使用 `SparseMatmul` 或类似 4:2 优化 API 会直接报错。

---

## [HW-3] 硬编码 L0C 大小

**严重度**:medium

**问题描述**:L0C 从 128KB 翻倍到 256KB。代码中如硬编码 128KB 会低估 L0C 容量,导致不必要的 double buffer 切换。

---

## [HW-4] 硬编码 Cube 核数

**严重度**:medium

**问题描述**:950PR Server 32 核、PCIE 28 核,与 910B 不同。tiling 计算 `coreNum` 时应通过 `GetCoreNumAic()` / `GetCoreNumVector()` 获取。

---

## [HW-5] 硬编码 1:1 核关系

**严重度**:medium

**问题描述**:950PR CubeCore : VectorCore = **1:2**(与 910B 一致,但不同子型号可能变化)。代码中假设 1:1 关系会导致 sync 逻辑错误。

---

## [HW-6] 硬编码 910B 数据格式

**严重度**:medium

**问题描述**:950PR 新增 FP8 / MXFP8 / MXFP4 / HiF8 等数据格式,如果代码中只检查 fp16/bf16/fp32/int8 可能漏掉新格式分支。

---

## 950PR 新增硬件特性

### 新增数据格式

| 格式 | 说明 | Cube MMAD | CV |
|---|---|:---:|:---:|
| FP8 (E4M3/E5M2) | 8 位浮点 | ✅ | 待确认 |
| MXFP8 | Microscaling FP8 | ✅ | — |
| MXFP4 | Microscaling FP4 | ✅ | — |
| HiF8 | 华为 FP8 | ✅ | — |
| FP16 / BF16 | 16 位浮点 | ✅ | ✅ |
| FP32 | 32 位浮点 | ✅ | ✅ |
| INT8 / UINT8 | 8 位整型 | ✅ | ✅ |
| INT32 | 32 位整型 | — | ✅ |

### 新增指令通路

| 通路 | 说明 |
|------|------|
| **L0C → UB** | CV 直通通路,允许 L0C 结果直接搬到 UB |
| **UB → L1** | CV 直通通路,允许 UB 数据回灌 L1 |
| **SSBuffer** | 跨核 SSBuffer 消息机制 |
| **BufferID 同步** | 替代 set/wait 强配对的同步机制 |

### 新增编程模型

| 模型 | 说明 | 适用场景 |
|---|---|---|
| **SIMT** | 类似 CUDA SIMT,通过 VF 协处理器 | 控制流密集、动态分支 |
| **SIMD-Regbase** | 寄存器级编程 | 高性能向量化 |
| **NDDMA** | 高维 DMA | 多维 tensor 搬移 |
| **CCU 通算融合** | 通信与计算融合 | MoE / 分布式算子 |

---

## 获取运行时架构参数的接口

```cpp
#include "utils/tiling/platform/platform_ascendc.h"

auto ascendcPlatform = platform_ascendc::PlatformAscendC(context->GetPlatformInfo());
NpuArch npuArch = ascendcPlatform.GetCurNpuArch();         // DAV_2201 / DAV_3510 / ...
platform_ascendc::SocVersion socVer = ascendcPlatform.GetSocVersion();
```

**关键接口**:

| 接口 | 用途 |
|---|---|
| `GetCurNpuArch()` | 获取 NpuArch,失败返回 `DAV_RESV` |
| `GetSocVersion()` | 获取 SocVersion,失败返回 `RESERVED_VERSION` |
| `GetUbSize()` | 获取 UB 字节数 |
| `GetL1Size()` | 获取 L1 字节数 |
| `GetL0CSize()` | 获取 L0C 字节数 |
| `GetCoreNumAic()` | 获取 Cube 核数 |
| `GetCoreNumVector()` | 获取 Vector 核数 |
| `GetCoreMemSize()` | 获取 Memory 容量 |
| `aclrtGetDeviceInfo(...)` | 通用设备信息 |

---

## 相关文档

- 原始来源:`ops/npu-arch/references/npu-hardware-params.md`
- 950PR 微架构细节:`ops/npu-arch/references/npu-arch-guide.md`
- SIMT 架构:`ops/npu-arch/references/simt-arch-guide.md`
