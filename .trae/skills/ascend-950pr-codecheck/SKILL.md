---
name: ascend-950pr-codecheck
description: 昇腾 950PR / Ascend950DT (DAV_3510) 算子代码静态检查 Skill。基于 CANNBot-Skills 仓库的官方检视规范（ascendc-code-review + ascendc-regbase-best-practice + npu-arch）。审查 950PR 上的算子代码（Ascend C / RegBase / SIMT / Tiling / Kernel）并报告 CodeCheck 问题，给出问题位置、规则依据与修复建议。覆盖 950PR 新特性（RegBase 路线、SIMT 编程、NDDMA、CCU、新数据格式 FP8/MXFP）、C++ 安全编码、AscendC API、RegBase vs MemBase 混淆、tail/mask 处理、同步规则、Tiling 设计、MC2 通信等领域。当用户提到 950PR / DAV_3510 / Ascend950 / RegBase / RegTensor / SIMT 算子 / 算子 CodeCheck / 静态检查 等关键词时触发。代码注释使用英文，说明文字使用中文。
---

# Ascend 950PR 算子代码静态检查 Skill

> **适用范围**:昇腾 950PR / Ascend950DT 系列芯片(架构代号 `DAV_3510`)
> **规则来源**:基于 CANNBot-Skills 官方检视规范:
> - `ops/ascendc-code-review`(RegBase 5 条 + AscendC API 10 条 + 性能 7 条 + 精度 + MC2 19 条 + C++ 安全 32 条)
> - `ops/npu-arch`(硬件参数真值表)
> - `ops/ascendc-regbase-best-practice`(API 白名单 + 6 类常见陷阱)
> - `ops/ascendc-simt-best-practices`(SIMT 编程模型)
>
> **重要性**:本 Skill 涵盖的规则来自 CANNBot 官方,经华为 CANN 团队维护,在 950PR 算子开发中是**最高频的 bug / 性能 / 架构错误来源**。

本 Skill **只做静态检查**。用户给一段 950PR 算子代码(tiling / kernel / RegBase / SIMT),Skill 按规则扫描并输出结构化问题清单(含位置、规则依据、修复建议)。**不生成新代码**。

---

## 工作流

当用户贴出代码并要求 CodeCheck / 静态检查 / review / 找 bug / 审查 时:

### Step 1: 识别代码路线

根据代码信号判断走的哪条路线,优先加载对应 reference:

| 路线 | 触发信号 | 加载 reference |
|---|---|---|
| **RegBase**(950PR 新增) | `RegTensor`, `MaskReg`, `asc_vf_call`, `__simd_vf__`, `__simd_callee__`, `__VEC_SCOPE__`, `AscendC::Reg::`, `AscendC::MicroAPI::`, `UpdateMask`, `LoadAlign`, `StoreAlign`, `LoadDist`, `StoreDist`, `CastTrait`, `arch35/`, `DAV_3510`, `ArchVersion::V3510` | `regbase.md`, `regbase-api.md`, `regbase-traps.md` |
| **SIMT**(950PR 新增) | `__simt_vf__`, `__simt_callee__`, `Simt::VF_CALL`, `Simt::Dim3`, `LAUNCH_BOUND` | `simt.md` |
| **标准 AscendC(MemBase)** | `LocalTensor`, `pipe.InitBuffer`, `DataCopy`, `DataCopyPad`, `AllocTensor`, `FreeTensor`, `CrossCoreSetFlag`, `CrossCoreWaitFlag`, `GetValue`, `SetValue` | `ascendc-api.md`, `ascendc-perf.md` |
| **Tiling 侧(host)** | 无 AscendC::,只在 host 调度 | `tiling.md`, `cpp-secure.md` |
| **MC2 通信算子** | `Hccl`, `AlltoAll`, `MC2`, `MoE`, `expert`, `EP`, `TP` | `mc2.md` |

### Step 2: 加载 C++ 安全编码底线

**无论哪条路线,都要加载** `cpp-secure.md`(32 条安全红线,所有代码 100% 强制)。

### Step 3: 加载硬件参数真值

根据架构代际(DAV_3510 vs DAV_2201),核对代码中所有硬编码数字(UB/L1/L0C 大小、核数、对齐值)是否与 950PR 真值一致 → 加载 `hardware-params.md`。

### Step 4: 逐条扫描,按统一格式输出报告

---

## 输出格式

对每一个发现的问题,按以下格式输出:

```
[CATEGORY-ID] 严重度: high/medium/low
路线: RegBase / SIMT / MemBase / Tiling / MC2 / 通用
位置: <file:line 或 代码片段引用>
问题: <具体描述>
规则依据: <reference 文档路径 + 规则编号,以及 CANNBot 原始规则来源>
修复建议: <具体代码修改方案,包含 GOOD 代码示例>
```

**类别 ID 前缀**:
- `SEC-` C++ 安全编码(`cpp-secure.md`,32 条)
- `API-` AscendC API 最佳实践(`ascendc-api.md`)
- `PERF-` 性能规范(`ascendc-perf.md`)
- `PREC-` 精度规范(`ascendc-perf.md`)
- `RB-` RegBase 路线(`regbase.md`,5 条官方)
- `RB-API-` RegBase API 使用(`regbase-api.md`)
- `RB-TRAP-` RegBase 常见陷阱(`regbase-traps.md`)
- `SIMT-` SIMT 编程模型(`simt.md`)
- `MC2-` MC2 通信算子(`mc2.md`,19 条官方)
- `HW-` 硬件参数(`hardware-params.md`)
- `ARCH-` 架构判断(`hardware-params.md`)

