# 950PR 算子 BAD / GOOD 样例库

供审查报告中"修复建议"部分引用。

---

## 样例 1:uint32 * uint32 乘法溢出(Tiling 阶段)

### BAD

```cpp
// BAD:tiling 中计算 total size
uint32_t batch = 64;
uint32_t channels = 512;
uint32_t height = 224;
uint32_t width = 224;
uint32_t elemSize = sizeof(half);

uint32_t totalBytes = batch * channels * height * width * elemSize;
// 64*512 = 32768
// 32768*224 = 7,340,032
// 7,340,032*224 = 1,644,167,168 (接近 UINT32_MAX)
// *2 = 3,288,334,336 OVERFLOW!
```

**触发**:
- [SEC-2.2] high — 无符号整数运算不回绕
- [TIL-1] high — shape 计算溢出

### GOOD

```cpp
uint64_t totalBytes = (uint64_t)batch * channels * height * width * elemSize;
if (totalBytes > UINT32_MAX) {
    // 报错或分块
}
```

---

## 样例 2:硬编码 UB 大小(950PR 特有)

### BAD

```cpp
// BAD:硬编码 256KB,910B 时代的值
uint64_t ubSize = 256 * 1024;
if (tileBytes > ubSize) {
    return;
}
```

**触发**:
- [HW-1] high — 硬编码 UB 大小
- [PERF-2] high — 禁止写死硬件参数

### GOOD

```cpp
auto platform = platform_ascendc::PlatformAscendC(context->GetPlatformInfo());
uint64_t ubSize = platform.GetUbSize();  // 950PR 返回 253952
if (tileBytes > ubSize) {
    return;
}
```

---

## 样例 3:RegBase 与 MemBase 混用(950PR 特有)

### BAD

```cpp
// DESIGN.md:声明使用 RegBase 路线
// 但 kernel 中混合使用 RegTensor 和原生 DataCopy

auto reg_a = RegTensor::Load(ub_a, count, mask);
reg_a.Compute(mask);

AscendC::Add(ub_b, ub_c, count);  // ❌ 在 RegBase 路径上用 MemBase API
```

**触发**:
- [RB-1] high — RegBase 与 MemBase/SIMD 路线混用

### GOOD

```cpp
auto reg_a = RegTensor::Load(ub_a, count, mask);
reg_a.Compute(mask);
RegTensor::Store(reg_a, ub_out, mask.GetValidCount());
// 全部走 RegBase API,无 MemBase vector API
```

> ⚠️ RegBase outer shell 可使用 `TPipe`、`TQue`、`LocalTensor` 做 UB staging,这是允许的。但 Compute 的数学核心必须进入 VF body。

---

## 样例 4:RegBase tail/mask 未处理(950PR 特有)

### BAD

```cpp
auto reg_a = RegTensor::Load(gm_ptr, block_size);
reg_a.Compute();                                       // ❌ 没传 mask
RegTensor::Store(reg_a, gm_out, reg_a.Size());         // ❌ 写入整个寄存器宽度
```

**触发**:
- [RB-3] high — 寄存器级计算边界

### GOOD

```cpp
auto mask = MaskReg::UpdateMask(valid_count);
auto reg_a = RegTensor::Load(gm_ptr, block_size, mask);
reg_a.Compute(mask);
RegTensor::Store(reg_a, gm_out, mask.GetValidCount()); // 只写回有效元素
```

---

## 样例 5:SIMT 编译期线程数三处不一致(950PR 特有)

### BAD

```cpp
// ❌ 错误:线程数从 tiling 数据动态获取
int32_t threadNum = static_cast<int32_t>(tilingData_->threadNum);
Simt::VF_CALL<OpComputeSimt<T>>(Simt::Dim3(threadNum), args...);
```

**触发**:
- [SIMT-10] high — 编译期线程数三处不一致
- [SIMT-5/6] high — VF 线程数不在 TilingData

### GOOD

```cpp
// ✅ 正确:三处一致 + constexpr
constexpr uint32_t THREAD_NUM = 512;

__simt_vf__ __aicore__ LAUNCH_BOUND(THREAD_NUM) inline void OpComputeSimt(...);

Simt::VF_CALL<OpComputeSimt<T>>(Simt::Dim3(THREAD_NUM), args...);
```

---

## 样例 6:`__simt_vf__` 内调用 `__simd_callee__`(950PR 特有)

### BAD

