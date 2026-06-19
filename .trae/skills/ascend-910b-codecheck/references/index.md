# 规则索引与速查

## 类别 → reference 映射

| 类别 ID | 主题 | reference 文件 |
|---|---|---|
| `INT-` | 整数溢出与类型提升 | `integer-overflow.md` |
| `TIL-` | Tiling 阶段 | `tiling.md` |
| `DM-` | 数据搬移 | `data-movement.md` |
| `ALN-` | 对齐与向量化 | `alignment-vector.md` |
| `SYNC-` | 同步与 event | `sync-event.md` |
| `API-` | Ascend C API 使用 | `api-usage.md` |
| `AIC-` | AIC/AIV 协作 | `aic-aiv.md` |
| `MEM-` | 内存安全 | `memory-safety.md` |
| — | 910B 内部官方规则(待补充) | `official-rules.md` |

## 严重度分级

- **high**:会导致**结果错误 / 越界访问 / 死锁**等问题,必须修。
- **medium**:会导致**性能显著下降 / 资源浪费 / 偶发错误**,强烈建议修。
- **low**:会导致**轻微性能问题 / 代码风格问题**,可选。

## 检查优先级

当代码量大、不能逐行检查时,按以下优先级:

1. **INT-001/002/004** — 整数溢出(用户提到的 uint32 问题)
2. **TIL-001/002/003/004** — Tiling 阶段的 shape/workspace/tail
3. **SYNC-001/002** — 同步缺失(死锁 / 数据竞争)
4. **MEM-001/002/004/006** — 内存安全(越界 / 悬空)
5. **ALN-001** — UB 32B 对齐(性能)
6. **API-001/002** — TQue/TPipe 生命周期
7. 其余

## 修复模板汇总

见各类 reference 末尾的"修复模板"小节。
