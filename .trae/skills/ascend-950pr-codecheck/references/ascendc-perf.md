# AscendC 性能与精度规范

> **来源**:CANNBot-Skills `ops/ascendc-code-review/references/ascendc-perf.md`
> **适用架构**:arch35 及以上(950PR)
> **侧别**:Kernel
> **性能规范**:PERF-1 ~ PERF-7
> **精度规范**:PREC-1(流水线同步)

---

## 性能优化规范

### [PERF-1] 循环内禁止逐元素操作

**严重度**:高

**问题**:循环内逐元素调用 AscendC API 会导致硬件无法流水线化,性能差。

**错误示例**:
```cpp
for (uint32_t i = 0; i < N; ++i) {
    AscendC::Add(xLocal[i], yLocal[i], 1);  // ❌ count=1 逐元素
}
```

**正确示例**:
```cpp
AscendC::Add(xLocal, yLocal, N);  // ✅ 一次性向量化
```

**检视方法**:循环内 API 调用的 count 是否过小。

---

### [PERF-2] 禁止写死硬件参数

**严重度**:高

**问题**:硬编码 UB 大小、Cube 核数、对齐值会导致跨硬件迁移时出错。

**正确做法**:使用 `PlatformAscendC` 接口动态获取。详见 `hardware-params.md`。

---

### [PERF-3] Double Buffer 使用

**严重度**:中

**原理**:Double Buffer 通过 in-flight 队列(深度 2)实现 CopyIn 与 Compute 流水化。

**使用方式**:
```cpp
AscendC::TQue<AscendC::TPosition::VECIN, 2> inQueue;  // depth=2
pipe.InitBuffer(inQueue, 2, bufferSize);
```

**适用场景**:
- ✅ 单次处理数据量大(超过 UB 一半)
- ✅ Compute 时间较长(可隐藏 CopyIn)
- ❌ 数据量小(单次 UB 就能放下),Double buffer 反而增加复杂度

---

### [PERF-4] PipeBarrier 使用

**严重度**:中

**需要 Barrier 的场景**:
- 跨 pipe 数据依赖(如 V → MTE3)
- 同一 UB 地址切换用途

**可接受的防御性写法**:
```cpp
AscendC::PipeBarrier<PIPE_ALL>();
```

**注意**:过粗的 barrier 会破坏流水线,只在必要时使用。

---

### [PERF-5] 单次搬运量优化

**严重度**:中

**优化建议**:
- 单次搬运数据量越大,带宽利用率越高
- 优先用 32B 对齐 + 较大 blockCount

---

### [PERF-6] 避免 GM 重复读取

**严重度**:中

**错误示例**:在循环内反复 `DataCopy` 同一块 GM。

**正确做法**:进 UB 一次,UB 内重复使用。

---

### [PERF-7] 尾块处理正确性

**严重度**:高

**常见错误**:tail 元素未单独处理,导致越界或算错。

**正确做法**:见 `tiling.md` 中的 tail block 规则。

**受影响的关键参数**:`blockCount`、`blockLen`、`actualCount`。

---

## 精度规范

### [PREC-1] 流水线同步正确性

**严重度**:高

**核心原则**:**CopyIn → Compute → CopyOut 三阶段必须显式同步**。

**三步流程**:
```cpp
// 1. CopyIn
auto xLocal = inQueue.AllocTensor<T>();
AscendC::DataCopy(xLocal, xGm, count);
inQueue.EnQue(xLocal);

// 2. Compute (sync CopyIn 完成)
auto xLocal = inQueue.DeQue<T>();
// ... compute ...
inQueue.FreeTensor(xLocal);

// 3. CopyOut (sync Compute 完成)
AscendC::DataCopy(yGm, yLocal, count);
```

**核心约束**:
- EnQue 与 DeQue 必须配对(同一 buffer 索引)
- 所有 stage 之间必须显式同步

---

## 相关文档

- 原始来源:`ops/ascendc-code-review/references/ascendc-perf.md`
- Tiling 设计:`tiling.md`
- 同步规则:`simt.md`(API-12)
