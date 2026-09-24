# 外部 MCP Gateway 增强 MCP Server 方案

## 0. 推荐结论与实施边界

推荐采用“**稳定内核接口 + 先 stdio 外部 upstream + 后续多 transport adapter**”方案。

第一版不同时实现 HTTP、动态发现、负载均衡和复杂熔断，而是完成一个真正可用、可验证的外部 MCP 闭环：

```text
静态配置的外部 stdio MCP Server
  → Upstream Manager 托管子进程
  → 完成兼容协议握手
  → 拉取 tools/list
  → 以 <upstream>.<tool> 注册 REMOTE_SERVER descriptor
  → 本地客户端直接 tools/call 聚合后的名字
  → Gateway 普通 route 分支
  → 外部 tools/call
  → 结果按原客户端 reply target 返回
```

选择 stdio 作为首个标准外部 adapter 的原因：

- 它是官方 MCP 的本地 transport，采用换行分隔 JSON-RPC，与本项目现有 stdio buffering 模式相近；
- 不需要在第一阶段同时引入 HTTP、TLS、重定向、代理、DNS rebinding 和 SSE 状态机；
- 子进程、stdin/stdout 和退出状态容易用 fake upstream 做确定性集成测试；
- adapter 接口稳定后，Streamable HTTP 可以作为第二种 transport 加入，而不改 gateway/core。

当前项目固定使用 `2024-11-05` 初始化流程。官方 MCP 在 2026-07-28 revision 已引入无 initialize 的新生命周期，并继续把 stdio 与 Streamable HTTP 作为标准 transport。因此 MVP 必须在配置中显式限定支持的协议 revision：第一版只接受项目已经实现并经过测试的 legacy initialize lifecycle；收到不兼容 revision 时明确拒绝，不能假装兼容。多 revision negotiation 是后续独立阶段。

当前 discovery/peer TCP framing 是本项目节点互联协议，应包装成可选 `peer_adapter` 复用，但不能把它标成通用外部 MCP transport。远程标准 MCP 的推荐扩展是 Streamable HTTP adapter。

方案状态：**设计建议，尚未实现**。本文中的“现有”均指源码已存在；“新增/建议”均是后续开发内容。

## 1. 目标

在现有 MCP Server 的 registry、gateway、discovery 和 peer transport 基础上，增加面向外部 MCP Server 的统一接入层，使本服务能够：

- 配置或动态发现多个外部 MCP Server；
- 聚合外部工具并安全地公布给本地客户端；
- 将 `tools/call` 路由到正确的外部节点；
- 对鉴权、授权、超时、取消、重试、熔断、限流和审计实施统一策略；
- 隔离单个外部节点故障，避免拖垮本地服务；
- 保持现有本地工具和插件调用路径兼容。

本文是架构增强方案，不代表这些能力已经全部实现。第 15 节定义第一版的精确完成标准，避免把远期能力混入 MVP。

## 2. 现有基础与缺口

### 2.1 已有基础

现有源码已经具备以下关键积木：

- `mcp_gateway_call()` 是所有工具调用的统一入口。
- registry 保存工具 descriptor 和 route。
- `MCP_TOOL_ROUTE_REMOTE_SERVER` 已出现在工具 route 枚举中。
- `gateway.proxy_tool` 可以显式指定 `server_id` 和远端工具名进行代理。
- discovery 能跟踪 peer、缓存远端 `tools/list` 并发起远端调用。
- peer transport 支持 JSON 帧、自定义二进制帧和 capability。
- in-flight map 保存请求 ID、回复目标、取消、timer 和异步上下文。
- network access policy 提供网络开关、IP allowlist 和预配置 discovery peers。
- session tool snapshot 防止客户端调用未曾看到的工具。

### 2.2 主要缺口

要成为可靠的外部 Gateway，还需要补齐或强化：

