# Badcase / Goodcase 样例库

## 样例 1: uint32 乘法溢出(Tiling 阶段最核心的问题)

### BAD

```cpp
// BAD: tiling 中计算 total size
uint32_t batch = 32;
uint32_t channels = 256;
uint32_t height = 224;
uint32_t width = 224;
uint32_t elemSize = sizeof(half);  // 2 bytes

uint32_t totalBytes = batch * channels * height * width * elemSize;
// 32 * 256 = 8192 (uint32 OK)
// 8192 * 224 = 1,835,008 (uint32 OK)
// 1,835,008 * 224 = 411,041,792 (uint32 OK, < 2^32)
// 411,041,792 * 2 = 822,083,584 (< 2^32 OK)
// 但若 batch=64, 64*256*224*224*2 = 1,644,167,168 (< 2^32)
// 若 batch=128, 128*256*224*224*2 = 3,288,334,336 (OVERFLOW!)
```

**触发问题**:
- [INT-001] high — `uint32 * uint32` 乘法链中间结果溢出
- [INT-004] high — 即使最终存到 uint64,中间已溢出

### GOOD

```cpp
// GOOD: 第一步就 cast 到 uint64
uint64_t totalBytes = (uint64_t)batch * channels * height * width * elemSize;

if (totalBytes > UINT32_MAX) {
    // 报错或分块
    return;
}
```

---

## 样例 2: Tiling tail block 处理

### BAD

```cpp
// BAD: 只处理完整的 block,丢掉了 tail
uint32_t N = 100;
uint32_t blockSize = 32;

for (uint32_t i = 0; i < N / blockSize; ++i) {
    processBlock(i * blockSize, blockSize);  // 只处理 96 个元素,丢掉 4 个
}
```

**触发问题**:
- [TIL-004] high — tail 未处理
- [MEM-001] high — 数据丢失

### GOOD

```cpp
// GOOD: 完整处理 full blocks + tail
uint32_t fullBlocks = N / blockSize;  // 3
uint32_t tailCount = N % blockSize;   // 4

for (uint32_t i = 0; i < fullBlocks; ++i) {
    processBlock(i * blockSize, blockSize);
}
if (tailCount > 0) {
    processTail(fullBlocks * blockSize, tailCount);
}
```

---

## 样例 3: 同步缺失

### BAD

```cpp
// BAD: SetFlag 后没 WaitFlag
AscendC::DataCopy(ubTensor, gmTensor);
AscendC::SetFlag<AscendC::HardEvent::MTE2_V>(0);
// 缺少 WaitFlag,Vector 指令可能在 DataCopy 完成前开始
AscendC::Add(outTensor, ubTensor, scalarTensor);
```

**触发问题**:
- [SYNC-001] high — 配对缺失,数据竞争

### GOOD

```cpp
AscendC::DataCopy(ubTensor, gmTensor);
AscendC::SetFlag<AscendC::HardEvent::MTE2_V>(0);
AscendC::WaitFlag<AscendC::HardEvent::MTE2_V>(0);
AscendC::PipeBarrier<PIPE_V>();
AscendC::Add(outTensor, ubTensor, scalarTensor);
```

---

## 样例 4: TQue 生命周期

### BAD

```cpp
// BAD: AllocTensor 后忘记 FreeTensor
auto tensor = que.AllocTensor<half>();
// ... 异常路径提前返回,FreeTensor 漏掉
que.FreeTensor(tensor);
```

**触发问题**:
- [API-002] high — 生命周期泄漏
- [MEM-003] medium — 异常路径未释放

### GOOD (RAII 模式)

```cpp
{
    auto tensor = que.AllocTensor<half>();
    // ... 即使异常,作用域结束会自动调用 FreeTensor
    que.FreeTensor(tensor);
}
```

---

## 样例 5: AIV 直接访问 GM

### BAD

```cpp
// BAD: AIV 直接读 GM
AscendC::GlobalTensor<half> g;
g.SetGlobalBuffer(ptr);
half val = g.GetValue(0);  // 性能极差,且部分硬件不支持
```

**触发问题**:
- [AIC-002] / [DM-005] high

### GOOD

```cpp
// GOOD: 先搬到 UB
AscendC::GlobalTensor<half> g;
g.SetGlobalBuffer(ptr);
AscendC::LocalTensor<half> ub;
AscendC::DataCopy(ub, g, count);
half val = ub.GetValue(0);
```

---

## 910B 特有样例待补充

更多样例待用户提供内部 badcase 后补充。
