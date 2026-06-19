# MC2 / MoE 通信算子检查规则(19 条红线)

> **来源**:CANNBot-Skills `ops/ascendc-code-review/references/mc2-specific.md`
> **适用场景**:MC2(Mass Communication)类算子、MoE 专家路由、量化通信、HCCL API
> **侧别**:Host / Kernel / 两者

---

## 触发判定

满足以下任一即可判定为 MC2 场景:
- 含 `Hccl` / `HCCL` / `mc2` / `MC2` 关键字
- 含 `AlltoAll` / `AllReduce` / `AllGather` / `ReduceScatter`
- 含 `expert` / `MoE` / `EP` / `TP` 关键字
- 含 `SyncAll` / `CrossCoreSync` / `CCU` 关键字

---

## 一、通信同步规则

### [MC2-01] 核间同步必要性 [Kernel] [红线]

**严重度**:高

通信算子必须正确进行核间同步,否则会死锁或读到旧数据。

### [MC2-02] 流同步正确性 [Kernel] [红线]

**严重度**:高

Stream 间同步必须用 `rtStreamSynchronize` / `aclrtSynchronizeStream`,禁止用 busy-wait。

### [MC2-03] SyncAll 同步生效 [Kernel] [红线]

**严重度**:高

`SyncAll` 必须放在所有跨核通信之后,否则部分核未到达。

### [MC2-04] 全局操作一致性 [Host/Kernel] [红线]

**严重度**:高

全局同步操作(如 `HcclBroadcast`)必须所有 rank 调用,否则部分 rank 永久等待。

---

## 二、MoE 专家路由规则

### [MC2-05] 专家索引边界检查 [Kernel] [红线]

**严重度**:高

**问题**:MoE 路由产生的 expert index 必须 `< numExperts`,否则越界访问 expert weights。

**正确做法**:
```cpp
int32_t expertIdx = routingTable[i];
if (expertIdx < 0 || expertIdx >= numExperts) {
    // 报错或 fallback
}
```

### [MC2-06] 专家分发组合一致性 [Host/Kernel] [红线]

**严重度**:高

不同 rank 的专家分发组合必须一致,否则通信后数据错位。

### [MC2-07] 专家参数校验与日志 [Host] [红线]

**严重度**:高

Host 侧对 `numExperts`、`topK`、`capacity` 等参数必须校验并打印。

### [MC2-08] MoE 属性获取规范 [Host] [红线]

**严重度**:高

通过 `GetAttr` 获取 MoE 相关属性时,默认值和兼容性必须处理。

---

## 三、量化精度规则

### [MC2-09] 量化参数类型一致性 [Host/Kernel] [红线]

**严重度**:高

量化 scale / zeroPoint 的 dtype 在 host 和 kernel 必须一致。

### [MC2-10] 量化模式校验完整性 [Host] [红线]

**严重度**:高

量化模式(per-tensor / per-channel / per-group)在 host 必须校验。

### [MC2-11] 量化精度保护 [Kernel] [红线]

**严重度**:高

通信前/后的反量化必须在 fp32 下进行,避免 fp16 累计误差。

### [MC2-12] EP/TP 配置类型匹配 [Host] [红线]

**严重度**:高

Expert Parallel 和 Tensor Parallel 的 degree、rank 配置必须类型匹配。

---

## 四、硬件约束规则

### [MC2-13] CCU 通信数据量限制 [Host] [红线]

**严重度**:高

CCU 单次通信数据量不能超过硬件上限(查 `npu-arch` skill)。

### [MC2-14] Tiling 校验与结构规范 [Host] [设计原则]

**严重度**:中

Tiling 结构体字段顺序必须与 DESIGN.md 严格一致,否则 host/device 解析错位。

---

## 五、HCCL 通信与安全规范

### [MC2-15] PTA 通信域字符串深拷贝 [Host] [红线]

**严重度**:高

**问题**:PTA 通信域字符串必须深拷贝,否则 string 析构后通信域失效。

**正确做法**:
```cpp
// ❌ 错误:浅拷贝
std::string groupName = "group_0";
HcclCommConfig config = {groupName.c_str(), ...};

// ✅ 正确:深拷贝或使用 std::string 字段
HcclCommConfig config;
config.groupName = strdup(groupName.c_str());  // 注意 free
```

### [MC2-16] 禁止直接引用 kernel_operator.h [Kernel] [红线]

**严重度**:高

**问题**:在 MC2 kernel 中直接 `#include "kernel_operator.h"` 会引入冲突。

**正确做法**:只引用必要的 Ascend C 头文件。

### [MC2-17] HCCL API 参数动态查阅验证 [Host/Kernel] [红线]

**严重度**:高

HCCL API 参数(如 `HcclDataType`、`HcclReduceOp`)必须用最新文档验证,不凭记忆。

### [MC2-18] HCCL 通信生命周期与参数 [Host/Kernel] [红线]

**严重度**:高

HcclComm 的创建、使用、销毁必须按生命周期管理,避免资源泄漏。

### [MC2-19] AlltoAllV 跨 rank 参数一致性 [Host/Kernel] [红线]

**严重度**:高

**问题**:AlltoAllV 的 sendbuf/displ 数组在不同 rank 必须有相同的总长度。

**正确做法**:所有 rank 用同一套 metadata,或通过 broadcast 同步。

---

## 相关文档

- 原始来源:`ops/ascendc-code-review/references/mc2-specific.md`
- 同步规则:`simt.md`、`ascendc-api.md`(API-12)
- 硬件参数:`hardware-params.md`