- 外部 upstream 的持久配置模型和生命周期管理；
- 标准化 MCP transport adapter，而不是只依赖当前 peer 协议；
- 自动工具聚合和稳定命名；
- 每个 upstream 的认证信息安全加载与轮换；
- 基于客户端、工具、upstream 和风险等级的授权策略；
- 端到端 deadline、取消传播和受控重试；
- 熔断、并发舱壁、背压和速率限制；
- 外部工具列表变化与本地 session snapshot 的一致性策略；
- 结构化指标、审计事件和敏感数据脱敏；
- upstream 失效、重连和优雅下线机制。

## 3. 建议架构

```text
MCP Client
    │ initialize / tools/list / tools/call
    ▼
现有 transport + protocol + core
    │
    ▼
Gateway Policy Pipeline
    ├── session snapshot 校验
    ├── identity / authorization
    ├── schema / size / risk 校验
    ├── rate limit / concurrency limit
    └── route resolution
             │
      ┌──────┼───────────┐
      ▼      ▼           ▼
 local    plugin    external upstream
 handler  manager        │
                         ▼
                Upstream Manager
                ├── connection pool
                ├── capability negotiation
                ├── tools cache
                ├── deadline/cancel/retry
                ├── circuit breaker
                └── health state
                         │
                         ▼
                Transport Adapters
                ├── stdio child process（MVP）
                ├── Streamable HTTP（第二阶段远程标准 transport）
                ├── existing peer protocol（内部节点兼容）
                └── legacy HTTP/SSE（仅有明确兼容需求时）
```

核心原则：保持 core 负责会话和响应归属，gateway 负责策略与路由，upstream manager 负责外部连接状态，transport adapter 负责具体协议字节。

## 4. 建议新增的数据模型

### 4.1 Upstream 配置

建议新增 `mcp_upstream_config`，至少包含：

- 稳定的 upstream ID 和显示名称；
- transport 类型和 endpoint；
- 启用状态；
- 认证引用，而不是明文 secret；
- 连接、请求和空闲超时；
- 最大响应大小；
- 最大并发数和排队长度；
- 重试策略；
- 工具 allowlist/denylist；
- 名称前缀或 namespace；
- TLS 和证书校验策略。

配置必须在系统边界做严格 schema 校验。未知字段、非法地址、越界端口、不安全 TLS 选项和明文 secret 应当被拒绝。

### 4.2 Upstream 运行状态

建议新增 `mcp_upstream`：

- immutable config snapshot；
- 当前连接和 MCP session 状态；
- capability；
- 工具目录缓存及版本/generation；
- pending requests；
- 健康状态和最近错误；
- 熔断器状态；
- 并发、失败和延迟指标。

配置对象与运行状态分离，避免运行过程中直接修改安全策略。配置更新采用“创建新快照并原子替换”的方式。

### 4.3 聚合工具描述

外部工具进入本地 registry 时，需要保留：

- 对客户端公开的唯一名称；
- upstream ID；
- 远端原始工具名；
- input schema；
- upstream tool generation；
- 风险等级和权限标签；
- timeout、retryable 和 cancelable 等策略元数据；
- route=`REMOTE_SERVER`。

### 4.4 第一版配置形态

第一版只接受启动时静态 JSON，不做热更新。建议形态：

```json
{
  "upstreams": [
    {
      "id": "files",
      "enabled": true,
      "transport": "stdio",
      "command": "/opt/mcp/bin/files-server",
      "args": ["--root", "/srv/shared"],
      "cwd": "/srv/shared",
      "env_allowlist": ["LANG"],
      "protocol_revision": "2024-11-05",
      "namespace": "files",
      "connect_timeout_ms": 3000,
      "request_timeout_ms": 10000,
      "max_message_bytes": 1048576,
      "max_concurrency": 8,
      "tool_allowlist": ["read_file", "list_directory"]
    }
  ]
}
```

安全约束：

