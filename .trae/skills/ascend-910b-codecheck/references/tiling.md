# Tiling 阶段 CodeCheck 规则

> **适用范围**:host 端的 tiling 计算、shape 推导、workspace 分配、block 切分。
> **重要性**:tiling 错误往往在运行时才发现,且难以复现。

---

## 规则清单

### [TIL-001] shape 计算溢出

**严重度**: high

**问题描述**:
shape 维度(B/N/C/H/W)通常是 int32 或 uint32,乘法链容易溢出。

```cpp
// BAD
uint32_t total = inputShape[0] * inputShape[1] * inputShape[2] * inputShape[3];
```

**正确做法**:
```cpp
// GOOD
uint64_t total = (uint64_t)inputShape[0] * inputShape[1] * inputShape[2] * inputShape[3];
// 或使用 AscendC 提供的 helper
uint64_t total = AscendC::CeilMul(...) ...
```

---

### [TIL-002] workspace 大小计算

**严重度**: high

**问题描述**:
- workspace 大小 = 输入 + 输出 + 临时 buffer
- 临时 buffer 通常需要 double buffer(2x)
- 一旦 overflow,设备访问会越界

```cpp
// BAD
uint32_t workspaceSize = inputSize + outputSize + 2 * tempSize;

// GOOD
uint64_t workspaceSize = (uint64_t)inputSize + outputSize + 2 * tempSize;
if (workspaceSize > MAX_UB_SIZE) {
    return ACL_ERROR_;  // 必须报错,不能默默截断
}
```

---

### [TIL-003] tile 大小超过 UB 容量

**严重度**: high

**问题描述**:
- 单核 UB 默认 256KB(datasize 不计)
- tileShape * sizeof(dtype) + tempBuffer 必须 < 256KB
- 没检查就传给 device,会触发 runtime error

**正确做法**:
```cpp
uint64_t tileBytes = (uint64_t)tileShape[0] * tileShape[1] * sizeof(half);
uint64_t tempBytes = ...;
if (tileBytes + tempBytes > UB_LIMIT) {
    // 缩小 tile 或报错
}
```

---

### [TIL-004] block 切分非整除时未处理 tail

**严重度**: high

**问题描述**:
N=100,blockSize=32 → 3 个满 block + 1 个 tail block(4 个元素)
如果不单独处理 tail,会**越界读/写**。

**正确做法**:
```cpp
uint32_t fullBlocks = N / blockSize;
uint32_t tailCount = N % blockSize;

for (uint32_t i = 0; i < fullBlocks; ++i) {
    processFullBlock(i, blockSize);
}
if (tailCount > 0) {
    processTail(fullBlocks * blockSize, tailCount);  // 必须单独处理
}
```

---

### [TIL-005] 对齐不足导致 ND2NZ 失败

**严重度**: medium

**问题描述**:
ND → NZ 转换要求 C 轴对齐到 16(half)或 8(float32)。
非对齐时性能下降或失败。

**正确做法**:
```cpp
uint32_t alignedC = CeilAlign(C, 16);  // half dtype
uint32_t paddedH = CeilAlign(H, 16);   // 部分硬件需要
```

---

### [TIL-006] tiling 信息字段顺序/大小写错

**严重度**: medium

**问题描述**:
TilingData 结构体在 host 和 device 端**必须严格一致**,字段顺序、大小、对齐都不能错。

**正确做法**:
- 放在 shared header 文件中
- host 和 device 都 include 同一份
- 加 `static_assert(sizeof(TilingData) == EXPECTED_SIZE);`

---

### [TIL-007] 多核划分不均

**严重度**: medium

**问题描述**:
把 N=100 分给 8 核 → 不是所有核负载一致。
部分核闲、部分核忙,延迟受最慢核影响。

**正确做法**:
```cpp
uint32_t baseCount = N / coreNum;
uint32_t remainder = N % coreNum;
// core 0..remainder-1 处理 baseCount+1,其他处理 baseCount
```

---

### [TIL-008] dtype 不匹配

**严重度**: high

**问题描述**:
- tiling 假设 fp16,实际输入是 fp32
- 计算 sizeof 时用了错误的 dtype

**正确做法**:
```cpp
uint64_t total = (uint64_t)numElements * GetDtypeSize(dtype);  // 不要硬编码 sizeof(half)
```

---

## 检测方法

审查时:
1. **所有 `*` 表达式**:检查是否有 INT-001 同时也是 TIL-001
2. **所有 workspace 计算**:检查溢出检查
3. **所有除法 `%`**:检查 tail 处理
4. **所有 sizeof**:检查是否硬编码 dtype
5. **所有 tiling 结构体**:检查 host/device 是否同一份定义

---

## 910B 硬件约束

- **单核 UB 大小**:默认 **256KB**(datasize 不计)。tile + 中间 buffer 必须 < 此值。
- **Cube 单元 M/N/K 对齐**:M/N 对齐到 16,K 对齐到 16(fp16)/ 8(int8)/ 32(int4)。
- **多核划分**:910B 通常每个 Core 跑独立的 tiling 块,host 端按 coreNum 切分。
- **AI Core 数量**:典型 32 / 48 / 64 核,实际可用依型号变化。
