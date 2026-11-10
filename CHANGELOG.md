# Changelog

## 1.4.0-patched（生成日期 2026-11-10）

基于上游 `yuelongzhang.cortex-debug-mcp` **1.4.0** 的补丁构建（二次分发）。

### 修复
- **SSE 500（共享 `McpServer` 不释放 transport）**：后端只有一个共享 `McpServer`，`GET /sse` 断开时只删除 map 记录、未 `close()` 协议与 `mcpServer`，导致 `Protocol._transport` 永久占用，之后所有 `GET /sse` 返回 HTTP 500（`Already connected to a transport`）。补丁在 `res.on("close")` 中 `transport.close()` + `mcpServer.close()`，连接前预清理残留 `_transport`，并对 `connect()` 做 try/catch 容错。
- 仅替换 `extension/dist/extension.js` 的 SSE 中段（surgical splice），其余 11 个 vsix 条目原样保留。

### 证据 / 校验
- `extension/dist/extension.js`：len = **1,529,487 B**，sha256 = **0aa0be9720a7556ddcd10f01d15891361e9a8c1440c4a543e9bac2dd25360c49**
- patched vsix：`yuelongzhang.cortex-debug-mcp-1.4.0-patched.vsix`，sha256 = **9d74ef566fb01ea9cb150a6ef8c06f67bb2c8887f6d3e4cfb70491c762fe1a13**
- 纯净 vs 已装精确 diff：prefix=1432945 / suffix=95365 / pristineMid=172B / installedMid=1177B / Δ=1005（详见 `PATCH-NOTES.md`）。
- 补丁特征串：`PATCH: shared McpServer kept its transport ref forever`、`if (this.mcpServer._transport)`。

### 备注
- 版本号保持 `1.4.0`（不 bump，避免与上游版本线冲突）；以 `-patched` 标注本构建。
- 上游：<https://github.com/MrAlexZhang/cortex-debug-MCP-server> ／ <https://github.com/paulopalaoro/cortex-mcp-bridge>