- `command` 必须是绝对路径并匹配管理员 allowlist；
- 使用 argv 数组直接 `uv_spawn()`，不得拼接 shell command；
- cwd 必须是允许目录；
- 子进程环境从最小基线构建，只传 allowlist；
- 配置不允许出现 token/password 等明文 secret 字段；
- namespace 和 upstream ID 必须匹配受限字符集，拒绝 `..`、控制字符和空字符串；
- 未识别字段默认拒绝，避免拼写错误静默失效。

### 4.5 Upstream 状态机

第一版已经需要明确状态，而不是用单个 connected 布尔量：

```text
DISABLED
   │ enable
   ▼
STARTING → INITIALIZING → SYNCING_CATALOG → READY
   │             │              │             │ process exit / protocol error
   └─────────────┴──────────────┴─────────────▼
                                               BACKOFF
                                                  │ bounded retry
                                                  └────→ STARTING

READY → DRAINING → STOPPED
```

只有 READY upstream 的 descriptor 可执行。BACKOFF 时 descriptor 可以保留用于稳定目录展示，但执行必须返回明确的 upstream-unavailable tool error；是否从 public catalog 隐藏由策略决定，不能在重连抖动时反复无序注册/注销。

### 4.6 Route binding 的所有权

当前 registry 只浅拷贝 `handler_data`。建议由 upstream catalog 长期拥有 `mcp_remote_tool_binding`：

```text
public_name
upstream_id
remote_name
catalog_generation
policy snapshot
```

REMOTE_SERVER descriptor 的 `handler_data` 指向该 binding。catalog 更新时必须按“构建新 generation → registry 原子替换该 source → 等旧调用不再引用 → 释放旧 generation”的顺序处理。

第一版不要在多个 `register/unregister` 调用之间直接暴露半更新目录。建议新增 registry 的 `replace_source_generation()`，先复制和校验整批 descriptor，再在 event-loop 线程一次交换。

### 4.7 请求身份必须先修正

当前 core in-flight 只用客户端 JSON-RPC id 的 typed key，多连接可重复 ID。Gateway MVP 的前置改动是新增服务器生成的唯一 `request_token`：

```text
request_token = monotonically generated opaque value
client identity = session_id + client_jsonrpc_id
upstream request id = upstream_id + request_token + attempt
```

- core in-flight 主键改为 request token；
- session 内建立 client ID 到 request token 的取消索引；
- upstream manager 只使用 request token，不拼接不可信 client ID；
- 迟到响应必须同时匹配 upstream、generation 和 upstream request ID。

这个改动应在接入第二个外部客户端之前完成，不能靠“客户端通常不会撞 ID”规避。

## 5. 工具命名和冲突处理

默认采用稳定 namespace：

```text
<upstream_alias>.<remote_tool_name>
```

例如：

```text
github.search_issues
filesystem.read_file
buildfarm.run_job
```

不建议默认把远端工具名无前缀地混入本地 registry，因为不同 upstream 很容易出现 `read_file`、`search`、`status` 等冲突。

兼容策略：管理员可以为个别工具配置 alias，但 alias 必须全局唯一。冲突时应拒绝加载新映射，不得静默覆盖已有工具。

## 6. 工具目录生命周期

```text
upstream 配置加载
  → 建立连接
  → initialize / capability negotiation
  → 调用远端 tools/list
  → 校验工具名和 schema
  → 添加 namespace 与安全元数据
  → 生成新的聚合 registry generation
  → 新客户端 tools/list 获得新目录
```

已有 session snapshot 不应被静默修改。推荐两种通知方式：

1. 支持时发送 `notifications/tools/list_changed`，提示客户端刷新。
2. 不支持通知时，旧 snapshot 保持可解释的稳定语义；工具被安全下线后，即使旧 snapshot 中存在，也必须在执行阶段返回明确的“工具已不可用/请刷新”错误。

工具目录更新建议采用 copy-on-write generation：先完整构建新目录，校验成功后一次性发布，避免客户端看见半更新状态。

## 7. 外部调用生命周期

