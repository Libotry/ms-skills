# 对齐与向量化 CodeCheck 规则 (Alignment & Vectorization)

---

## [ALN-001] UB 数据 32B 对齐

**严重度**: high

UB 上搬入的数据如果不是 32B 对齐,Vector 计算单元无法满载,性能下降 50% 以上。

**修复**:
- tiling 阶段保证 tile shape 对齐到 32B(half dtype = 16 元素)
- 用 `AscendC::CeilAlign` 处理

---

## [ALN-002] Mask 寄存器长度错误

**严重度**: high

`AscendC::Add(x, y, mask, count)` 中 mask 长度必须等于实际处理元素数,否则会读到未初始化内存。

**修复**:
```cpp
uint32_t actualCount = min(blockSize, remaining);
AscendC::Add(dst, src0, src1, actualCount);  // count 必须是真实长度,不是 blockSize
```

---

## [ALN-003] vector 对齐与 scalar 混用

**严重度**: medium

vector 操作要求所有操作数在同一 buffer 且对齐,中间插入 scalar 处理会破坏流水线。

**修复**: 把 scalar 处理提到 vector 循环外。

---

## [ALN-004] uint8/int8 对齐浪费

**严重度**: low

uint8 dtype 也能 32B 对齐(一次 32 元素),不要按 16 元素处理浪费带宽。

---

## [ALN-005] fp16 / fp32 混算

**严重度**: high

fp16 和 fp32 混合计算会触发硬件自动转换,性能与精度都受影响。

**修复**: 显式 cast 到目标 dtype,或确认硬件路径支持。

---
