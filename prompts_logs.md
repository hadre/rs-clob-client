# [2026-01-11 21:58:57]检查秘钥泄露风险
## Prompt
这是一个polymarket 官方的rust sdk实现，你先检查先所有的代码，看看代码中是否存在可能的秘钥泄露到第三方的风险
## 操作记录
1. 使用 rg 扫描仓库内与 secret/token/key/auth 相关的代码位置并定位潜在敏感点。
2. 重点查看 `src/auth.rs`、`src/ws/traits.rs`、`src/rtds/types/request.rs`、`src/lib.rs`、`src/serde_helpers.rs` 的实现与日志行为。
3. 整理并输出潜在泄露风险与对应文件位置、影响说明及建议。
4. AI 回复:
Using ai-log because logging is required for this request.

**Findings**
- High: `serde_helpers::deserialize_with_warnings` 在开启 tracing 时会记录完整 JSON 响应；`create_api_key/derive_api_key` 返回的 `Credentials` 含 `secret`/`passphrase`，可能被日志系统采集并泄露。参考 `src/serde_helpers.rs:116`、`src/lib.rs:255`。
- Medium: WebSocket 认证载荷会把 `secret`/`passphrase` 明文序列化进消息体，且代码未强制 `wss://` 或默认主机校验，若用户配置到非官方或非加密端点，可能泄露给第三方。参考 `src/ws/traits.rs:36`、`src/rtds/types/request.rs:157`。
- Medium: Builder 远程签名模式会向远端发送请求路径/方法/体及可选 Bearer token，并从远端返回 builder api key/passphrase；若远端为第三方主机即存在泄露风险。参考 `src/auth.rs:349`。

**Notes**
- 未发现硬编码的生产秘钥；测试里出现的私钥注明为公开测试密钥。

**Next steps**
1. 为 tracing 日志做敏感字段脱敏或对返回凭证的 endpoint 关闭完整 JSON 记录。
2. 对 WebSocket 认证强制/校验 `wss://` 或要求显式确认发送到非默认主机。
3. 在 `Config::Remote` 增加文档警示并建议仅使用受信任的 HTTPS 端点。
# [2026-01-11 22:03:22]tracing日志脱敏处理
## Prompt
你先把tracing的日志做敏感字段脱敏的处理
## 操作记录
1. 检查 `src/serde_helpers.rs` 的 tracing 日志输出与 `format_value` 行为，确定需要统一脱敏入口。
2. 新增敏感字段识别与递归脱敏函数，并在 tracing 日志与 `format_value` 中使用脱敏后的 JSON。
3. 增加单元测试覆盖敏感字段脱敏的输出行为。