```text
客户端 tools/call
  → core 验证 JSON-RPC 与 session 状态
  → 建立本地 in-flight entry
  → gateway 校验 session snapshot
  → authorization + schema + risk policy
  → 根据 descriptor 解析 upstream 和远端工具名
  → 检查限流、并发舱壁和熔断器
  → 计算剩余 deadline
  → upstream manager 分配新的远端 request ID
  → transport adapter 发出 tools/call
  → 记录 local ID ↔ upstream ID 映射
  → 收到远端 result/error
  → 校验大小和结构并进行必要脱敏
  → core 按原 reply target 回复客户端
  → 同时清理两侧 in-flight 状态
```

本地 request ID 不能直接假设在所有 upstream 上唯一。必须保存双向映射，并处理连接重建后迟到的响应。

## 8. 超时、取消和重试

### 8.1 Deadline

应使用一个端到端 deadline，而不是每层重新获得完整 timeout：

```text
client/request timeout
  → gateway 计算绝对 deadline
  → 排队、连接和远端执行共享剩余时间
```

否则排队 10 秒后仍给远端完整 30 秒，会破坏客户端原本的 30 秒上限。

### 8.2 取消

收到客户端取消时：

1. 标记本地 in-flight cancelled。
2. 若 upstream 声明支持取消，则传播远端取消通知。
3. 停止向客户端发送正常成功结果。
4. 释放本地映射；迟到的远端响应只更新指标，不得误配给其他请求。

### 8.3 重试

默认不重试 `tools/call`。只有同时满足以下条件才允许：

- descriptor 明确标记 idempotent/retryable；
- 错误属于连接前失败、明确的临时错误或允许重试的响应；
- deadline 仍有足够余量；
- 使用有限次数和指数退避加随机抖动；
- 不因重试突破并发和流量配额。

未知执行结果的断线不能自动重试非幂等工具，否则可能重复产生副作用。

## 9. 故障隔离与流量治理

每个 upstream 独立维护：

- 最大连接数；
- 最大并发请求数；
- 有界等待队列；
- token bucket 或 leaky bucket 速率限制；
- 熔断器；
- 超时和响应大小上限。

建议熔断状态：

```text
CLOSED → 连续/窗口失败超过阈值 → OPEN
OPEN → 冷却期结束 → HALF_OPEN
HALF_OPEN → 探测成功 → CLOSED
HALF_OPEN → 探测失败 → OPEN
```

一个 upstream 的连接堆积不能占满全局 in-flight、内存或 event loop。除 per-upstream 舱壁外，还需要全局上限。

## 10. 安全设计

### 10.1 出站访问控制

- endpoint 必须符合 scheme allowlist。
- 解析后的每个 IP 都要检查网络策略，防止 DNS rebinding 绕过。
- 默认拒绝 loopback、link-local、云元数据地址和未授权私网段；只有显式配置才放行。
- 重定向后的目标必须重新校验。
- TCP/HTTP 响应大小、header 数量和 JSON 深度必须受限。

### 10.2 认证信息

- 配置中只保存 secret reference。
- secret 从环境、受限文件描述符或 secret manager 注入。
- 日志、状态工具和错误响应不得输出 token、Authorization header 或完整凭据路径。
- 支持凭据轮换时，创建新 credential snapshot，再替换新连接使用的凭据。

### 10.3 调用授权

授权至少使用以下维度：

- 客户端身份或 session principal；
- upstream ID；
- 工具名；
- 工具风险等级；
- 参数约束；
- 时间、并发和配额。

“工具出现在 `tools/list`”不能替代执行时授权，因为权限可能在 snapshot 生成后被撤销。gateway 应在每次调用时重新执行关键授权检查。

### 10.4 内容与错误隔离

- 远端返回内容视为不可信输入。
- 校验 JSON-RPC version、ID、result/error 互斥关系和最大嵌套深度。
- 对外返回稳定的 gateway 错误码，不直接泄露内部地址、凭据、堆栈或底层库错误。
- 审计日志记录元数据和摘要，不默认记录完整参数或结果。

