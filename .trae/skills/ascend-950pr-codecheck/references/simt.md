# SIMT 编程模型检查规则(950PR 新增)

> **来源**:CANNBot-Skills `ops/ascendc-code-review/references/simt-constraints.md`
> **适用架构**:arch35 及以上(950PR / 950DT)
> **侧别**:Kernel
> **新增于**:CANNBot-Skills 2026-05-25

950PR 新增 **SIMT 编程模型**,通过协处理器(VF)实现类似 CUDA SIMT 的并行编程。需要严格遵守以下规则,否则编译失败或运行异常。

---

## 禁止事项(红线)

### [SIMT-1] VF 参数 size 超过 28x32bit

**严重度**:high

**问题描述**:SIMT VF 入口函数的参数 size 控制在 **28x32bit** 以内。超过会导致寄存器分配失败。

**参数类型占用**:

| 类型 | 占用 (32bit 单位) |
|------|-----------------|
| int32_t / uint32_t / float | 1 |
| int64_t / uint64_t / double | 2 |
| `__gm__ T*` / `__ubuf__ T*` | 2 (64bit 地址) |
| half / bfloat16_t | 1 |
| bool | 1 |

**修复原则**:
1. VF 参数能用模板参数传递,**尽可能使用模板参数传递**
2. VF 参数能在 SIMT VF 内部计算得到的,**就不要作为参数传递**
3. 如果参数 size 超过 28x32bit:
   - 将 VF 内部频繁使用的参数作为入参
   - 其余参数存到 UB 的 buffer 中
   - 将 buffer 地址(`__ubuf__` 指针)作为入参

---

### [SIMT-2] 用 `cce_parallel` 启动 SIMT 函数

**严重度**:high

**问题描述**:必须使用 `AscendC::Simt::VF_CALL` 启动 SIMT 函数。`cce_parallel` 是错误的启动方式。

---

### [SIMT-3] 在 SIMT 函数内部直接使用 UB

**严重度**:high

**问题描述**:在 SIMT 函数内部直接使用 UB 是禁止的,需在 tiling 侧设置 `SetLocalMemory`。

---

### [SIMT-4] 在 kernel 代码中对 tilingdata 进行赋值

**严重度**:high

**问题描述**:TilingData 是 host 侧传入的只读配置,**禁止在 kernel 代码中修改**。

---

### [SIMT-5] 将 VF 线程数放入 TilingData

**严重度**:high

**问题描述**:VF 线程数是**编译期常量**,不属于 tiling 参数。禁止将 `threadNum/blockDimX/blockDimY` 等放入 TilingData 结构体。

---

### [SIMT-6] 在 `Simt::Dim3(...)` 中使用 tilingData 读取的变量

**严重度**:high

**问题描述**:Dim3 参数必须是 `constexpr` 变量或数字字面量,禁止用从 tilingData 读取的运行时变量。

---

### [SIMT-7] `__simt_vf__` 内调用 `__simd_callee__` 函数

**严重度**:high

**问题描述**:VF 和 SIMD 函数是两种不同的执行模型,禁止跨模型调用。

**函数调用约束矩阵**:

| 调用方 | 被调用方 | 是否允许 |
|--------|---------|---------|
| `__simt_vf__` | `__simt_callee__` | 允许 |
| `__simt_vf__` | `constexpr` 函数 | 允许 |
| `__simt_vf__` | `__simd_callee__` | **禁止** |
| `__simt_vf__` | 普通 `__aicore__` 函数 | **禁止** |
| `__simd_vf__` | `__simd_callee__` | 允许 |
| `__simd_vf__` | `constexpr` 函数 | 允许 |
| `__simd_vf__` | `__simt_callee__` | **禁止** |

---

### [SIMT-8] kernel 入口参数缺失 `_out` 后缀

**严重度**:medium

**问题描述**:inplace 算子场景下,重复的 Input/Output 未在入口参数中对 Output 添加 `_out` 后缀,会导致符号匹配失败。

---

## 必须遵守规则

### [SIMT-9] VF_CALL 启动规范

**严重度**:high

**规则**:
- 必须使用 `AscendC::Simt::VF_CALL` 启动 SIMT 函数
- 禁止使用 `cce_parallel` 启动 SIMT 函数

---

### [SIMT-10] 编译期线程数三处一致

**严重度**:high

**核心规则**:SIMT VF 的线程数**必须在编译期确定**,禁止从 tiling 数据动态获取。

**三处一致**:以下三处必须使用同一个 `constexpr` 常量:

1. `constexpr uint32_t THREAD_NUM = 512;`
2. `LAUNCH_BOUND(THREAD_NUM)`
3. `Simt::Dim3(THREAD_NUM)`