```cpp
__simd_callee__ inline int64_t CalcOffset(int64_t base, int64_t stride, int64_t idx) {
    return base + stride * idx;
}

__simt_vf__ __aicore__ inline void OpSimt(...) {
    int64_t offset = CalcOffset(0, 1, i);  // ❌ 跨模型调用
}
```

**触发**:
- [SIMT-7] high — `__simt_vf__` 内调用 `__simd_callee__`

### GOOD

```cpp
__simt_callee__ inline int64_t CalcOffset(int64_t base, int64_t stride, int64_t idx) {
    return base + stride * idx;
}

__simt_vf__ __aicore__ LAUNCH_BOUND(512) inline void OpSimt(...) {
    int64_t offset = CalcOffset(0, 1, i);  // ✅ 同一模型内调用
}
```

---

## 样例 7:DataCopy 未 32 字节对齐

### BAD

```cpp
AscendC::DataCopy(xLocal, xGm, 4);  // ❌ cols=4 (16 bytes),未对齐
```

**触发**:
- [API-3] high — DataCopy 32 字节对齐要求

### GOOD

```cpp
AscendC::DataCopyExtParams copyParams;
copyParams.blockLen = cols * sizeof(float);  // 单位:字节
AscendC::DataCopyPad(xLocal, xGm, copyParams);
```

---

## 样例 8:动态内存分配(950PR Kernel 禁用)

### BAD

```cpp
std::vector<int> vec;       // ❌ Kernel 侧禁止
int* ptr = new int[10];     // ❌
int* arr = malloc(100);     // ❌
```

**触发**:
- [API-7] high — 禁止动态内存分配
- [SEC-5.2] high — 资源泄露防护

### GOOD

```cpp
constexpr uint32_t SIZE = 1024;
pipe.InitBuffer(inQueue, 2, SIZE);  // ✅ 静态分配
```

---

## 样例 9:GlobalTensor SetValue/GetValue

### BAD

```cpp
for (uint32_t i = 0; i < size; i++) {
    output.SetValue(i, input.GetValue(i));  // ❌ 逐元素,性能极差
}
```

**触发**:
- [API-1] high — 禁止使用 GlobalTensor::SetValue/GetValue

### GOOD

```cpp
AscendC::DataCopyPad(output, input, copyParams, padParams);  // ✅ 批量
```

---

## 样例 10:tail block 未处理

### BAD

```cpp
uint32_t N = 100;
uint32_t blockSize = 32;
for (uint32_t i = 0; i < N / blockSize; ++i) {
    processBlock(i * blockSize, blockSize);  // ❌ 只处理 96 个,丢掉 4 个
}
```

**触发**:
- [TIL-3] high — tail block 处理
- [PERF-7] high — 尾块处理正确性

### GOOD

```cpp
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

## 样例 11:4:2 稀疏支持(950PR 已移除)

### BAD

```cpp
// 950PR 上启用 4:2 稀疏
if (platform.IsSupportSparsity()) {
    tiling.sparseMode = 4;  // ❌ 950PR 不支持
}
```

**触发**:
- [HW-2] high — 使用 4:2 稀疏支持

### GOOD

```cpp
auto platform = platform_ascendc::PlatformAscendC(context->GetPlatformInfo());
NpuArch arch = platform.GetCurNpuArch();
if (arch != NpuArch::DAV_3510 && platform.IsSupportSparsity()) {
    tiling.sparseMode = 4;
}
```

---

## 样例 12:SIMT DCache 不足(950PR 特有)

### BAD

```cpp
// ❌ 错误:DCache 设置太小,触发编译校验报错
constexpr uint64_t DCACHE_SIZE = 16 * 1024;  // < 32KB
uint64_t ubsize = 256 * 1024;
context->SetLocalMemorySize(ubsize - DCACHE_SIZE);
```

**触发**:
- [SIMT-15] high — DCache 不足 32KB

### GOOD

```cpp
// ✅ 正确:DCache >= 32KB
constexpr uint64_t DCACHE_SIZE = 128 * 1024;  // 128KB
uint64_t ubsize = 256 * 1024;
context->SetLocalMemorySize(ubsize - DCACHE_SIZE);
```

---

## 样例 13:std:: 计算函数(Kernel 侧禁用)

### BAD

```cpp
#include <cmath>
float val = std::sqrt(x);  // ❌ Kernel 侧不支持 std::
```

**触发**:
- [API-2] high — 禁止使用 std:: 计算函数

### GOOD

```cpp
AscendC::Sqrt<T>(dstLocal, srcLocal, count);
```

---

## 相关文档

- 完整规则索引:`references/index.md`
- 各 reference 中的 BAD/GOOD 示例
