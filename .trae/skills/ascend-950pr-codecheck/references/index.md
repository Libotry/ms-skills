# 950PR CodeCheck 规则索引与速查

## 类别 → reference 映射

| 类别 ID | 主题 | 来源 | 规则数 | reference 文件 |
|---|---|---|---|---|
| `SEC-` | C++ 安全编码 | CANNBot `cpp-secure.md` | **32 条** | `cpp-secure.md` |
| `API-` | AscendC API 最佳实践 | CANNBot `ascendc-api.md` | **10 条** | `ascendc-api.md` |
| `PERF-` | 性能规范 | CANNBot `ascendc-perf.md` | 7 条 | `ascendc-perf.md` |
| `PREC-` | 精度规范(流水线同步) | CANNBot `ascendc-perf.md` | 1 条 | `ascendc-perf.md` |
| `RB-` | RegBase 路线 | CANNBot `regbase-review-checks.md` | **5 条** | `regbase.md` |
| `RB-API-` | RegBase API 使用 | CANNBot `regbase-best-practice/api/` | 5 条 | `regbase-api.md` |
| `RB-TRAP-` | RegBase 常见陷阱 | CANNBot `pitfalls/` | 6 类 | `regbase-traps.md` |
| `SIMT-` | SIMT 编程模型 | CANNBot `simt-constraints.md` | **15 条** | `simt.md` |
| `MC2-` | MC2/MoE 通信 | CANNBot `mc2-specific.md` | **19 条** | `mc2.md` |
| `HW-` | 硬件参数 | CANNBot `npu-hardware-params.md` | 6 条 | `hardware-params.md` |
| `TIL-` | Tiling 设计 | 综合 | 10 条 | `tiling.md` |

**总规则数**:**116 条**(包含 950PR RegBase + SIMT + MC2 三大新特性)

---

## 严重度分级

- **high**(红线):会导致结果错误 / 越界 / 死锁 / 编译失败,**必须修**。
- **medium**:会导致性能显著下降 / 偶发错误,**强烈建议修**。
- **low**:轻微问题 / 代码风格,**可选**。

---

## 路线识别与规则加载顺序

审查时按以下顺序判定:

1. **C++ 安全编码(`cpp-secure.md`)** — 100% 强制,任何代码都加载
2. **路线识别** — 根据代码信号判断走哪条路线
3. **加载对应路线规则**:
   - RegBase → `regbase.md` + `regbase-api.md` + `regbase-traps.md`
   - SIMT → `simt.md`
   - 标准 AscendC → `ascendc-api.md` + `ascendc-perf.md`
   - Tiling → `tiling.md` + `cpp-secure.md`
   - MC2 → `mc2.md` + `simt.md`(如果涉及同步)
4. **硬件参数核对(`hardware-params.md`)** — 950PR 真值表

---

## 950PR 特有红线(15 条,命中即必须修)

按命中频率和严重度排序:

### RegBase 路线

1. `[RB-1] high` — RegBase 与 MemBase/SIMD 路线混用
2. `[RB-2] high` — 凭函数名猜测 RegBase API
3. `[RB-3] high` — 寄存器级 tail/mask 未处理
4. `[RB-4] high` — 参考实现约束不一致
5. `[RB-5] high` — 架构来源未标注

### SIMT 编程

6. `[SIMT-1] high` — VF 参数 size 超过 28x32bit
7. `[SIMT-2] high` — 用 `cce_parallel` 启动 SIMT
8. `[SIMT-7] high` — `__simt_vf__` 内调用 `__simd_callee__`
9. `[SIMT-10] high` — 编译期线程数三处不一致
10. `[SIMT-15] high` — DCache 不足 32KB

### AscendC API

11. `[API-1] high` — 使用 `GlobalTensor::SetValue/GetValue`
12. `[API-7] high` — 动态内存分配
13. `[API-3] high` — DataCopy 未 32 字节对齐

### 硬件参数

14. `[HW-1] high` — 硬编码 UB 256KB(950PR 真值 248KB)
15. `[HW-2] high` — 使用 4:2 稀疏(950PR 不支持)

---

## 检查优先级

当代码量大时按以下顺序:

1. **C++ 安全红线(SEC-1.x ~ SEC-4.x)** — 必须 100% 通过
2. **950PR 硬件参数(HW-*)** — 硬编码数字
3. **RegBase/SIMT 路线一致性(RB-1, SIMT-7)** — 混用编译失败
4. **同步规则(SIMT-10, API-12)** — 死锁/数据竞争
5. **API 黑名单(API-1, API-2, API-7)** — 性能/功能
6. **tail/mask 处理(RB-3, PERF-7)** — 数据丢失
7. **性能规范(PERF-1 ~ PERF-7)** — 性能优化
8. 其余

---

## 950PR vs 910B 关键差异

| 项 | 910B (DAV_2201) | 950PR (DAV_3510) |
|---|---|---|
| UB | 192 KB | **248 KB** |
| L0C | 128 KB | **256 KB**(翻倍) |
| L1 | — | **512 KB**(新增) |
| 4:2 稀疏 | 支持 | **不支持** |
| 同步机制 | SetFlag/WaitFlag | **BufferID(新增)** |
| 编程模型 | MemBase/SIMD | **+ RegBase + SIMT + NDDMA + CCU** |
| 数据格式 | FP16/BF16/FP32/INT8 | **+ FP8/MXFP8/MXFP4/HiF8** |
| Cube 核数(Server) | 24 | **32** |

---

## 数据来源声明

本 Skill 所有规则的权威来源:

- **CANNBot-Skills 仓库**:`https://gitcode.com/cann/cannbot-skills`
- **关键目录**:
  - `ops/ascendc-code-review/references/` — 主检视规则
  - `ops/npu-arch/references/` — 硬件参数真值
  - `ops/ascendc-regbase-best-practice/references/` — RegBase 深度
  - `ops/ascendc-simt-best-practices/` — SIMT 最佳实践

如发现本 Skill 与 CANNBot-Skills 最新规则不一致,**以 CANNBot-Skills 为准**。