**正确写法**:
```cpp
constexpr uint32_t THREAD_NUM = 512;

// VF 函数声明
__simt_vf__ __aicore__ LAUNCH_BOUND(THREAD_NUM) inline void OpComputeSimt(...);

// VF 调用
Simt::VF_CALL<OpComputeSimt<T>>(Simt::Dim3(THREAD_NUM), args...);
```

**错误写法(严禁)**:
```cpp
// 线程数从 tiling 数据获取(运行时变量)
int32_t threadNum = static_cast<int32_t>(tilingData_->threadNum);
Simt::VF_CALL<OpComputeSimt<T>>(Simt::Dim3(threadNum), args...);
```

**为什么**:
- 硬件在编译期需要知道线程数以分配寄存器和调度资源
- 运行时变量无法用于寄存器分配决策
- 动态线程数会导致编译错误或运行时异常

**数据量小时的调整**:如果数据量较小导致线程空转过多,应通过调整**核数**(`SetBlockDim`)来适配,而不是减少线程数。

---

### [SIMT-11] 函数修饰符必须正确

**严重度**:high

**规则**:
- `AscendC::Simt::VF_CALL` 所调用的函数声明和定义中必须带有 `__simt_vf__` 修饰符
- `__simt_vf__` 修饰的函数内所调用的自定义子函数必须带有 `__simt_callee__` 修饰

---

### [SIMT-12] SIMT callee 子函数必须正确修饰

**严重度**:high

**正确写法**:
```cpp
// SIMT callee 子函数
__simt_callee__ inline int64_t CalcOffset(int64_t base, int64_t stride, int64_t idx) {
    return base + stride * idx;
}

__simt_vf__ __aicore__ LAUNCH_BOUND(512) inline void OpSimt(...) {
    int64_t offset = CalcOffset(0, 1, i);  // 调用 __simt_callee__
}
```

**错误写法**(缺少 `__simt_callee__` 修饰,编译错误):
```cpp
__aicore__ inline int64_t CalcOffset(int64_t base, int64_t stride, int64_t idx) {
    return base + stride * idx;
}
```

---

### [SIMT-13] SIMT 算子应直接使用 GM_ADDR

**严重度**:medium

**规则**:纯 SIMT 算子应该直接使用传入的 `GM_ADDR` 参数,避免申请 GlobalTensor 后 GetPhyAddr。

---

### [SIMT-14] 头部 License 时间正确

**严重度**:low

**规则**:代码文件头部 License 声明中时间修改为实际年份。

---

## UB 与 DCache 约束(950PR SIMT 特有)

### [SIMT-15] UB 256KB 分区约束

**严重度**:high

**950PR UB 总量 = 256KB**(注意:虽然 DAV_3510 UB 是 248KB,但 SIMT 模式下统计按 256KB 总量)

| 区域 | 大小 | 说明 |
|------|------|------|
| 静态内存 | 编译期确定 | `__ubuf__` 数组声明 |
| 动态内存 | tiling 侧 `SetLocalMemory` 设置 | TBuf/LocalTensor 申请 |
| 预留空间 | 固定 8KB | 编译器预留,不可使用 |
| Data Cache | 256KB - 静态 - 动态 - 8KB | SIMT 专有 DCache |

**DCache 最低要求**:
- **DCache 必须 >= 32KB**
- 若 DCache < 32KB,编译校验报错
- 计算:可用 UB = 256KB - 8KB - 32KB = **216KB**

**Tiling 侧设置**:
```cpp
constexpr uint64_t DCACHE_SIZE = 128 * 1024;  // 定义为 128KB
uint64_t ubsize = 256 * 1024;
context->SetLocalMemorySize(ubsize - DCACHE_SIZE);
```

**内存超用后果**:

| 问题 | 表现 | 原因 |
|------|------|------|
| DCache 不足 | 编译校验报错 | DCache < 32KB |
| 动态内存不足 | 运行时 UB 访问越界 | SetLocalMemorySize 设置过大 |
| 静态 + 动态超限 | 编译或运行时错误 | 静态 + 动态 > 216KB |

---

## 数据格式扩展(950PR 新增)

950PR 新增以下数据类型,代码中如使用应核对 CANN SDK 支持:

| 数据格式 | 说明 | 用途 |
|---|---|---|
| **FP8** | 8 位浮点(E4M3 / E5M2) | Cube MMAD |
| **MXFP8** | Microscaling FP8 | 高精度低 bit |
| **MXFP4** | Microscaling FP4 | 极致压缩 |
| **HiF8** | 华为 FP8 格式 | Cube MMAD |

---

## 相关文档

- 原始来源:`ops/ascendc-code-review/references/simt-constraints.md`
- 950PR SIMT 架构参考:`ops/npu-arch/references/simt-arch-guide.md`
- 950PR 硬件参数:`hardware-params.md`
