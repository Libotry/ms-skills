# RegBase 常见陷阱(6 类)

> **来源**:CANNBot-Skills `ops/ascendc-regbase-best-practice/references/pitfalls/`
> - `common_traps.md`
> - `api_misuse.md`
> - `regbase_vs_membase_confusions.md`
> - `symptom_to_cause.md`

---

## 1. 分层陷阱

| 陷阱 | 风险 | 修复 |
|---|---|---|
| 把整个 kernel 写成一个大 VF 函数 | 不可读,CopyIn→Compute→CopyOut 分层缺失 | 按 Host / Kernel / UB / VF 分层 |
| 把 GM/UB copy 和寄存器计算写在同一层 | ownership 不清 | UB staging 和 VF compute 分开 |
| 在 Process 里直接堆寄存器级数学 | CopyIn→Compute→CopyOut 不可读 | 显式分层 |
| 只写 "RegBase path",没有说明 Host / Kernel / UB / VF 边界 | 审查无法判断层级 | 文档明确每层职责 |

---

## 2. API 陷阱

| 陷阱 | 风险 | 修复 |
|---|---|---|
| 编造 `AscendC::Reg::*` 签名 | 编译失败 | 查 SDK 文档、header 或参考实现 |
| 把 MemBase / LocalTensor API 当成 RegBase VF API | 层级错误,无法编译或语义错 | 回到 RegBase API 白名单 |
| 没有检查 header 或 SDK 文档 | 幻觉 API | 必查 `regbase_api_whitelist.md` |
| API 参数顺序和 mask 位置靠猜 | 编译失败或运行错 | 查 API 白名单 + 真实参考实现 |

### 验证步骤

1. 在白名单或 SDK 文档中确认 API 家族
2. 查真实参考实现是否使用相同模式
3. 确认参数顺序、模板参数、dtype 限制
4. 确认 API 所在层:Host、UB pipeline 还是 VF/寄存器
5. 在 DESIGN.md 的 API 映射表中记录依据

### 构建失败时**不要**做

- ❌ 不要把函数名改成"看起来类似"的名字
- ❌ 不要删掉 mask 让编译先过
- ❌ 不要把 RegBase 路径改成 LocalTensor 路径而不回到设计
- ❌ 不要把参考实现中不同 dtype 的调用直接搬过来

---

## 3. tail / mask 陷阱

| 陷阱 | 风险 | 修复 |
|---|---|---|
| copy 层处理了 tail,但 VF store 没有 mask | tail 写出无效数据 | VF store 显式传 mask |
| mask 使用对齐长度而不是有效长度 | 写出 padded 数据 | 用 valid_count |
| compare mask 和 store mask 混用但语义不同 | 索引错位 | 严格区分 |
| padding 值进入数学结果 | 精度异常 | Duplicate 时用极值,且 math 不参与 padding 位置 |

---

## 4. dtype / precision 陷阱

| 陷阱 | 风险 | 修复 |
|---|---|---|
| fp16/bf16 不说明是否升 fp32 | 精度不一致 | 显式标注 |
| cast 回输出 dtype 的位置不清楚 | 多余 cast / 精度损失 | 显式标注 cast 位置 |
| reduce accumulation dtype 不明确 | 累加溢出或精度不够 | 显式 fp32 累加 |
| quant/dequant 缺少 scale、rounding、saturate 说明 | 量化结果错 | 完整说明 4 个参数 |

---

## 5. 同步陷阱

| 陷阱 | 风险 | 修复 |
|---|---|---|
| 用 `SyncAll` 修本地 stage ordering | 性能浪费 | 改用本地 set/wait 或 BufferID |
| 把 `MaskReg` 当同步 | 编译失败或运行错 | MaskReg 只用于 mask,不用于同步 |
| 没有参考实现就加 `SetFlag` / `WaitFlag` | flag 配对错 | 查参考实现的 flag 模式 |
| cross-core flag 用来补本地 UB 交接 | flag 数量爆 | cross-core 只用于跨核,本地用 BufferID |

---

## 6. 症状 → 原因 速查表

| 症状 | 优先原因 | 下一步检查 |
|---|---|---|
| 编译找不到 API | API 名或 header 不正确 | 查 `api_misuse.md`、`regbase_api_whitelist.md` |
| 模板实例化失败 | dtype route 或 `RegTensor<T>` 不匹配 | 查 API 签名和 dtype 限制 |
| 运行越界 | tiling count、UB offset 或 tail store 错 | 查 `tiling_review_notes.md` |
| 输出全 0 | store 未执行、mask 全 `false`、输出 UB 未入队 | 查数据流和 mask |
| 只有 tail 错 | `MaskReg` 或有效长度错 | 查 `precision_failures.md` |
| 性能异常差 | GM/UB 往返多、barrier 过重、fusion 未生效 | 查 `regbase_performance_practices.md` |
| 代码看似 RegBase 但审查不过 | MemBase / RegBase 混用 | 查 `regbase_vs_membase_confusions.md` |

---

## RegBase vs MemBase 混淆(核心区别)

| 维度 | RegBase | MemBase / LocalTensor 路径 |
|---|---|---|
| compute 核心 | `__VEC_SCOPE__` 内 `RegTensor` / `MaskReg` | `LocalTensor` 上调用 vector API |
| 中间值 | 尽量留在寄存器 | 常物化到 UB tensor |
| 有效通道控制 | `MaskReg` | API count / mask 语义依具体 API |
| 数据流语言 | UB address → 寄存器 → UB address | LocalTensor → LocalTensor |

### 可以共存的部分

RegBase outer shell 可以使用 `TPipe`、`TQue`、`LocalTensor` 做 UB staging。**这不等于 MemBase**。关键在于 **Compute 的数学核心是否进入寄存器级 VF body**。

### 不应混用的部分

- ❌ 在 VF body 内调用只适用于 `LocalTensor` 的 API
- ❌ 用 `LocalTensor` 中间值替代本应短生命周期的 `RegTensor`
- ❌ 设计文档一边说 `asc_vf_call`,一边给出 MemBase API 伪代码
- ❌ 审查时看到 `TQue` 就否定 RegBase

---

## 相关文档

- 原始来源:
  - `ops/ascendc-regbase-best-practice/references/pitfalls/common_traps.md`
  - `ops/ascendc-regbase-best-practice/references/pitfalls/api_misuse.md`
  - `ops/ascendc-regbase-best-practice/references/pitfalls/regbase_vs_membase_confusions.md`
  - `ops/ascendc-regbase-best-practice/references/pitfalls/symptom_to_cause.md`
- RegBase 路线规则:`regbase.md`
- RegBase API 白名单:`regbase-api.md`
