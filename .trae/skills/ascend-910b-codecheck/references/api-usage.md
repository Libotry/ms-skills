# API 使用 CodeCheck 规则 (Ascend C API)

---

## [API-001] TPipe / TQue 初始化顺序

**严重度**: high

`TPipe` 必须先初始化,再初始化 `TQue`,且 `TQue` 必须绑定到 `TPipe`。

```cpp
// BAD
AscendC::TQue<...> que;
AscendC::TPipe pipe;  // 顺序错

// GOOD
AscendC::TPipe pipe;
pipe.InitBuffer(que, depth, size);
```

---

## [API-002] LocalTensor 生命周期

**严重度**: high

`LocalTensor` 在 `TQue` 之外创建是错误的;必须 `AllocTensor` / `FreeTensor` 配对。

```cpp
// BAD
auto tensor = que.AllocTensor<half>();
// 缺少 FreeTensor

// GOOD
auto tensor = que.AllocTensor<half>();
// ... use tensor
que.FreeTensor(tensor);
```

---

## [API-003] 全局/局部 API 混用

**严重度**: high

`AscendC::GlobalTensor` 与 `AscendC::LocalTensor` 不要直接互相赋值,必须通过 `DataCopy`。

---

## [API-004] dtype 不匹配的 API 调用

**严重度**: high

`DataCopyExtParams` 等结构体的 dtype 字段必须与实际 tensor 一致,否则硬件报错。

---

## [API-005] TilingData 字段访问越界

**严重度**: high

通过 `tiling.Get<TilingData>()->field` 访问字段,字段名错或偏移错都会读到错数据。

**修复**:
- 用强类型 `Get<MyTiling>()->myField`
- 加 `static_assert` 校验 layout

---

## [API-006] AscendC 头文件顺序

**严重度**: low

`kernel_operator.h` 必须在所有 AscendC 头之前,否则部分宏定义会冲突。
