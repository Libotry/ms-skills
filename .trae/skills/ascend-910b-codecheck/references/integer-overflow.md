# 整数与溢出检查规则 (Integer Overflow & Type Promotion)

> **适用范围**:昇腾 910B / 910A / 910C 算子代码,涵盖 Tiling(host) 与 Kernel(device) 两端。
> **重要性**:这是 910B CodeCheck 中**最常被命中**的一类问题。

---

## 规则清单

### [INT-001] `uint32 * uint32` 必须显式提升为 uint64

**严重度**: high

**问题描述**:
在 C/C++ 中,`uint32_t a * uint32_t b` 的乘法**先在 32 位无符号整型下计算**,得到 32 位结果(已溢出/截断),即使后续赋值给 `uint64_t`,**低 32 位已经是错误值**。

```cpp
// BAD - 先溢出,再扩展
uint32_t N = 100000;
uint32_t C = 100000;
uint64_t total = N * C;  // 结果是 32 位截断后的值,不是 1410065408 的真实乘积
```

**正确做法**:
```cpp
// GOOD - 任一操作数先提升
uint64_t total = (uint64_t)N * C;
// 或
uint64_t total = static_cast<uint64_t>(N) * C;
```

**典型场景**:
- Tiling 中计算 `totalSize = N * C * H * W * sizeof(T)`
- 计算 `workspaceSize`
- 计算 `blockCount * blockSize`
- 计算 offset / stride

---

### [INT-002] `int32_t` 累加溢出

**严重度**: high

**问题描述**:
```cpp
// BAD
int32_t sum = 0;
for (int i = 0; i < 100000; ++i) {
    sum += a[i];  // int + int = int,大数会溢出
}
```

**正确做法**:
```cpp
// GOOD
int64_t sum = 0;
for (int i = 0; i < 100000; ++i) {
    sum += a[i];
}
```

---

### [INT-003] 移位溢出

**严重度**: medium

**问题描述**:
- `uint32_t x = 1 << 32;` 是 **UB**(C++ standard)
- `int32_t x = 1 << 31;` 是 **UB**(signed overflow)
- `uint64_t x = 1ULL << 63;` 是合法的,但需注意符号

**正确做法**:
- 涉及 32 位以上位移时,**先 cast 到 uint64**
- 涉及 signed 移位时,优先用 unsigned

---

### [INT-004] 乘法运算链的中间结果

**严重度**: high

**问题描述**:
```cpp
// BAD - 中间结果溢出
uint32_t total = N * C * H * W;  // 即使最终存到 uint64,N*C 已溢出
```

**正确做法**:
```cpp
// GOOD - 第一步就提升
uint64_t total = (uint64_t)N * C * H * W;
```

> ⚠️ C++ 中 `(uint64_t)(N * C * H * W)` **仍然错误**,因为 cast 在乘法完成后才执行。

---

### [INT-005] `sizeof()` 与指针运算

**严重度**: medium

**问题描述**:
```cpp
// BAD - count 是 uint32,乘 sizeof 后可能溢出
uint32_t count = 100000;
void* p = malloc(count * sizeof(int));  // 4*100000=400000,这里可能没问题,但习惯要养成
```

**正确做法**:
```cpp
// GOOD - 显式 cast
size_t size = (size_t)count * sizeof(int);
void* p = malloc(size);
```

---

### [INT-006] 隐式窄化转换

**严重度**: medium

**问题描述**:
```cpp
// BAD - 编译告警或截断
uint64_t big = 0xFFFFFFFFFFFFFFFFULL;
uint32_t small = big;  // 截断
```

**正确做法**:
- 不要把 uint64 的结果回退到 uint32
- 如果必须,加显式 cast 并加注释说明意图

---

### [INT-007] 浮点与整数混合

**严重度**: low

**问题描述**:
- 浮点结果转 uint32 时,如果超出范围是 UB
- 浮点比较应避免直接 `==`

**正确做法**:
- 用 `std::nearbyint` / `std::lround` 替代
- 显式 clamp 到合法范围

---

## 检测方法

审查时按以下步骤扫描:

1. **找出所有 `*` 乘法表达式**:看左右操作数类型,只要任一是 uint32/int32 且结果可能 > 2^32,标记。
2. **找出所有 `<<` 移位表达式**:检查移位量是否 >= 类型位数。
3. **找出所有累加/累乘循环**:检查累加器类型是否足够。
4. **找出所有 `malloc/calloc/new` 的 size 参数**:检查 size 计算是否溢出。

---

## 修复模板

### 模板 1: Tiling 中的 size 计算

```cpp
// BEFORE
uint32_t totalSize = batch * channels * height * width * sizeof(half);

// AFTER
uint64_t totalSize = (uint64_t)batch * channels * height * width * sizeof(half);
if (totalSize > UINT32_MAX) {
    // 报错或分块处理
}
```

### 模板 2: Kernel 中的累加

```cpp
// BEFORE
int32_t acc = 0;
for (int i = 0; i < N; ++i) {
    acc += data[i];
}

// AFTER
int64_t acc = 0;
for (int i = 0; i < N; ++i) {
    acc += data[i];
}
// 或根据精度需求,用 float/half 累加
```

---

## 910B 相关提示

> 910B Vector 单元的累加器在大多数指令下是 32 位(`acc`),如果需要 64 位累加,需用 `AscendC::Axpy` / `AscendC::Muls` 等组合,或显式用 scalar 寄存器;不要假设硬件自动扩展。
> 
> 910B 的 Scalar 单元支持 64 位 ALU,大数乘法/移位可以走 scalar,但性能较差。
