---
name: ascend-910b-codecheck
description: 昇腾 910B 算子代码静态检查 Skill。用于审查用户提供的算子代码（Ascend C / AscendCL / Tiling / Kernel）并报告 CodeCheck 问题，给出问题位置、规则依据与修复建议。覆盖整数溢出与类型提升、Tiling 阶段、数据搬移、对齐与向量化、同步、API 使用、AIC/AIV 协作、内存安全等类别。当用户提到 910B / Ascend 算子 / AscendC / Tiling / CodeCheck / 静态检查 / 算子代码审查 / 找 bug 等关键词时触发。代码注释使用英文，说明文字使用中文。
---

# Ascend 910B 算子代码静态检查 Skill

> **适用范围**:昇腾 910B / 910A / 910C 系列芯片上的 Ascend C 算子开发。
> **重要性**:本 Skill 涵盖的 CodeCheck 问题,在 910B 算子开发中是**最高频的 bug 与性能陷阱来源**。

本 Skill **只做静态检查**。用户给一段算子代码(tiling / kernel / host 调度),Skill 按规则扫描并输出结构化问题清单(含位置、规则依据、修复建议)。**不生成新代码**。

---

## 工作流

当用户贴出代码并要求 CodeCheck / 静态检查 / review / 找 bug / 审查 时:

1. **确认代码范围**:判断是 Tiling 代码(host 端)、Kernel 代码(device 端 Ascend C)、还是混合。
2. **读 references 索引**:根据代码类型,按需读取相关 reference 文档:
   - 整数与溢出相关 → `references/integer-overflow.md`
   - Tiling 阶段 → `references/tiling.md`
   - 数据搬移 / UB / GM → `references/data-movement.md`
   - 对齐与向量化 → `references/alignment-vector.md`
   - 同步与 event → `references/sync-event.md`
   - API 使用 → `references/api-usage.md`
   - AIC/AIV 协作 → `references/aic-aiv.md`
   - 内存安全 → `references/memory-safety.md`
3. **逐条扫描**:按 reference 中的规则清单逐项检查代码。
4. **输出报告**:使用统一的输出格式(见下方"输出格式"章节)。

---

## 输出格式

对每一个发现的问题,按以下格式输出:

```
[CATEGORY-ID] 严重度: high/medium/low
位置: <file:line 或 代码片段引用>
问题: <具体描述>
规则依据: <reference 文档路径 + 规则编号>
修复建议: <具体代码修改方案,包含 GOOD 代码示例>
```

**类别 ID 前缀**:
- `INT-` 整数与溢出
- `TIL-` Tiling 阶段
- `DM-` 数据搬移
- `ALN-` 对齐与向量化
- `SYNC-` 同步与 event
- `API-` API 使用
- `AIC-` AIC/AIV 协作
- `MEM-` 内存安全

报告结尾输出一段**总结**,包含:
- 问题总数(按严重度分组)
- 优先级建议(必须修 / 建议修 / 可选修)
- 是否存在潜在的死锁/越界等危险问题

---

## 核心规则快查(V0 摘要)

> 完整规则在 `references/` 下,这里只列最常踩的高频问题。

### 高频问题 TOP 10

1. **[INT-001] high** — `uint32 * uint32` 在 32 位语义下计算后再赋值给 uint64,**先溢出再扩展**,结果错误。必须显式 `(uint64_t)a * b`。
2. **[INT-002] high** — `int32_t` 累加可能溢出;`int32 + int32` 结果必须升为 `int64_t` 后再赋值。
3. **[INT-003] medium** — 移位 `<< 32` 在 uint32 上是 UB,在 uint64 上才安全;遇到 32 位位移应先升位。
4. **[TIL-001] high** — Tiling 计算 `totalSize = N * C * H * W * sizeof(T)` 时,任意维度是 uint32 相乘都会溢出,应**第一步就 cast 到 uint64**。
5. **[TIL-002] high** — workspace / tiling 输出的 buffer size 必须 < UB 容量(默认 256KB),且预留 double buffer 时的对齐。
6. **[DM-001] high** — `DataCopy` 跨非对齐边界时性能/正确性下降,应提前 align。
7. **[ALN-001] high** — UB 上搬入的数据如果不是 32B 对齐,Vector 计算性能显著下降甚至出错。
8. **[SYNC-001] high** — `SetFlag` / `WaitFlag` 配对缺失或顺序错误,会导致流水线死锁或数据竞争。
9. **[API-001] medium** — `TPipe` / `TQue` 初始化顺序错误,导致 buffer 不可用。
10. **[MEM-001] high** — Tail block(剩余元素 < block size)未单独处理,导致越界写。

---

## 910B 硬件相关要点

910B 是当前昇腾主流训练/推理芯片,以下是常见 CodeCheck 需要关注的硬件特性:

- **单核 UB 容量**:默认 256KB(datasize 不计),溢出即越界。
- **32B 对齐要求**:Vector 计算要求 UB 数据 32B 对齐,否则降级到 scalar 路径。
- **AIC/AIV 分离**:Cube 与 Vector 是不同核,必须通过 GM/L1 通信,并正确同步。
- **典型 dtype**:fp16(half)、fp32(float)、int8/uint8、int32。
- **Cube 单元支持的 shape**:M/N/K 必须对齐到 16(fp16)或 8(int8)。

---

## 触发关键词

下列任意一种情况应触发本 Skill:
- 用户提到「910B」「Ascend 算子」「AscendC」「Tiling」「CodeCheck」「静态检查」「算子 review」「找 bug」
- 用户贴一段 C++/Ascend C 代码并问「有没有问题」「帮我审查」「CodeCheck 一下」

---

## 维护说明

- V0: 2026-06-19 创建骨架(910B 版,只做静态检查)
- 待办: 收集典型 badcase/goodcase 到 `assets/`
- 待办: 用户可补充 910B 内部 CodeCheck 规则文档