## 11. 可观测性

建议指标：

- 每个 upstream 的连接状态和重连次数；
- `tools/list` 刷新成功、失败和缓存年龄；
- 调用总数、成功、错误、取消、超时和拒绝数；
- 排队时间、远端耗时和端到端耗时直方图；
- 当前 in-flight 和队列深度；
- 重试次数、熔断状态和限流拒绝数；
- 请求/响应字节数，但不记录敏感正文。

审计事件建议包含 trace ID、客户端主体、公开工具名、upstream ID、策略结果、最终状态和耗时。敏感参数只记录字段名或经过批准的不可逆摘要。

## 12. 建议模块与文件边界

为避免继续扩大 `server.c` 和 `gateway.c`，建议按职责增加小模块：

```text
include/mcp/upstream/upstream.h
src/upstream/upstream_manager.c
src/upstream/upstream_config.c
src/upstream/upstream_catalog.c
src/upstream/upstream_request.c
src/upstream/circuit_breaker.c
src/upstream/rate_limiter.c
src/upstream/adapters/stdio_adapter.c
src/upstream/adapters/http_adapter.c
src/upstream/adapters/peer_adapter.c
```

建议依赖方向：

```text
core → gateway → upstream manager → adapter
                  ↓
               registry
```

adapter 不直接操作客户端 reply target；它只把远端完成事件交还 upstream manager，再由 gateway/core 完成本地请求。

### 12.1 稳定 adapter 接口

transport adapter 只处理上游字节和连接生命周期，不直接写 registry，也不持有本地客户端 reply target。建议最小接口语义：

```c
struct mcp_upstream_adapter_ops {
    int (*start)(void *adapter, const struct mcp_upstream_config *config);
    int (*send_request)(void *adapter,
                        const char *upstream_request_id,
                        const char *method,
                        json_t *params,
                        unsigned long long deadline_ms);
    int (*cancel)(void *adapter, const char *upstream_request_id);
    void (*begin_drain)(void *adapter);
    void (*destroy)(void *adapter);
};
```

adapter 通过回调向 manager 报告：ready/capabilities、response、notification、transport error 和 closed。manager 负责 request mapping、catalog、策略和重连；因此加入 HTTP adapter 时不需要修改 gateway route。

### 12.2 stdio adapter 的具体责任

- 用 `uv_spawn()` 直接执行 allowlisted binary；
- 建立 stdin/stdout pipe，stderr 单独限速采集且永不混入协议 stdout；
- stdout 按换行恢复完整 JSON，限制单条消息和累计缓冲；
- 完成指定 revision 的 lifecycle 后才报告 READY；
- 为 tools/list/tools/call 生成独立 upstream request ID；
- 校验 response 的 jsonrpc、id、result/error 互斥关系；
- child exit 时完成所有 pending 为 transport error，并进入有界 backoff；
- drain 时停止新请求，等待 deadline 后终止 child。

### 12.3 Streamable HTTP adapter 的边界

第二阶段引入经过维护的 HTTP/TLS client 库，不建议在 gateway 内用裸 libuv 自行实现 HTTP、TLS、代理、重定向和证书校验。

HTTP adapter 独立负责：URI/TLS/redirect 校验、官方 revision 对应的 header/body/SSE 语义、per-request cancellation、响应大小限制、认证 header 注入和连接池。manager 仍只看到统一 request/response/notification 事件。

### 12.4 现有 peer adapter

现有 discovery 的 initialize、tools cache、pending proxy 和 framed TCP 可逐步包成 `peer_adapter`。第一版不必先重写 discovery；可以让 manager 的 peer adapter 调用已有 discovery API，待标准 adapter 稳定后再决定是否合并重复的 cache/request 状态。

必须保留边界：peer adapter 是本项目内部节点互联兼容层；对外宣称支持的标准 transport 只有实际完成互操作测试的 stdio/Streamable HTTP。

