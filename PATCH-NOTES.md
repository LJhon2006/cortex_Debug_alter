# PATCH-NOTES — SSE 500 修复细节

本补丁只改 `extension/dist/extension.js` 中的 **SSE 处理段**，其余字节与上游 1.4.0 完全一致。

## 根因

后端整个进程只有 **一个共享 `McpServer`**（`@modelcontextprotocol/sdk`）。`GET /sse` 分支大致为：

```js
const transport = new SSEServerTransport("/message", res);
const sessionId = transport.sessionId;
this.transports.set(sessionId, transport);
res.on("close", () => { this.transports.delete(sessionId); }); // 只 delete，没有 close 协议
await this.mcpServer.connect(transport);
```

问题：客户端断开时只 `this.transports.delete(...)`，**没有 `await transport.close()` / `await this.mcpServer.close()`**。SDK 的 `Protocol.connect()` 有守卫 `if (this._transport) throw new Error("Already connected to a transport ...")`，于是 `_transport` 永久占住 → 之后任何 `GET /sse` 都抛 “Already connected”，返回 **HTTP 500**。（`/health` 仍 200，服务未死。）

## 补丁位置（`dist/extension.js` 的 `/sse` 分支）

1. **断开即释放**：`res.on("close", ...)` 中在 `transports.delete(sessionId)` 之后补 `await transport.close()` + `await this.mcpServer.close()`，清掉 `Protocol._transport`。
2. **连接前预清理**：新连接进来前，若 `this.mcpServer._transport` 仍存在，先 `close()` 清残留，避免复用旧 transport。
3. **connect 容错**：把 `await this.mcpServer.connect(transport)` 包 try/catch，失败返回 500 JSON 而不是未捕获异常。

补丁特征（便于核对已装/重打是否正确）：

```
// PATCH: shared McpServer kept its transport ref forever
if (this.mcpServer._transport)
```

（并含在 `res.on("close")` 中 `await transport.close()` 与 `await this.mcpServer.close()` 的清理逻辑。）

## 纯净 vs 已装：精确 diff（外科式区间）

用逐字节比较得出，两个 `extension.js` **仅差一个有界的 SSE 中段**：

| 项 | 值 |
|---|---|
| pristine `dist/extension.js` | 1,528,482 B，sha256 `86a0b626…e3e21b1` |
| installed（patched）`dist/extension.js` | 1,529,487 B，sha256 `0aa0be9720a7556ddcd10f01d15891361e9a8c1440c4a543e9bac2dd25360c49` |
| 公共前缀 `prefix` | 1,432,945 |
| 公共后缀 `suffix` | 95,365 |
| pristine 中段 | 172 B |
| installed 中段 | 1,177 B |
| Δ（尺寸差） | 1,005 B |

**拼接方式**（以纯净为底，仅注入 SSE 中段）：

```
patched = pristine[0 .. 1432945)
        ++ installedMid(1177B)                       // 即 installed[1432945 .. 1434122)
        ++ pristine[1433117 .. 1528482)              // 1433117 = 1432945 + 172
```

结果与已装版本 **字节相等**（`Buffer.equals` = true），sha256 = `0aa0be97…5360c49`，长度 1,529,487 B。

## 升级后重打

扩展升级会覆盖 `dist/extension.js`。重打时不要整份复制旧文件，只把上述 installed 中段（1177B，sha `9a36cddc…526ec`）搬进纯净版 `[1432945, 1433117)` 位置；`extension.vsixmanifest`、`[Content_Types].xml` 等其它条目原样保留。

## 校验

- `extension/dist/extension.js`：len=1,529,487，sha256=`0aa0be9720a7556ddcd10f01d15891361e9a8c1440c4a543e9bac2dd25360c49`，含两条 PATCH 特征串。
- patched vsix sha256=`9d74ef566fb01ea9cb150a6ef8c06f67bb2c8887f6d3e4cfb70491c762fe1a13`。
