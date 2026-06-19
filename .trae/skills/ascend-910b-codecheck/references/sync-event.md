# 同步与 Event CodeCheck 规则 (Sync & Event)

---

## [SYNC-001] SetFlag / WaitFlag 配对缺失

**严重度**: high

`SetFlag` 与 `WaitFlag` 必须成对出现,缺失会导致死锁或数据竞争。

```cpp
// BAD - 配对缺失
AscendC::SetFlag<AscendC::HardEvent::MTE2_V>(eventId);
// 缺少 WaitFlag

// GOOD
AscendC::SetFlag<AscendC::HardEvent::MTE2_V>(eventId);
AscendC::WaitFlag<AscendC::HardEvent::MTE2_V>(eventId);
```

---

## [SYNC-002] 跨核同步缺失

**严重度**: high

AIC(A1)计算完成后,AIV(A2)开始搬出结果,中间必须 sync。

**修复**:
```cpp
// AIC 端
AscendC::CrossCoreSetFlag<0x0, PIPE_MTE3>(flagId);
// AIV 端
AscendC::CrossCoreWaitFlag<0x0, PIPE_MTE3>(flagId);
```

---

## [SYNC-003] pipe barrier 缺失

**严重度**: high

pipeline 切换时,如果没有 barrier,前一个 pipe 的指令可能未完成。

**修复**:
```cpp
AscendC::PipeBarrier<PIPE_ALL>();
```

---

## [SYNC-004] event ID 复用冲突

**严重度**: medium

多个循环复用同一个 event ID 但没 wait,会导致乱序。

**修复**: 每个 event 在 SetFlag 之前必须有对应的 WaitFlag。

---

## [SYNC-005] scalar 同步缺失

**严重度**: medium

scalar 参数从 GM 搬到 scalar 寄存器后,后续计算前需要等 sync。

**修复**:
```cpp
AscendC::DataCopy(scalarReg, scalarGm);
AscendC::PipeBarrier<PIPE_S>();
// 现在 scalarReg 可用
```