## 13. 分阶段实施

### 阶段 0：先固定现有行为和请求身份

实现内容：

- 为现有 built-in、plugin、`gateway.proxy_tool`、session snapshot 建立回归契约。
- 建立可脚本控制的 fake stdio MCP server，可制造延迟、坏 JSON、重复/迟到响应、退出和工具目录变化。
- 引入 server-generated request token，消除多 session 相同 client ID 的冲突。
- 定义 adapter/manager/catalog 接口，但暂不改变默认调用路径。

完成判据：现有 smoke tests 不回归；两个并发客户端都使用 ID 1 时能分别完成和取消；fake upstream 测试可稳定复现所有故障模式。

### 阶段 1：静态 stdio upstream 与显式诊断调用

实现内容：

- 严格解析一份静态 upstream 配置。
- 实现 upstream manager 状态机和 stdio adapter。
- 完成 `2024-11-05` initialize/initialized、tools/list 和 tools/call。
- 暂时提供仅管理/诊断用途的显式 upstream call API，以验证 request mapping、deadline、child exit 和响应校验。
- 加入最大消息、最大 pending、每 upstream 并发和基础指标。

完成判据：真实/fixture stdio MCP server 可被启动、初始化、列工具和调用；进程退出、超时、非法响应都能完成本地 in-flight，且没有 secret/argv 泄漏。

### 阶段 2：namespaced 工具聚合和 REMOTE_SERVER route

实现内容：

- 校验远端 tools/list 和 input schema。
- 生成 `<upstream>.<remote_tool>` binding 和 descriptor。
- 实现 registry source-generation 原子替换。
- 在 `mcp_gateway_call()` 中实现 `REMOTE_SERVER` 普通路由。
- 保留 `gateway.proxy_tool` 作为内部 peer 的兼容/诊断路径，不作为新外部接口。
- 工具目录变化保持 session snapshot 语义；旧 session 不自动扩权。

完成判据：本地客户端通过正常 `tools/list` 看见 namespaced 外部工具并直接 `tools/call`；同名工具不冲突；upstream 下线后旧 snapshot 调用得到稳定 unavailable 错误；catalog 更新期间不暴露半目录。

### 阶段 3：取消、drain 与基础故障隔离

实现内容：

- 端到端 absolute deadline 和剩余时间传播。
- stdio legacy lifecycle 的 `notifications/cancelled` 传播；不支持时本地取消并丢弃迟到响应。
- bounded queue、每 upstream 并发舱壁和基础 CLOSED/OPEN/HALF_OPEN 熔断。
- server shutdown 时 upstream 进入 DRAINING，并有全局 drain deadline。
- 结构化 metrics 和脱敏审计元数据。

完成判据：一个 upstream 卡死/崩溃不会耗尽全局 in-flight；取消或关闭最终都能归零；非幂等请求不会因未知执行结果自动重发。

### 阶段 4：Streamable HTTP 与协议 revision 抽象

实现内容：

- 引入经过审计的 HTTP/TLS client 依赖。
- 实现 Streamable HTTP adapter，并按协商 revision 选择 lifecycle/取消/通知语义。
- 将当前 legacy initialize lifecycle 与新 revision 行为封装在 protocol session 层，不泄漏到 gateway。
- 加入认证引用、TLS、redirect 和网络策略。

完成判据：通过官方兼容 server 的互操作测试；stdio 与 HTTP 使用相同 manager/catalog/gateway 契约；HTTP 安全边界测试覆盖 redirect、DNS/IP、证书、响应大小和认证脱敏。

### 阶段 5：复杂功能按需求增加

- 安全配置热更新、凭据轮换、upstream drain 和灰度发布；
- 仅对明确 idempotent/retryable 工具做有限重试；
- rate limit、租户配额、细粒度授权；
- 多实例负载均衡和健康路由；
- catalog TTL/change subscription；
- OpenTelemetry trace、完整审计出口；
- 只有存在遗留依赖时才增加 legacy HTTP/SSE adapter。

