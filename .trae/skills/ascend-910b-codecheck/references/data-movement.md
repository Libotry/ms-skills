# 数据搬移 CodeCheck 规则 (Data Movement)

涵盖 GM↔UB、UB↔L1、UB↔寄存器之间的数据搬移。

---

## [DM-001] 跨非对齐边界 DataCopy

**严重度**: high

ND tensor 按 shape 切分时,如果边界不对齐到 32B,DataCopy 性能下降甚至出错。

**修复**: 在 tiling 阶段将 shape 对齐到 32B(half dtype 下 16 元素)。

---

## [DM-002] ND2NZ 转换要求

**严重度**: high

C 轴必须对齐到 16(half)或 8(fp32),H 轴对齐到 16。

```cpp
// tiling 时
uint32_t c0 = CeilAlign(C, 16);  // C0 维度
uint32_t h0 = CeilAlign(H, 16);  // H0 维度(部分硬件需要)
```

---

## [DM-003] UB 容量未检查

**严重度**: high

搬入 UB 的数据 + 中间 buffer > 256KB 会越界。

**修复**:
```cpp
if (inputBytes + outputBytes + tempBytes > UB_LIMIT) {
    return;
}
```

---

## [DM-004] 跨 loop 变量复用未清零

**严重度**: medium

UB buffer 在循环间复用时,旧数据可能残留,导致后续计算错误。

**修复**:
- 每次复用前 `AscendC::LocalTensor::SetValue(0)` 或重新 init
- 或使用 TQue 自动管理

---

## [DM-005] GM 直接访问未走 DataCopy

**严重度**: high

AIV 直接 `GlobalTensor<T> g[i]` 会绕过 L2 cache,性能差且可能不正确。

**修复**: 必须先 `DataCopy` 到 UB,再从 UB 处理。

---

## [DM-006] 异步队列深度不足

**严重度**: medium

Double buffer 需要 2 个 in-flight 队列,如果只配 1 个,流水线会 stall。

**修复**:
```cpp
AscendC::TQue<AscendC::TPosition::VECIN, 2> queIn;  // depth=2
```

---

## [DM-007] stride 与 base addr 不一致

**严重度**: medium

ND tensor 的 stride 计算错误,会导致读到错位置。

**修复**: 用 helper 函数计算 stride,不要硬编码。
