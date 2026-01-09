# clob/ws/types/response.rs 阅读笔记

## deserialize_any 的详细功能

`deserialize_any` 是 `serde` 提供的一个“动态分派”反序列化入口。它不会预先假定输入是对象/数组/字符串/数字，而是让反序列化器根据**实际 JSON 顶层类型**选择合适的回调。

### 工作机制（直观理解）

- 你实现一个 `Visitor`，包含 `visit_map`、`visit_seq`、`visit_str` 等方法。
- `deserialize_any` 会先看输入的真实类型：
  - 如果是 **对象 `{...}`**，就调用 `visit_map`。
  - 如果是 **数组 `[...]`**，就调用 `visit_seq`。
  - 如果是 **字符串/数字/布尔/null**，就调用对应的 `visit_*`。
- 这样你就能用**同一套入口**处理“多种可能形状”的输入。

### 典型用途

- **需要区分顶层结构**：比如“可能是对象或数组”的接口返回。
- **只想窥视结构而非完整反序列化**：可以只在 `visit_map/visit_seq` 中读取关键字段。
- **提高性能**：避免对不感兴趣的数据做完整解析。

### 在本项目中的用法

在 `peek_message_shape` 中：

- 使用 `deserialize_any(ShapePeeker)` 让 `serde` 自动判定输入是对象还是数组。
- 对象时只提取 `event_type`，数组时只标记为 `Array`。
- 达到“快速分流”的目的：不需要时不进行完整的 `WsMessage` 反序列化。

### 注意点

- `deserialize_any` 更灵活，但也更“低级”；需要你在 `Visitor` 中处理各种可能类型。
- 如果输入类型是固定的（比如只会是对象），优先用 `deserialize_struct` / `deserialize_map` 等更明确的方法。