每个阶段都必须保持功能开关默认关闭时不改变当前本地路径，并提供向前兼容的数据结构而非在 gateway 中堆叠 transport 条件分支。

## 14. 验证计划

### 单元测试

- 配置 schema、endpoint 和网络策略校验；
- request token、session client ID 索引和重复 ID；
- namespace 和冲突处理；
- request ID 双向映射；
- deadline 计算；
- 重试资格判断；
- 熔断器和 rate limiter 状态机；
- secret 和错误脱敏。

### 集成测试

- fake stdio upstream 的 initialize、tools/list 和 tools/call；
- stdout 半行/多行、stderr 噪声、超长行和 child exit；
- 远端工具上线、变更和下线；
- 客户端 snapshot 与 catalog generation 的交互；
- 取消、超时、断线、迟到响应和重复响应；
- 多 upstream 同名工具；
- 一个 upstream 故障时其他 upstream 和本地工具不受影响。

### 端到端测试

- 客户端通过 gateway 调用外部工具并收到原 transport 上的响应；
- 权限撤销后旧 snapshot 中的工具也不能继续执行；
- 非幂等调用在未知执行结果时不会自动重试；
- graceful shutdown 能 drain 或明确终止所有外部请求；
- 日志和状态接口不泄露认证信息及敏感参数。

### 每阶段强制回归

- 本地 built-in 与 plugin 路由结果不变；
- stdio/UDP/TCP/pipe 原 ingress 行为不变；
- `gateway.proxy_tool` 的内部 peer 兼容路径不变，除非有明确迁移测试；
- 功能开关关闭时不创建 upstream 子进程、连接、工具或 timer；
- 所有分配失败和半初始化路径可安全 destroy；
- 测试先于实现编写，并覆盖成功、拒绝、超时、取消、断线和 shutdown。

## 15. 建议的最小可行版本

MVP 精确范围是阶段 0～2，不包含阶段 3～5：

1. 静态 JSON 配置，启动后不可热更新。
2. 一个或多个受控 stdio child upstream，只支持明确配置的 `2024-11-05` compatibility lifecycle。
3. server-generated request token，修复多 session client ID 冲突。
4. initialize、tools/list、tools/call 三类上游交互。
5. `<upstream>.<tool>` namespace 和 `REMOTE_SERVER` descriptor。
6. registry source-generation 原子发布和既有 session snapshot 语义。
7. 每 upstream 最大并发、有界 pending、connect/request timeout、最大消息和 child restart backoff。
8. 不自动重试任何 tools/call；不做热更新、负载均衡、HTTP、认证轮换和多租户授权。
9. 外部调用错误统一转换成稳定 tool error，不泄露 command、cwd、环境或底层堆栈。
10. 功能开关默认关闭，关闭时现有行为完全不变。

MVP 验收场景：

```text
启动 gateway
  → 拉起 fake/真实 stdio upstream
  → catalog READY
  → 本地客户端 initialize + tools/list
  → 看见 files.read_file
  → tools/call 成功返回
  → 再验证超时、child exit、非法 response、重复 client ID
  → shutdown 后 child、timer、pending、registry binding 全部归零
```

完成这些条件后，架构已经证明“配置 → catalog → route → request mapping → response → cleanup”的闭环。后续 Streamable HTTP、授权、熔断、重试和动态配置都可以在 adapter/manager/policy 边界内增加，无需重写 core 请求生命周期。

## 16. 静态证据来源

