# AscendC API 最佳实践(950PR)

> **来源**:CANNBot-Skills `ops/ascendc-code-review/references/ascendc-api.md`(10 条官方 API 规则)
> **适用架构**:arch35 及以上(950PR)
> **侧别**:Kernel
> **默认启用**:是

> ⚠️ **检视前置要求**:审查本条例时,必须先使用 `ascendc-docs-search` skill 获取相关 API 的最新文档,确认 API 参数、限制、对齐要求等信息是否与条例描述一致。若文档与条例有差异,以最新 API 文档为准。

---

## [API-1] 禁止使用 GlobalTensor::SetValue/GetValue

**严重度**:high

**问题描述**:`GlobalTensor::SetValue()` 和 `GlobalTensor::GetValue()` 是**逐元素操作,性能极差**。生产代码中禁止使用。

**错误示例**:
```cpp
// ❌ 禁止:逐元素访问 GM
for (uint32_t i = 0; i < size; i++) {
    output.SetValue(i, input.GetValue(i));
}
```

**正确示例**:
```cpp
// ✅ 正确:使用 DataCopyPad 批量搬运
AscendC::DataCopyPad(output, input, copyParams, padParams);
```

**检视方法**:`grep -n "SetValue\|GetValue" <file>`

> **注意**:调试时代码中出现的 `GetValue` 用于打印调试信息是允许的。

---

## [API-2] 禁止使用 std:: 计算函数

**严重度**:high

**问题描述**:Kernel 侧**不支持 C++ 标准库**,必须使用 Ascend C 提供的专用 API。

**禁止列表**:

| std:: 函数 | Ascend C 替代 |
|-----------|--------------|
| `std::abs` | `AscendC::Abs(dst, src, count)` |
| `std::min/max` | `(a < b) ? a : b` 或 `AscendC::Min/Max` |
| `std::sqrt` | `AscendC::Sqrt(dst, src, count)` |
| `std::pow` | `AscendC::Power(dst, src, count)` |
| `std::exp` | `AscendC::Exp(dst, src, count)` |
| `std::log` | `AscendC::Log(dst, src, count)` |
| `std::sin/cos` | `AscendC::Sin/Cos(dst, src, count)` |
| `std::floor/ceil` | `AscendC::Floor/Ceil(dst, src, count)` |

**检视方法**:`grep -n "std::\(abs\|min\|max\|sqrt\|exp\|log\|sin\|cos\|floor\|ceil\)" <file>`

---

## [API-3] DataCopy / DataCopyPad 32 字节对齐

**严重度**:high

**问题描述**:DataCopy 要求数据量必须 **32 字节对齐**,非对齐会导致数据错误。**推荐统一使用 DataCopyPad 自动处理对齐**。

| 数据类型 | 对齐元素数 | 最小对齐字节数 |
|---------|-----------|--------------|
| half (2 bytes) | 16 | 32 |
| float (4 bytes) | 8 | 32 |
| int32_t (4 bytes) | 8 | 32 |

**错误示例**:`AscendC::DataCopy(xLocal, xGm, 4);` // cols=4 (16 bytes),数据错误

**正确示例**:
```cpp
AscendC::DataCopyExtParams copyParams;
copyParams.blockLen = cols * sizeof(float);  // 单位:字节
AscendC::DataCopyPad(xLocal, xGm, copyParams);
```

---

## [API-4] Compare API 256 字节对齐

**严重度**:high

**问题描述**:Compare API 要求 `count` 个元素所占空间必须 **256 字节对齐**。

**处理方案**(使用 Padding):
```cpp
// 1. 计算对齐大小(float 类型:64 的倍数)
constexpr uint32_t A0 = 32;
constexpr uint32_t A0_ALIGN = (A0 + 63) / 64 * 64;  // = 64

// 2. UB Buffer 使用对齐大小
pipe.InitBuffer(inQueue, 1, R * A0_ALIGN * sizeof(float));

// 3. CopyIn 时填充极值
Duplicate(xLocal, -FLT_MAX, R * A0_ALIGN);  // ArgMax 用极小值
// 再拷贝实际数据到前 A0 个位置

// 4. API 调用使用对齐大小
Compare(cmpLocal, srcLocal, maxLocal, CMPMODE::GT, A0_ALIGN);

// 5. CopyOut 只输出有效数据
DataCopy(dstGm, yLocal, A0);  // 只输出 A0 个
```

**极值选择**:
- ArgMax / 找最大值:`-FLT_MAX` 或 `-INFINITY`
- ArgMin / 找最小值:`FLT_MAX` 或 `INFINITY`

---

## [API-6] AllocTensor / FreeTensor 配对

**严重度**:high

**问题描述**:使用队列管理内存时,AllocTensor 和 FreeTensor **必须配对调用**,否则会导致内存泄漏。

**检视方法**:
```bash
grep -c "AllocTensor" <file>
grep -c "FreeTensor" <file>
# 两者数量应该相等
```

---

## [API-7] 禁止动态内存分配

**严重度**:high

**问题描述**:AI Core **无动态内存管理能力**,禁止使用动态内存分配。

**禁止**:`std::vector`、`new int[10]`、`malloc(100)`

**正确**:静态分配 `int arr[10]`、`constexpr uint32_t SIZE = 1024`、`pipe.InitBuffer(inQueue, 2, SIZE)`

**检视方法**:`grep -n "new \|malloc\|std::vector" <file>`

---

## [API-8] repeatTimes 限制(≤255)

**严重度**:medium

**问题描述**:部分 Ascend C API 的 `repeatTimes` 参数类型是 `uint8_t`,最大值 255。超过需要分批处理。

**处理**:当 repeatTimes > 255 时,按 255 分批:
```cpp
constexpr uint32_t MAX_REPEAT = 255;
uint32_t fullBatches = repeatTimes / MAX_REPEAT;
uint32_t remainder = repeatTimes % MAX_REPEAT;
```

---

## [API-9] Cast RoundMode 正确性

**严重度**:medium

**问题描述**:Cast API 的 RoundMode 参数必须正确选择。

| 转换方向 | 推荐 RoundMode |
|---------|---------------|
| float → half | `CAST_ROUND`(四舍五入) |
| half → float | `CAST_NONE` |
| float → int | `CAST_FLOOR` 或 `CAST_ROUND` |

---

## [API-10] DataCopy vs DataCopyPad 的 blockLen 单位差异

**严重度**:high

**问题描述**:`blockLen` 的单位取决于**哪个 API** 使用它,而非结构体本身:

| API | blockLen 单位 |
|-----|-------------|
| `DataCopy` (老接口) | **32 字节块数** |
| `DataCopyExtParams` (新接口) | **字节数** |

混用会导致数据搬移量错误(常 ×16 或 ÷16)。

---

## [API-12] CrossCoreSetFlag / WaitFlag 必须对称

**严重度**:high

**问题描述**:跨核同步 flag 必须**一一配对**:
- 所有代码路径(含提前 return)是否都有 SetFlag?
- 是否与 Matmul 高阶 API 混用?
- 同一 flagId 是否超过 15 次?(硬件限制)

---

## 相关文档

- 原始来源:`ops/ascendc-code-review/references/ascendc-api.md`
- 性能规范:`ascendc-perf.md`
- 精度规范:`ascendc-perf.md`(PREC-* 系列)
- Tiling 设计:`tiling.md`