报告结尾输出一段**总结**,包含:
- 问题总数(按严重度分组、按路线分组)
- 优先级建议(必须修 / 建议修 / 可选修)
- 是否存在红线问题(红线 = 必须修)

---

## 950PR 硬件参数速查(基于 `npu-hardware-params.md` 真值表)

| 参数 | 950PR (DAV_3510) | 910B (DAV_2201) | 差异 |
|---|---|---|---|
| **UB** | **248 KB** | 192 KB | +56 KB |
| **L1** | **512 KB** | — | 新增 |
| **L0C** | **256 KB** | 128 KB | **翻倍** |
| **BT** | **4 KB** | 1 KB | +3 KB |
| **稀疏 4:2** | **不支持** | 支持 | 移除 |
| **Cube:Vector 核数比** | **1:2** | 1:2 | 一致 |
| **Cube 核数(Server)** | **32** | 24 | +8 |
| **Cube 核数(PCIE)** | **28** | — | — |
| **频率** | 1.65 GHz | 1.8 GHz | 略降 |
| **L2(Server)** | 128 MB | 192 MB | 减小 |
| **L2(PCIE)** | 112 MB | — | — |
| **Memory(Server)** | 128 GB | 64 GB | 翻倍 |

**新数据格式(950PR 新增)**:FP8 / MXFP8 / MXFP4 / HiF8 Cube MMAD

**新增编程通路(950PR 新增)**:
- **SIMT 编程模型**:`__simt_vf__` 协处理器
- **SIMD-Regbase**:寄存器级编程
- **NDDMA**:高维 DMA
- **CCU 通算融合**:三种通信范式
- **CV 直通**:L0C→UB、UB→L1、SSBuffer 消息
- **BufferID 同步**:替代 set/wait 强配对

> ⚠️ **运行时必须用 `PlatformAscendC` 接口获取硬件参数,禁止硬编码。**

---

## 950PR 红线问题(Top 15)

按严重度和命中频率排序,以下问题在 950PR 上出现即"必须修":

### RegBase 路线相关(950PR 核心)

1. **[RB-1] high** — RegBase 与 MemBase/SIMD 路线混用(同一算子内)
2. **[RB-2] high** — 凭函数名猜测 RegBase API,未查白名单(构建失败/幻觉 API)
3. **[RB-3] high** — 寄存器级 tail/mask 未处理(只 copy 层处理 tail,VF store 无 mask)
4. **[RB-4] high** — 参考实现约束不一致(伪代码与 RegBase 流水线不匹配)
5. **[RB-5] high** — 架构来源未标注(RegBase-native vs compat 混用)

### SIMT 编程模型(950PR 新增)

6. **[SIMT-1] high** — 线程数用运行时变量(`Simt::Dim3(threadNum)`),必须 `constexpr`
7. **[SIMT-2] high** — 用 `cce_parallel` 启动 SIMT 函数,必须用 `AscendC::Simt::VF_CALL`
8. **[SIMT-3] high** — `__simt_vf__` 内调用 `__simd_callee__` 函数(类型错)
9. **[SIMT-4] high** — VF 参数 size 超过 28x32bit
10. **[SIMT-5] high** — 编译期线程数 `LAUNCH_BOUND` 与 `Simt::Dim3` 不一致

### AscendC API 与性能

11. **[API-1] high** — 使用 `GlobalTensor::SetValue/GetValue`(逐元素访问,性能极差)
12. **[API-7] high** — 使用 `new/malloc/std::vector` 动态分配(AI Core 无动态内存)
13. **[API-3] high** — `DataCopy` 数据未 32 字节对齐(应改 `DataCopyPad`)
14. **[HW-1] high** — 硬编码 UB 大小为 256KB(950PR 真值是 248KB)
15. **[HW-2] high** — 假设 4:2 稀疏支持(950PR 已移除)

---

## 触发关键词

下列任意一种情况应触发本 Skill:
- 用户提到「950PR」「Ascend950」「DAV_3510」「arch35」「950DT」
- 用户提到「RegBase」「RegTensor」「__simd_vf__」「SIMT 算子」「NDDMA」「CCU」
- 用户提到「Ascend 算子」「AscendC」「Tiling」「CodeCheck」「静态检查」「算子 review」「找 bug」
- 用户贴一段 C++/Ascend C 代码并问「有没有问题」「帮我审查」「CodeCheck 一下」

---

## 与 910B 版本的差异

本 Skill **专注 950PR**;910B 版本的 Skill 在 `ascend-910b-codecheck`。

主要差异:
- 950PR 支持 RegBase/SIMT 编程模型,910B 不支持
- 950PR UB 大小不同(248KB vs 192KB)
- 950PR L0C 大小不同(256KB vs 128KB)
- 950PR 移除 4:2 稀疏支持
- 950PR 同步机制从 set/wait 改为 BufferID

---

## 维护说明

- V1: 2026-06-19 基于 CANNBot-Skills 仓库官方规则创建
- 数据源:`https://gitcode.com/cann/cannbot-skills`
- 待办:持续同步 CANNBot-Skills 仓库的规则更新