- `docs/module_topology.md:154-163`：gateway 的职责、路由和 session snapshot。
- `docs/module_topology.md:209-217`：discovery、远端工具缓存和代理调用。
- `include/mcp/tools/tool.h`：`REMOTE_SERVER` route、descriptor 和 invocation。
- `include/mcp/gateway/gateway.h`：gateway 公开接口和 proxy tool。
- `src/gateway/gateway.c`：snapshot 校验、本地/插件/远端分派和计数器。
- `src/discovery/server_discovery.c`：peer 生命周期、工具缓存和远端 pending 请求。
- `src/transport/peer_transport.c`：节点间 JSON 与自定义帧传输。
- `src/core/in_flight.h`：本地请求的回复目标、取消、timer 和异步上下文。
- `src/network/network_access_policy.c`：网络开关、IP allowlist 和预配置 peer。

以上方案中的新增模块、标准 transport adapter、完整鉴权授权、熔断和限流属于建议设计，不能从当前源码推断为已经实现。

## 17. 可行性风险与控制措施

| 风险 | 为什么会发生 | MVP 控制措施 | 后续扩展点 |
| --- | --- | --- | --- |
| 多客户端 request ID 冲突 | 当前 in-flight 只用 client ID key | 阶段 0 引入 request token | 分布式 trace/request identity |
| catalog 半更新 | 当前 registry 是逐项 register/unregister | source-generation 构建后原子交换 | TTL、subscription、跨实例 catalog |
| route binding 悬空 | registry 浅拷贝 handler_data | binding 由 generation 持有，drain 后释放 | 引用计数/epoch reclamation |
| 子进程协议污染 | 外部 server 把日志写 stdout | stdout 仅协议，stderr 单独限流采集 | 结构化 OpenTelemetry |
| 上游卡死 | 无通用 gateway timeout | absolute deadline + bounded pending | 熔断、配额、负载均衡 |
| 重复副作用 | 断线后未知是否已执行 | MVP 零自动重试 | 仅幂等且结果明确时有限重试 |
| shutdown 卡住 | pending 永不完成 | drain deadline，超时统一完成错误 | 持久 task/恢复语义 |
| 协议 revision 漂移 | 当前项目固定 2024-11-05，官方协议持续演进 | 配置固定并拒绝不兼容 revision | 独立 lifecycle codec/negotiation |
| HTTP 安全复杂度 | TLS、redirect、DNS、代理均可引入漏洞 | MVP 不做 HTTP | 使用成熟库并逐项安全测试 |
| 旧 snapshot 与安全撤权 | snapshot 是历史可见性 | 每次执行重新检查实时 route/policy | policy generation 与审计 |

总体可行性判断：

- **高可行**：request token、stdio adapter、静态配置、tools catalog、REMOTE_SERVER route；都能沿用 libuv/Jansson/registry/gateway 现有结构。
- **中等可行**：Streamable HTTP、认证、动态 catalog；需要新增成熟 HTTP/TLS 依赖和 protocol revision 层。
- **高复杂度**：多租户授权、跨实例负载均衡、持久任务、无中断动态配置；应在 MVP 闭环和可观测性稳定后再开发。

## 18. 官方协议参考与版本边界

- [MCP 2026-07-28 发布说明](https://blog.modelcontextprotocol.io/posts/2026-07-28/)：新 revision 的无握手/无协议 session 生命周期、Streamable HTTP 路由头、list 缓存和 legacy HTTP+SSE 退场方向。
- [官方 transport 演进说明](https://blog.modelcontextprotocol.io/posts/2025-12-19-mcp-transport-future/)：stdio 面向本地部署、Streamable HTTP 面向远程部署，自定义 transport 仍可用于专用场景。
- [官方 TypeScript SDK revision 迁移说明](https://ts.sdk.modelcontextprotocol.io/v2/migration/support-2026-07-28)：stdio/HTTP 在不同 revision 下的 lifecycle 与取消语义差异，可作为互操作测试参照；本项目不依赖该 SDK 实现。

版本边界必须写入代码和测试：当前服务端响应硬编码 `2024-11-05`，所以方案不能直接宣称兼容最新 revision。第一版 external stdio adapter 是受限 compatibility bridge；只有完成相应 wire contract 和互操作测试后，才能在支持列表中加入新的 revision 或 Streamable HTTP。
