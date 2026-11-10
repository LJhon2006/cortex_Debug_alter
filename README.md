# Cortex Debug MCP（1.4.0 · patched）

把 **Cortex-Debug / PlatformIO** 的调试能力通过 **stdio MCP** 暴露给 AI 客户端。
本仓库是 **基于 [yuelongzhang/cortex-debug-mcp](https://github.com/MrAlexZhang/cortex-debug-MCP-server) 1.4.0 的补丁构建与二次分发**，修复了上游 SSE 后端的一个会导致连接永久 500 的 bug。**非原作者作品**，版权与署名见“上游与致谢”。

> English abstract: A patched build of the `cortex-debug-mcp` VS Code extension (v1.4.0). It exposes Cortex-Debug / PlatformIO debug capabilities to MCP clients over stdio. This fork fixes a bug where the SSE backend keeps a single shared `McpServer` and never releases its transport on client disconnect, causing all subsequent `GET /sse` to fail with HTTP 500. Built from the upstream 1.4.0 with a surgical SSE section patch; see `PATCH-NOTES.md` and `CHANGELOG.md`.

---

## 这是什么

- **两段式结构**：
  - VS Code 扩展后端 `extension/dist/extension.js`：一个 `McpServer`（`@modelcontextprotocol/sdk`）+ 内置 HTTP 服务，暴露 `GET /sse`、`POST /message`、`GET /health`。
  - stdio↔SSE 桥 `extension/dist/stdioProxy.js`：把 MCP 客户端的 stdio 桥接到扩展后端的 SSE。
- **25 个 MCP 工具**：会话控制/栈、变量/表达式、寄存器、内存、断点、执行控制、符号、原生 `gdb_command`、OpenOCD 实时/外设工具。
- 后端**仅在 VS Code 打开且本扩展激活时**才存在。

## 修复了什么（SSE 500）

上游 1.4.0 的后端整个进程只有**一个共享 `McpServer`**；`GET /sse` 分支里 `res.on("close", ...)` 只把 transport 从 map 删除，**没有 `close()` 协议/`mcpServer.close()`**，于是 `Protocol._transport` 被永久占用：

- 一旦某个 SSE 客户端断开（哪怕异常断开），之后所有 `GET /sse` 都返回 **HTTP 500**；
- 日志报 `Error: Already connected to a transport. Call close() before connecting to a new transport, or use a separate Protocol instance per connection.`；
- `/health` 仍 200（服务活着）。

本补丁在断开时释放 transport、连接前清理残留、并对 connect 做容错，详见 `PATCH-NOTES.md`。

## 安装

```bash
# 直接装本仓库内已构建好的 vsix（同版本 1.4.0 覆盖需 --force）
code --install-extension yuelongzhang.cortex-debug-mcp-1.4.0-patched.vsix --force
```

或先卸载旧版本再安装：

```bash
code --uninstall-extension yuelongzhang.cortex-debug-mcp
code --install-extension yuelongzhang.cortex-debug-mcp-1.4.0-patched.vsix
```

> 若 VS Code 正在运行，安装后 **Reload Window / 重启 VS Code** 让扩展进程加载新的 `dist/extension.js` 后修复才在连接会话中生效。

## MCP 客户端配置（stdio）

方式 A —— 直接用仓库内的 stdio 代理（把路径换成本仓库 `extension/dist/stdioProxy.js` 的绝对路径）：

```json
{
  "servers": {
    "cortex-debug-mcp": {
      "type": "stdio",
      "command": "C:/Program Files/nodejs/node.exe",
      "args": ["C:/path/to/this/repo/extension/dist/stdioProxy.js"]
    }
  }
}
```

方式 B —— DSH（cordis.patch.yml）里用“启动器”方式：用一个小的 launcher 自动探测后端端口并以 `CORTEX_DEBUG_MCP_PORT` 环境变量启动 `dist/stdioProxy.js`（因为扩展的端口会在 7580 起漂移、且状态文件可能缺失）：

```yaml
- insert:
    - id: mcp-cortex-debug
      name: "@deepseek-ai/dsh-mcp-client"
      config:
        serverName: cortex-debug-mcp
        transport: stdio
        command: "C:/Program Files/nodejs/node.exe"
        args: ["C:/path/to/cortex-debug-mcp-launch.js"]   # 该 launcher 设置 CORTEX_DEBUG_MCP_PORT 后启动 dist/stdioProxy.js
        toolCallTimeoutMs: 120000
        reconnect: { enabled: true }
```

stdio 代理支持的端口/状态来源：环境变量 `CORTEX_DEBUG_MCP_BACKEND_URL` / `CORTEX_DEBUG_MCP_PORT`，或 `%TEMP%\cortex-debug-mcp-backend.json`。

## 上游与致谢

- 本扩展 fork 自：<https://github.com/MrAlexZhang/cortex-debug-MCP-server>（作者 yuelongzhang / yuelong.zhang）
- 其上游再追溯：<https://github.com/paulopalaoro/cortex-mcp-bridge>
- 原扩展标识：publisher `yuelongzhang`，name `cortex-debug-mcp`，version `1.4.0`，license MIT。

本仓库仅为上述作品的**补丁构建/二次分发**，作者署名与版权归原作者。

## 限制

- 多数 DAP 读操作要求目标已 **paused**；`get_registers` / `get_memory` 依赖 Cortex-Debug 自定义 DAP 请求。
- OpenOCD 系工具（`get_chip_info`、`read_live_memory`、外设类）在 **J-Link-only** 会话可能失败；可用 `gdb_command` 逃生。
- 后端只在 **VS Code 打开且扩展激活**时存在；关闭 VS Code 后 MCP 不可用（设计如此）。
- 扩展升级会覆盖 `dist/extension.js`，SSE 补丁会丢失，需按 `PATCH-NOTES.md` 重打。

## 免责声明

- 本仓库按“现状”提供，不提供任何明示或暗示担保；使用风险自负。
- 这是一个社区补丁分发，**不代表上游作者立场**，也未获上游背书。
- 请遵守上游项目许可证（MIT）。

## 目录结构

```
README.md / CHANGELOG.md / PATCH-NOTES.md / LICENSE / SHA256SUMS.txt / .gitignore
yuelongzhang.cortex-debug-mcp-1.4.0-patched.vsix   # 已构建好的补丁 vsix
extension.vsixmanifest / [Content_Types].xml       # vsix 打包元数据
extension/
  dist/extension.js      # 已含 SSE 修复（sha256 见 SHA256SUMS.txt）
  dist/stdioProxy.js
  package.json / README.md / LICENSE.txt / images/ / scripts/
```

## 校验

见 `SHA256SUMS.txt`。关键值：`extension/dist/extension.js` = 1,529,487 B，sha256 `0aa0be97…5360c49`。
