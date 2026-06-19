# 内存安全 CodeCheck 规则 (Memory Safety)

---

## [MEM-001] Tail block 未处理

**严重度**: high

N=100,blockSize=32 → 100/32=3 余 4。如果只循环 3 次,最后 4 个元素丢失。

**修复**: 见 `tiling.md` 中 [TIL-004]。

---

## [MEM-002] UB 越界写

**严重度**: high

写入超过 `TQue` 分配的空间,会覆盖其他 buffer 或硬件寄存器。

**修复**:
- 计算时显式检查 `tileBytes <= queSize`
- 用 `AscendC::LocalTensor` 的 operator[] 时,bounds check 编译期打开

---

## [MEM-003] 跨 loop 变量未重置

**严重度**: medium

```cpp
// BAD
for (int i = 0; i < N; ++i) {
    auto t = que.AllocTensor<half>();
    // use t
    que.FreeTensor(t);
}
// 如果 N=0 时 loop 没执行,t 不会被 free
```

**修复**: 用 RAII 或保证循环至少执行一次(必要时 dummy iteration)。

---

## [MEM-004] GlobalTensor 越界读

**严重度**: high

```cpp
// BAD - shape 变化后没更新 baseOffset
AscendC::GlobalTensor<half> g;
g.SetGlobalBuffer(ptr + baseOffset);
g.GetValue(index);  // index 超 shape 时越界
```

**修复**: 每次循环前更新 offset,带 bounds check。

---

## [MEM-005] UB buffer 残留

**严重度**: medium

复用 UB buffer 时,旧数据可能未被覆盖(如新数据 < 旧数据),导致部分位置读到旧值。

**修复**:
- 用 `Duplicate` / `SetValue(0)` 显式清零
- 或保证新数据完全覆盖旧数据

---

## [MEM-006] 指针悬空

**严重度**: high

`LocalTensor` 在 `FreeTensor` 后不能再使用;`GlobalTensor` 的 base ptr 变更后,旧 reference 失效。

**修复**:
- 不要持有跨 scope 的 tensor 引用
- 用 TQue 自动管理生命周期

---

## [MEM-007] 错误的数据宽度

**严重度**: medium

`half*` 指针访问 `float` 数据,或反之,会读到错位字节。

**修复**: 严格按 dtype 访问,类型不匹配时显式 cast。
