# Tiling 设计检查规则(950PR)

> **来源**:综合 `ops/ascendc-tiling-design`、`ops/aiss-tiling-solver`、910B 通用规则
> **侧别**:Host 端

---

## [TIL-1] shape 计算溢出(950PR 重点)

**严重度**:high

**问题**:shape 维度通常是 uint32 / int32,乘法链容易溢出。

**正确做法**:
```cpp
// ✅ GOOD:第一步就 cast 到 uint64
uint64_t total = (uint64_t)batch * channels * height * width;
```

详见 `cpp-secure.md` 中 [SEC-2.1] 和 [SEC-2.2]。

---

## [TIL-2] UB 容量检查使用真值(950PR 特有)

**严重度**:high

**问题**:硬编码 UB 256KB 是 910B 时代的常见错误,**950PR 真值是 248KB**。

**正确做法**:
```cpp
// ✅ GOOD:通过 PlatformAscendC 获取
auto platform = platform_ascendc::PlatformAscendC(context->GetPlatformInfo());
uint64_t ubSize = platform.GetUbSize();
uint64_t tileBytes = (uint64_t)tileShape[0] * tileShape[1] * sizeof(half);
uint64_t tempBytes = ...;
if (tileBytes + tempBytes > ubSize) {
    return;
}
```

详见 `hardware-params.md`。

---

## [TIL-3] tail block 处理

**严重度**:high

```cpp
// ✅ GOOD
uint32_t fullBlocks = N / blockSize;
uint32_t tailCount = N % blockSize;
for (uint32_t i = 0; i < fullBlocks; ++i) {
    processBlock(i * blockSize, blockSize);
}
if (tailCount > 0) {
    processTail(fullBlocks * blockSize, tailCount);
}
```

---

## [TIL-4] TilingData 结构与 host/device 一致性

**严重度**:high

**问题**:TilingData 结构体在 host 和 device 端必须严格一致,字段顺序、对齐、大小都不能错。

**正确做法**:
- 放在 shared header 文件
- host 和 device 都 include 同一份
- 加 `static_assert(sizeof(TilingData) == EXPECTED_SIZE);`
- 加 `static_assert(offsetof(TilingData, fieldN) == EXPECTED_OFFSET);`

---

## [TIL-5] VF 线程数不在 TilingData

**严重度**:high(SIMT 特有)

详见 `simt.md` 中 [SIMT-5] 和 [SIMT-6]。VF 线程数是编译期 constexpr,禁止放 TilingData。

---

## [TIL-6] SetLocalMemorySize 计算(950PR SIMT)

**严重度**:high(SIMT 特有)

**950PR SIMT 模式下**:
```cpp
constexpr uint64_t DCACHE_SIZE = 128 * 1024;  // 定义为 128KB
uint64_t ubsize = 256 * 1024;
context->SetLocalMemorySize(ubsize - DCACHE_SIZE);
```

DCache 必须 >= 32KB,否则编译校验报错。详见 `simt.md` [SIMT-15]。

---

## [TIL-7] 4:2 稀疏支持检测

**严重度**:high(950PR 特有)

950PR 已**移除 4:2 稀疏支持**。Tiling 中如启用 4:2 路径会运行时报错。

```cpp
// ❌ BAD:950PR 上启用 4:2
if (platform.IsSupportSparsity()) {
    tiling.sparseMode = 4;  // 950PR 不支持
}

// ✅ GOOD:动态判断
auto platform = platform_ascendc::PlatformAscendC(context->GetPlatformInfo());
NpuArch arch = platform.GetCurNpuArch();
if (arch != NpuArch::DAV_3510 && platform.IsSupportSparsity()) {
    tiling.sparseMode = 4;
}
```

---

## [TIL-8] 多核划分不均

**严重度**:medium

```cpp
// ✅ GOOD
uint32_t baseCount = N / coreNum;
uint32_t remainder = N % coreNum;
// core 0..remainder-1 处理 baseCount+1,其他处理 baseCount
```

---

## [TIL-9] dtype 不匹配

**严重度**:high

```cpp
// ✅ GOOD:不要硬编码 sizeof(half)
uint64_t total = (uint64_t)numElements * GetDtypeSize(dtype);
```

---

## [TIL-10] RegBase 路线下 tiling count 与 UB offset 对齐

**严重度**:high(RegBase 特有)

详见 `regbase.md` 中 [RB-3]。tiling 计算的当前块大小必须与 RegTensor Load 的 block_size 一致。

---

## 相关文档

- 950PR 硬件参数:`hardware-params.md`
- SIMT 模式:`simt.md`
- RegBase 路线:`regbase.md`
- C++ 安全:`cpp-secure.md`
