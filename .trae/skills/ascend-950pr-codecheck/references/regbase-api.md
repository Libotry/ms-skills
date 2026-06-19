# RegBase API 白名单与签名参考

> **来源**:CANNBot-Skills `ops/ascendc-regbase-best-practice/references/api/`
> - `regbase_api_whitelist.md`
> - `regbase_api_reference.md`
> - `regbase_api_sync.md`

> ⚠️ **重要声明**:**此文档不能替代 CANN SDK header 和官方文档**。审查 RegBase 代码时,必须先查阅 SDK 最新文档和参考实现。

---

## RegBase API 家族概览

### 1. RegTensor 核心

```cpp
template <typename T>
class RegTensor {
public:
    static RegTensor Load(__ubuf__ T* addr, uint32_t count, MaskReg& mask);
    void Compute(MaskReg& mask);
    void Store(__ubuf__ T* addr, uint32_t validCount);
    
    // dtype 转换
    template <typename U>
    RegTensor<U> Cast(CastTrait trait);
    
    // 维度操作
    RegTensor<T> LoadAlign(...);
    RegTensor<T> LoadDist(...);
    void StoreDist(...);
};
```

### 2. MaskReg

```cpp
class MaskReg {
public:
    static MaskReg UpdateMask(uint32_t validCount);  // 最常用
    bool IsAllTrue();
    uint32_t GetValidCount();
};
```

### 3. 计算 API

- 算术:`Add`、`Sub`、`Mul`、`Div`、`Muls`、`Adds`
- 规约:`ReduceSum`、`ReduceMax`、`ReduceMin`
- 比较:`Compare`、`Max`、`Min`
- 数学:`Exp`、`Log`、`Sqrt`、`Rsqrt`

### 4. 同步 API(950PR 替代 set/wait)

- `BufferID`(950PR 新机制)
- `CrossCoreBufferID`

---

## [RB-API-1] 使用未验证的 RegTensor 签名

**严重度**:high

**问题**:凭函数名猜测 `AscendC::Reg::*` 签名。

**修复**:
1. 查 `regbase_api_whitelist.md`
2. 查 SDK header
3. 查真实参考实现

---

## [RB-API-2] 缺少 MaskReg 参数

**严重度**:high

**问题**:每个计算步骤未传递 MaskReg。

**正确做法**:
```cpp
auto reg_a = RegTensor::Load(addr, count, mask);  // Load 传 mask
reg_a.Compute(mask);                              // Compute 传 mask
RegTensor::Store(reg_a, dst, mask.GetValidCount()); // Store 用 validCount
```

---

## [RB-API-3] Load/Store Dist 错

**严重度**:high

**问题**:`LoadDist` / `StoreDist` 用错,导致 UB/寄存器数据布局错位。

**修复**:查 API reference + 真实参考实现。

---

## [RB-API-4] dtype 模板参数缺失

**严重度**:high

**问题**:`RegTensor<T>` 的 `T` 不匹配实际数据类型。

**修复**:明确 `RegTensor<half>`、`RegTensor<float>` 等。

---

## [RB-API-5] Cast Trait 不正确

**严重度**:high

**问题**:`Cast(trait)` 中 trait 选择错误,导致数据截断或溢出。

**正确做法**:查 SDK 中 `CastTrait` 的可选值。

---

## RegBase 与 MemBase API 严格区分

| API | 适用路径 |
|-----|---------|
| `AscendC::Reg::*`、`RegTensor`、`MaskReg`、`__simd_vf__` | **RegBase only** |
| `AscendC::Add`、`AscendC::Mul`、`LocalTensor` | **MemBase only**(但 RegBase outer shell 可用) |
| `AscendC::DataCopy`、`AscendC::DataCopyPad` | MemBase;RegBase 用对应封装 |
| `AscendC::pipe.InitBuffer`、`AscendC::TQue` | RegBase outer shell 可用 |

**核心原则**:VF body(在 `__VEC_SCOPE__` 或 `__simd_vf__` 内)**只能调用 RegBase API**,不能调用 MemBase vector API。

---

## 同步 API(950PR 重要变化)

950PR 新引入 **BufferID** 同步机制,替代传统 set/wait 强配对:

```cpp
// ✅ 950PR 推荐:BufferID 同步
BufferID bufId;
bufId.Set(...);
// ... 数据写入 ...
bufId.Wait();

// ❌ 910B 时代:set/wait
SetFlag<...>(eventId);
WaitFlag<...>(eventId);
```

**优势**:BufferID 不需要 SetFlag/WaitFlag 强配对,语义更清晰。

---

## 相关文档

- 原始来源:
  - `ops/ascendc-regbase-best-practice/references/api/regbase_api_whitelist.md`
  - `ops/ascendc-regbase-best-practice/references/api/regbase_api_reference.md`
- RegBase 路线:`regbase.md`
- RegBase 陷阱:`regbase-traps.md`
