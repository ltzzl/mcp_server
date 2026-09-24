# Project Instructions: mcp_server

C11 实现的独立 MCP（Model Context Protocol）后端服务，基于 libuv 事件循环 + jansson JSON。
不被 Codex 直接调用：Codex ─stdio→ `mcp_stdio_proxy_adapter` ─tcp→ `mcp_server`。

## Tech Stack
- 语言：C11（CMake `CMAKE_C_EXTENSIONS OFF`，严格标准；含 `_WIN32` 分支）
- 构建：CMake ≥ 3.20，模块在 `cmake/`，平台配置在 `config/{linux,windows}_defconfig.cmake`
- 依赖：libuv + jansson，bundled 在 `external/`（git submodule，锁定版本）
- 测试：Python smoke tests（`tests/*.py`，经 CTest 调度）

## Architecture
所有传输汇入单一消息队列，由 libuv async 回调 `core_async_cb`（`src/core/server.c`）串行处理：
传输层 → `mcp_jsonrpc_parse_line` → `queue_message`+`uv_async_send` → `handle_request`/`handle_notification`
→ `mcp_gateway_call`（**唯一工具执行路径**）→ registry 查描述符 → `descriptor->handler()`。
- 网关是所有工具的统一入口（含内置工具），为将来 embedded/remote/plugin 路由统一策略与审计。
- 会话状态机：initialize → (strict 时) await initialized notification → initialized；`gate_allows_method` 门控。
- `tools/call` 只允许调用该会话 `tools/list` 快照里可见的工具，否则拒绝。
- per-reply-target 会话：stdio 单会话；tcp/udp/pipe 每 peer 一个 `mcp_client_session`（链表）。
- in-flight 表（`src/core/in_flight.*`）支持 `notifications/cancelled` 取消。

## Project Structure
- `src/core/` 核心、会话状态机、消息队列、in-flight
- `src/gateway/` 工具调度网关（唯一执行路径）
- `src/registry/` 工具注册表 + public/internal 列表
- `src/tools/` 内置工具 + `shell_exec`（沙箱执行，独立模块）
- `src/transport/` stdio、udp；`src/listener/` framed listener（tcp+pipe，4字节大端长度头）
- `src/protocol/` JSON-RPC 解析/构造
- `include/mcp/` 公开头文件（`plugin/plugin_abi.h`、`embedded/mep.h` 为占位，尚未实现）
- `cmake/` `config/` 平台检测与特性开关；`external/` 锁定的第三方源码

## Build & Run
- 初始化：`git submodule update --init --recursive`（或 `./scripts/bootstrap.sh`）
- 构建：`cmake -S . -B build -C config/linux_defconfig.cmake -DCMAKE_BUILD_TYPE=Release && cmake --build build --parallel`
- 测试：`ctest --test-dir build`（需启用 `MCP_BUILD_TESTS`）
- 运行（TCP）：`MCP_ENABLE_STDIO=0 MCP_ENABLE_TCP=1 MCP_TCP_HOST=127.0.0.1 MCP_TCP_PORT=18767 ./build/src/mcp_server`
- 关键环境变量：`MCP_ENABLE_{STDIO,TCP,UDP,PIPE}`、`MCP_TCP_{HOST,PORT}`、`MCP_ENABLE_SHELL_EXEC`（高危，默认关）

## Conventions
- 提交：Conventional Commits（`feat:` / `fix:` / `chore:`，可带 scope，如 `feat(transport):`）
- 命名：函数 `mcp_<module>_<action>`，文件 snake_case，类型 `struct mcp_*`
- 错误处理：返回 int（0 成功 / -1 失败），分层 cleanup；jansson 手动 `json_decref` 管理引用计数
- 特性裁剪：工具用 `MCP_HAS_TOOL_*`、传输用 `MCP_HAS_TRANSPORT_*` 编译期开关
- `system.shell_exec` 是 L4 高危工具，默认禁用，带 CPU/内存/进程/超时限制与审计日志（`config/tools/shell_exec.json`）

## Where to Look
| 我想... | 看 |
|---------|-----|
| 加一个内置工具 | `src/tools/builtin_tools.c`（`register_tool` + handler） |
| 改 JSON-RPC 方法处理 | `src/core/server.c` `handle_request` |
| 改工具调度/鉴权 | `src/gateway/gateway.c` |
| 加一种传输 | `src/transport/` 或 `src/listener/` + `mcp_server_start_*` |
| 改构建/平台开关 | `cmake/MCP*.cmake`、`config/*_defconfig.cmake` |
| 加测试 | `tests/*.py` + `tests/CMakeLists.txt` |
