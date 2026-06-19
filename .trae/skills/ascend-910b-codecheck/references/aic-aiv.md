# AIC/AIV 协作 CodeCheck 规则 (AIC & AIV Cooperation)

---

## [AIC-001] AIC 直接搬出到 GM

**严重度**: high

AIC(Cube)算子计算完成后,结果应该走 AIV 搬出,不应直接 `DataCopy` 到 GM(部分硬件不支持)。

**修复**: AIC 写到 L0/L1 → AIV 搬到 GM。

---

## [AIC-002] AIV 直接访问 GM(无 DataCopy)

**严重度**: high

AIV 不能像 CPU 一样 `*ptr` 访问 GM,必须先 `DataCopy` 到 UB。

---

## [AIC-003] scalar 参数未通过 GM 传递

**严重度**: medium

bias、scale 等标量参数应通过 GM 传入,而不是硬编码到 kernel。

**修复**: host 端写入 GM,device 端 `DataCopy` 到 scalar 寄存器。

---

## [AIC-004] 跨核 ID 不一致

**严重度**: high

`CrossCoreSetFlag(0x0, ...)` 的 `0x0` 是目的核 ID,host 和 device 双方必须一致。

---

## [AIC-005] AIC/AIV 启动顺序

**严重度**: high

通常 AIC 先启动,计算完成后通知 AIV。
如果 AIV 先启动等数据,可能错过 sync;如果 AIC 启动晚,AIV 会 stall。

**修复**: 按 `kernel.h` 中的 `aic/aiv` 入口分别配置启动顺序。
