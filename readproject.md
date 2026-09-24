# MCP Server 项目阅读笔记

本文档依据以下两类静态证据整理：

- 模块分解文档：`docs/module_topology.md`
- 项目源码以及 `.codegraph/codegraph.db` 中的符号和调用关系

证据边界：本文能够说明源码当前表达的结构、依赖和控制流，但没有执行编译、测试或真实网络通信，因此不把静态结论表述成运行时或部署环境证明。

## 阅读路线

1. 全局流程与数据种类
2. 进程启动与对象生命周期
3. 输入入口：stdio、UDP、TCP 和 pipe
4. JSON-RPC 消息生命周期
5. 工具注册表与会话快照
6. `tools/call` 完整生命周期
7. 插件与文件传输
8. 服务发现与远端调用
9. 关闭、错误处理与资源清理
10. 端到端串联与外部 Gateway 增强方向

本文档已经写入全部十个步骤，适合离线连续阅读。用户可动态维护每一步状态：

- **未阅读**：尚未开始或本轮要求重新阅读；
- **阅读中**：已经开始，但理解检查尚未完成；
- **已阅读**：已经完成理解检查，并在该步末尾记录自己的理解。

文档内容是否已经写入与阅读状态相互独立；不需要等待前一步确认才能看到后续章节。

---

## 第 1 步：全局流程与数据种类

状态：**未阅读**

### 1.1 项目的整体定位

这个项目不是某一个具体工具的实现，而是一个 MCP Server 运行平台。它负责接收 MCP 客户端请求，维护客户端会话，公布工具列表，并把工具调用分派到本地内建工具、插件或远端 MCP Server。

可以先把完整系统理解为五类参与者：

1. **MCP 客户端**：发出 `initialize`、`tools/list`、`tools/call` 等 JSON-RPC 消息。
2. **本项目的 MCP Server**：接收消息、维护状态、校验请求并协调各模块。
3. **本地工具**：由当前进程直接执行的内建 handler。
4. **插件工具**：由 plugin manager 加载和调用。
5. **远端 MCP Server**：经 discovery 和 peer transport 被发现及代理调用。

### 1.2 六个逻辑层次

```text
启动层            src/main
   ↓
传输与监听层      src/transport + src/listener
   ↓
协议与核心层      src/protocol + src/core
   ↓
目录与路由层      src/registry + src/gateway
   ↓
执行与扩展层      src/tools + src/plugin + src/discovery
   ↓
策略与基础设施层  src/network + src/common
```

各层的核心边界：

- `main` 决定启用哪些入口，并建立和运行 libuv 事件循环。
- transport/listener 只解决字节如何进入和离开进程。
- protocol 把字节中的 JSON 文本解释成 JSON-RPC 消息。
- core 是中心协调者，管理队列、会话、请求状态和回复目标。
- registry 管理“有哪些工具及其元数据”。
- gateway 管理“一次工具调用应当走哪条执行路径”。
- tools/plugin/discovery 分别承接本地、插件和远端能力。
- network/common 提供准入策略和共享基础能力。

### 1.3 六类核心数据

#### A. 启动配置

来源主要是环境变量和 `mcp_server_config`。它决定传输入口、地址端口、消息大小限制、初始化约束和 discovery 是否启用。

生命周期：

```text
环境变量 → main 中解析 → mcp_server_config → mcp_server 保存 → 各子模块初始化
```

#### B. 原始传输数据

它可能是一行 stdio JSON、一个 UDP datagram，或者 TCP/pipe 上的一帧数据。此时它仍只是字节，不具备工具调用语义。

#### C. JSON-RPC 消息

协议层解析后形成 `mcp_jsonrpc_message`，主要包含：

- 消息类型：request 或 notification
- `method`
- `id`
- `params`

#### D. 长期状态

长期状态包括客户端 session、工具 registry、每个 session 的工具快照、插件信息和远端节点缓存。

需要区分：

- registry 是服务器此刻拥有的工具目录。
- session snapshot 是某个客户端最近一次看到的工具目录。

因此，registry 新增工具并不意味着旧 session 可以立即调用；客户端需要再次执行 `tools/list` 刷新快照。

#### E. 单次调用状态

一次 `tools/call` 会产生 invocation，并可能产生 in-flight entry：

- descriptor：长期存在的工具说明和路由元数据。
- invocation：本次调用的工具名、参数和 invocation ID。
- in-flight entry：把请求 ID、回复通道、超时、取消状态和异步上下文绑定起来。

#### F. 节点间与文件传输数据

普通控制请求使用 JSON-RPC；文件块等大量二进制数据可使用 peer transport 的自定义帧。例如文件传输插件注册 `MFT1` 帧处理器，使“启动传输”和“传输数据块”走不同的数据路径。

### 1.4 一次请求的全局生命周期

```text
客户端 JSON
  → transport/listener 接收字节
  → protocol 解析 JSON-RPC
  → core 将消息和 reply target 放入队列
  → core 检查 session 状态
  → 请求进入 in-flight 管理
  → gateway 检查 session snapshot 并查询 registry
  → 本地 handler / plugin / discovery 远端代理
  → 同步结果或异步完成
  → core 构造 JSON-RPC response
  → 按原 reply target 返回
  → 移除 in-flight 状态并释放临时对象
```

### 1.5 阅读确认记录

状态已按用户要求重置为“未阅读”。阅读完成后，在此记录自己的理解。

### 1.6 静态证据

- `src/core/server_internal.h:22-69`：session 状态、session 快照以及 `mcp_server` 聚合对象。
- `src/protocol/jsonrpc.h:8-18`：JSON-RPC 消息结构。
- `include/mcp/tools/tool.h:15-45`：descriptor、invocation 和工具 route。
- `src/core/in_flight.h:13-41`：reply target 与 in-flight entry。
- `src/gateway/gateway.c:157-239`：统一调用入口及路由分支。
- CodeGraph 静态调用关系确认 `handle_request → mcp_gateway_call`，以及 gateway 到插件管理器等下游关系。

---

## 第 2 步：进程启动与对象生命周期

状态：**未阅读**

### 2.1 本步要回答的问题

这一阶段不处理具体请求，而是建立“以后能够接收并处理请求”的运行环境。需要理解四个问题：

1. 谁读取启动配置？
2. `mcp_server` 为什么是对象所有权中心？
3. 初始化和启动为什么是两件事？
4. 启动失败或进程退出时，资源如何回收？

### 2.2 第一阶段：`main()` 形成启动配置

入口位于 `src/main/main.c:200`。`main()` 先构造 `mcp_server_config`，然后读取环境变量决定：

- 是否启用 stdio、UDP、pipe 和 TCP；
- 是否启用 discovery；
- UDP/TCP 的监听地址和端口；
- pipe 路径；
- discovery 绑定、广播和显式 peer 配置；
- 最大消息长度；
- stdio EOF 是否触发服务器退出；
- 是否严格等待客户端的 `initialized` notification。

如果四种入口都被关闭，代码会重新启用 stdio。这是一个启动兜底，防止服务器创建成功后完全没有输入通道。

这里的数据变化是：

```text
环境变量字符串
  → env_bool/env_uint/env_str
  → 局部启动选项 + mcp_server_config
  → 后续传给 mcp_server_init 和 mcp_server_start_*
```

重要边界：`main()` 决定进程级部署选择，但不处理 JSON-RPC 业务。

### 2.3 第二阶段：创建 libuv event loop

`main()` 在 `src/main/main.c:259-263` 调用 `uv_loop_init()`。

event loop 是所有异步事件的调度基础，后续的 stdio、网络连接、定时器、signal 和 core async handle 都依赖它。它由 `main()` 在栈上创建，并把地址交给 server；因此：

- `main()` 拥有 loop 的存储期；
- `mcp_server` 只保存 `uv_loop_t *` 并使用它；
- 必须先销毁/关闭依赖 loop 的 handle，最后才能 `uv_loop_close()`。

### 2.4 第三阶段：`mcp_server_init()` 建立对象图

`main()` 在 `src/main/main.c:265` 调用 `mcp_server_init()`。初始化主体位于 `src/core/server.c:895-963`。

首先分配并建立根对象：

```text
calloc(mcp_server)
  → 保存 loop 和 config
  → 初始化 stdio session
  → 初始化 in-flight map
```

然后按依赖关系建立子对象：

```text
mcp_server
├── network_access_policy
├── shell_policy_snapshot
├── sandbox_control
├── stdio transport
├── UDP transport（构建特性启用时）
├── core_async
├── tool registry
├── peer transport
├── plugin manager
├── gateway
├── discovery
├── shell jobs
├── builtin plugins
└── builtin tools
```

`mcp_server` 是对象所有权中心的原因，不只是它持有这些指针，还因为请求处理时需要同时访问其中多个模块。例如一次工具调用可能同时需要 session snapshot、registry、gateway、plugin manager、discovery 和 in-flight map。

### 2.5 为什么先策略、再执行组件

网络策略、shell 策略快照和 sandbox control 在 transport、gateway 和工具注册之前创建。这样后续组件一旦开始工作，就已经有明确的准入边界，而不是先开放能力再补策略。

这里的 shell policy 使用“启动时快照”语义：初始化时读取环境形成固定策略对象，后续执行引用该快照。修改外部环境并不会自动改变已经运行的 server；若要动态更新，需要额外的配置重载设计。

### 2.6 为什么先建 registry，再建 gateway

`mcp_gateway_create(&server->gateway, server->registry)` 明确把 registry 交给 gateway。因此依赖方向是：

```text
registry 先存在
   ↓
gateway 保存并查询 registry
   ↓
请求到来后按 descriptor 做路由
```

gateway 不是工具存储容器，而是 registry 的消费者。如果 registry 尚未创建，gateway 就没有可查询的工具目录。

### 2.7 为什么“初始化”和“启动监听”分开

`mcp_server_init()` 只建立内部能力和对象图。真正打开外部入口由后续函数完成：

- `mcp_server_start_stdio()`
- `mcp_server_start_udp()`
- `mcp_server_start_pipe()`
- `mcp_server_start_tcp()`
- `mcp_server_start_discovery()`

分离带来三个效果：

1. 同一个 server 可以按部署需求启用不同入口组合。
2. 所有核心依赖可以在接受第一条外部消息前准备完成。
3. 某个入口启动失败时，可以统一调用 `mcp_server_destroy()` 回滚整个对象图。

注意 discovery 的额外约束：当前 `main()` 只在 TCP 和 discovery 都启用时启动 discovery。这反映远端节点发现最终需要一个可通告、可连接的 TCP 服务端点。

### 2.8 第四阶段：进入事件循环

入口启动后，`main()` 安装 shutdown signal，并在 `src/main/main.c:348` 调用：

```c
uv_run(&loop, UV_RUN_DEFAULT);
```

从此控制方式由“顺序执行启动代码”切换为“事件驱动”：

```text
输入到达 / 连接建立 / timer 到期 / signal 到达
                  ↓
             libuv callback
                  ↓
       transport、listener 或 core 继续处理
```

因此 `uv_run()` 不是处理业务的模块，它是让各模块注册的异步 handle 和 callback 获得执行机会。

### 2.9 启动失败的回滚路径

初始化中的任何关键步骤失败都会调用 `mcp_server_destroy(server)`。因为 server 使用 `calloc` 清零，尚未创建的成员保持空指针，而各 destroy/close 路径按“允许部分初始化”的方式处理。

这一模式可以概括为：

```text
逐步构造
  → 任一步失败
  → 对当前半成品调用统一 destroy
  → destroy 只释放已经成功建立的成员
```

它避免了每个失败分支手工复制一套不同的资源释放代码。

### 2.10 正常关闭的资源顺序

事件循环退出后，`main()` 先关闭 signal handles，再次运行 event loop 让关闭回调完成，然后调用 `mcp_server_destroy()`，最后关闭 loop。

`mcp_server_destroy()` 的主要顺序是：

1. 关闭运行时 handles。
2. 清空尚未处理的消息队列。
3. 清理 stdio session 和所有 peer session。
4. 停止 shell jobs。
5. 销毁 gateway、plugin manager、discovery 和 shell jobs。
6. 销毁 sandbox、策略快照、peer transport、registry 和 in-flight map。
7. 销毁具体传输与 listener。
8. 销毁网络策略并释放 server。
9. 回到 `main()`，最后关闭 libuv loop。

这不是创建顺序的机械完全逆序，而是按运行依赖先“停止活动”，再释放被活动对象引用的底层资源。例如先停止 jobs，再销毁策略和传输。

### 2.11 本步的数据生命周期

```text
环境变量
  → main 的启动配置
  → mcp_server 保存配置
  → 配置派生出策略、transport 和各管理器
  → start_* 打开外部入口
  → uv_run 驱动长期运行
  → shutdown 先停止事件来源
  → 清空短期状态
  → 销毁长期对象
  → 关闭 event loop
```

其中三种存活时间必须区分：

- 进程级：libuv loop、`mcp_server`、registry、gateway、plugin manager。
- session 级：stdio session 和每条 peer 连接关联的 session、tool snapshot。
- request 级：消息节点、invocation 和 in-flight entry。

当前第 2 步只建立了进程级背景；session 和 request 的创建细节将在后续步骤展开。

### 2.12 CodeGraph 与源码证据

- `src/main/main.c:200-241`：读取环境变量并形成启动配置。
- `src/main/main.c:259-279`：初始化 loop、server 和 stdio 入口。
- `src/main/main.c:283-336`：按开关启动 UDP、pipe、TCP 和 discovery。
- `src/main/main.c:338-354`：安装信号、运行 event loop 并执行最终清理。
- `src/core/server.c:895-963`：创建 server 对象图。
- `src/core/server.c:965-1006`：销毁对象图。
- `src/core/server_internal.h:37-69`：server 实际持有的核心成员。
- CodeGraph 静态调用关系确认 `main → mcp_server_init`，以及 `main → mcp_server_start_stdio/udp/pipe/tcp/discovery`。

### 2.13 理解检查

请确认自己能够解释以下问题：

1. 为什么 libuv loop 由 `main()` 创建，却由 server 内的很多模块使用？
2. 为什么 registry 必须在 gateway 之前创建？
3. `mcp_server_init()` 和 `mcp_server_start_*()` 分别负责什么？
4. 为什么销毁时要先停止活动来源，再释放策略、registry 和 transport 等底层对象？

### 2.14 阅读确认记录

状态已按用户要求重置为“未阅读”。阅读完成后，在此记录自己的理解。

---

## 第 3 步：stdio、UDP、TCP 和 pipe 输入入口

状态：**未阅读**

阅读状态：由用户后续动态更新。

### 3.1 本步要回答的问题

第 2 步已经说明四种入口如何启动。本步沿着一条消息真正进入进程的路径，回答以下问题：

1. 四种入口分别用什么规则判断“一条完整消息已经到达”？
2. 为什么 TCP 和 pipe 共用 listener，而 stdio 和 UDP 使用各自的 transport？
3. core 如何记住响应应该返回到 stdout、UDP 来源地址，还是某条 TCP/pipe 连接？
4. 输入缓冲何时释放，JSON-RPC 消息又由谁接管？
5. 无效长度、无效 JSON、连接关闭和 EOF 分别如何处理？

先记住本步的核心结论：**四种入口只在“如何取得一条完整消息”和“如何找到返回路径”上不同；一旦 JSON-RPC 解析成功，都会形成相同的 `mcp_jsonrpc_message + mcp_reply_target`，再进入同一个 core 队列。**

### 3.2 四种入口的共同归一化模型

四条路径可以先压缩成一张图：

```text
stdio 字节流 ──换行──┐
UDP 数据报 ─────────┤
TCP 字节流 ──长度帧─┼─→ 一段完整 payload
pipe 字节流 ─长度帧─┘
                         ↓
               mcp_jsonrpc_parse_line()
                         ↓
       mcp_jsonrpc_message + mcp_reply_target
                         ↓
                    queue_message()
                         ↓
                  uv_async_send()
                         ↓
                    core_async_cb()
                         ↓
          handle_request / handle_notification
```

这里有两个彼此独立的数据对象：

- `mcp_jsonrpc_message` 表示“客户端说了什么”，保存 request/notification 类型、method、id 和 params。
- `mcp_reply_target` 表示“结果应该回到哪里”，保存 stdio、UDP 地址或 stream connection。

core 不根据 JSON-RPC `id` 猜测连接。消息进入时就把 reply target 与消息绑定，之后即使调用异步完成，in-flight entry 仍会保存同一个 reply target。

### 3.3 stdio：用换行符切分消息

#### 3.3.1 启动

`mcp_server_start_stdio()` 做两件事：

```text
stdin/stdout 文件描述符
  → mcp_stdio_transport_open()
  → mcp_stdio_transport_start(stdio_on_line, stdio_on_exit, server)
```

`open()` 把现有 stdin/stdout 文件描述符交给两个 `uv_pipe_t`；这里的 `uv_pipe_t` 是 libuv 对流式句柄的统一封装，不等于本节稍后讲的“本地 pipe 监听入口”。`start()` 再对 stdin 调用 `uv_read_start()`，注册读回调。

#### 3.3.2 从任意字节块恢复出完整行

底层一次 read 不保证正好读到一条 JSON。可能出现：

- 半条 JSON 分两次到达；
- 一次 read 包含多条 JSON；
- 行尾是 `\n` 或 `\r\n`。

因此 `stdio_transport.c` 维护 `rx_buf/rx_len/rx_cap`：

```text
read_cb 收到字节
  → 追加到 rx_buf
  → 查找 '\n'
  → 对每条完整非空行调用 on_line
  → 从缓冲区消费已处理字节
  → 没有换行的尾部继续保留，等待下一次 read
```

`\r\n` 中的 `\r` 不交给 JSON 解析器；空行被跳过。完整行或尚未闭合的累计内容一旦超过 `max_line_bytes`，transport 会结束 stdio 输入，避免接收缓冲无限增长。

#### 3.3.3 进入 core

`stdio_on_line()` 创建：

```text
reply_to.transport = MCP_REPLY_STDIO
```

然后同步调用 `mcp_jsonrpc_parse_line()`。若解析失败，错误对象直接经 stdout 返回，不进入 core 消息队列；若成功，则调用 `queue_message()`。

stdio 只有一个固定的 `server->stdio_session`。所以 stdio 输入不需要按地址或连接查找 session。

#### 3.3.4 输出与 EOF

输出路径会把 JSON 序列化为紧凑文本并附加 `\n`，然后进入 stdio transport 的写队列。队列确保前一次 `uv_write()` 完成后再继续发送，写缓冲在完成回调中释放。

stdin EOF 的行为取决于 `stdio_eof_shutdown`：

- 只有 stdio 入口时，默认把 EOF 视为整个 server 的关闭信号。
- 同时启用了 UDP/TCP/pipe 时，默认只清理 stdio session 并关闭 stdout，其他入口仍可继续运行。
- `MCP_STDIO_EOF_SHUTDOWN` 可以强制 EOF 关闭整个 server。

这说明 stdio 的生命周期可以短于进程生命周期，但只有一个 stdio session。

### 3.4 UDP：一个 datagram 就是一条消息

#### 3.4.1 启动与天然消息边界

`mcp_server_start_udp()` 依次调用：

```text
mcp_udp_transport_open(bind_host, bind_port)
  → uv_udp_init + uv_udp_bind
mcp_udp_transport_start(udp_on_datagram, udp_on_error, server)
  → uv_udp_recv_start
```

UDP 与字节流不同：一次 datagram 自身就是消息边界，因此不需要查找换行，也不需要 4 字节长度头。transport 为一次接收分配最大 `max_datagram_bytes` 大小的缓冲，并把本次 payload 和来源 `sockaddr` 一起交给 core。

#### 3.4.2 先做来源准入，再解析 JSON

`udp_on_datagram()` 的顺序是：

```text
收到 datagram
  → server 是否正在关闭？
  → network access policy 是否允许来源地址？
  → 来源是否为 IPv4/IPv6？
  → 复制来源地址到 reply target
  → 解析 JSON-RPC
  → 排队
```

不允许的来源被直接丢弃，不返回错误。这一点不同于“来源合法但 JSON 无效”：后者会向原 UDP 地址返回 JSON-RPC parse/invalid-request 错误。

UDP reply target 是：

```text
reply_to.transport = MCP_REPLY_UDP
reply_to.udp_peer  = 来源 IP + 端口
```

因此来自同一 socket 的多个客户端仍可按来源地址区分 session 和响应目标。发送时 `mcp_udp_transport_send()` 会复制响应 payload 和 peer 地址，异步发送完成后释放副本。

#### 3.4.3 UDP 的特殊边界

- UDP 没有“连接建立/断开”，只能依据来源地址识别对端。
- datagram 不会像 TCP 一样自动重组；超长、丢包、乱序和重传不由该 transport 修复。
- 当前 `udp_on_error()` 没有业务处理，只接收状态后返回；因此 transport 层错误不会自动生成客户端 JSON-RPC 响应。
- discovery 的 offline notification 是一个例外：core 可在入口回调中把它交给 discovery 直接消费，不再进入普通消息队列。

### 3.5 TCP：多连接字节流加长度前缀

#### 3.5.1 为什么不能把一次 read 当成一条消息

TCP 只提供有序字节流。一次 read 可能得到半帧、一帧或多帧，所以项目定义了自己的 framing：

```text
+----------------------+---------------------------+
| 4 字节大端 payload 长度 | payload（JSON 或节点二进制帧） |
+----------------------+---------------------------+
```

例如长度头是 `00 00 00 64`，表示后面 payload 长度为 100 字节。长度头本身不计入 payload 长度。

#### 3.5.2 监听和准入

`mcp_server_start_tcp()` 创建 `mcp_framed_listener`，把 `max_line_bytes` 作为 `max_frame_bytes`，然后注册：

- `framed_on_accept`：检查远端网络地址是否被 network policy 允许；
- `framed_on_message`：接收已经去掉 4 字节头的完整 payload；
- `framed_on_close`：连接关闭时移除其 session。

listener 接受连接后，为每条连接创建独立的 `mcp_framed_connection`，每个 connection 都有自己的接收缓冲。因此两个 TCP 客户端的半帧不会混在一起。

#### 3.5.3 累积、拆帧和长度校验

每条连接的 `read_cb()` 把新字节追加到自己的 `rx_buf`，`process_rx()` 循环处理：

```text
缓冲不足 4 字节
  → 等待更多数据

已有 4 字节
  → 读取大端 frame_len
  → frame_len == 0 或超过 max_frame_bytes：关闭连接
  → payload 尚未收全：等待更多数据
  → payload 完整：调用 framed_on_message
  → 消费整帧，继续检查缓冲中是否还有下一帧
```

无效长度会关闭连接，而不是尝试在字节流中猜测下一个边界。这避免失去同步后把后续任意字节误当成合法请求。

#### 3.5.4 JSON 与节点二进制帧的分流

`framed_on_message()` 并非无条件解析 JSON。它先调用 `framed_maybe_dispatch_binary()`：

- 以 `{` 开头的 payload 明确走 JSON-RPC；
- 非 `{` 开头且连接已经绑定 peer server 身份时，可交给 `peer_transport` 的 magic frame handler；
- 文件传输插件的 `MFT1` 就使用这条二进制路径；
- 没有被二进制处理器接收的 payload 再尝试 JSON-RPC 解析，失败则回 JSON-RPC 错误。

所以 TCP framed listener 是一个复用的数据入口：普通客户端 JSON-RPC 和已识别节点的扩展帧共享 framing，但在 core 入口处分流。它不代表所有非 `{` 数据都一定是合法二进制帧。

#### 3.5.5 回复与关闭

TCP reply target 保存当前 connection 指针：

```text
reply_to.transport = MCP_REPLY_STREAM
reply_to.stream    = conn
```

发送响应时，core 会去掉 JSON dump 末尾为 stdio 准备的换行，再由 `mcp_framed_connection_send()` 重新添加 4 字节大端长度头。连接关闭后，`framed_on_close()` 用同一个 reply target 找到并删除对应 session，释放其 tool snapshot。

### 3.6 pipe：地址不同，消息协议与 TCP 相同

这里的 pipe 是 `MCP_PIPE_PATH` 指定的本地 IPC 监听端点：POSIX 默认是 Unix domain socket 路径 `/tmp/mcp-server.sock`，Windows 默认是 named pipe。不要把它与 stdio transport 内部用来包装 stdin/stdout 的 `uv_pipe_t` 混为一谈。

pipe 与 TCP 共用 `mcp_framed_listener`，所以二者具有相同的：

- 4 字节大端长度前缀；
- 每连接独立接收缓冲；
- `max_frame_bytes` 校验；
- `framed_on_message()` JSON/二进制分流；
- `MCP_REPLY_STREAM` 回复方式；
- 连接关闭时清理 session 的行为。

差异主要在建立连接的位置：

| 对比项 | TCP | pipe |
| --- | --- | --- |
| 绑定目标 | IP + port | 本地 pipe/socket 路径 |
| libuv server handle | `uv_tcp_t` | `uv_pipe_t` |
| 接受连接后的 handle | `uv_tcp_t` | `uv_pipe_t` |
| network address allowlist | 接受后检查 | 不走 IP 地址检查 |
| 消息 framing | 4 字节大端长度 | 4 字节大端长度 |
| core 回调 | `framed_on_message` | `framed_on_message` |

因此 TCP 和 pipe 的差异主要停留在 listener 的连接建立层；一旦拿到完整 payload，后续 core 数据生命周期相同。

### 3.7 reply target：把“输入来源”变成“返回地址”

三种 reply transport 覆盖了四个入口：

| 输入入口 | reply transport | 身份/返回依据 | session 归属 |
| --- | --- | --- | --- |
| stdio | `MCP_REPLY_STDIO` | 固定 stdout | 唯一 `stdio_session` |
| UDP | `MCP_REPLY_UDP` | IPv4/IPv6 来源地址和端口 | 按 UDP peer 匹配 |
| TCP | `MCP_REPLY_STREAM` | 当前 framed connection | 按 connection 匹配 |
| pipe | `MCP_REPLY_STREAM` | 当前 framed connection | 按 connection 匹配 |

`reply_targets_equal()` 体现了这种身份规则：stdio 永远与 stdio 相等；stream 比较 connection 指针；UDP 比较同地址族下的 socket address 内容。

这个设计带来一个关键统一点：后续 `handle_request()`、gateway 和工具 handler 不需要各写一套 stdio/UDP/TCP/pipe 版本。异步执行只需把 reply target 保存在 in-flight entry，完成时 `send_json_object()` 再选择正确输出模块。

### 3.8 输入缓冲到队列对象的所有权转移

这是理解数据生命周期最关键的一段：

```text
transport/listener 临时接收缓冲
  → 入口 callback 在缓冲仍有效时同步解析
  → parser 新建 mcp_jsonrpc_message
     - method：复制字符串
     - id：增加 JSON 引用计数
     - params：增加引用计数；缺失时新建空对象
  → queue_message 新建 mcp_message_node
  → node 保存 message 指针并复制 reply target
  → transport/listener 可以消费或释放原始接收缓冲
  → core_async_cb 出队并处理
  → 销毁 message，再释放 node
```

所以队列中不保存指向 stdio/UDP/framed 原始 payload 的裸指针。原始缓冲释放后，method/id/params 仍由新建的 message 持有。

`queue_message()` 分配 node 失败时会立即销毁 message，避免泄漏；成功后 node 被追加到 FIFO 队尾，并调用 `uv_async_send()` 唤醒 core。`core_async_cb()` 一次会持续出队，按 message 类型分派给 request 或 notification handler。

### 3.9 为什么入口回调不直接执行业务

入口回调主要做五件事：

1. 检查 server 是否正在关闭。
2. 建立 reply target。
3. 做入口特有的准入或二进制分流。
4. 解析 JSON-RPC。
5. 把统一消息放入 core 队列。

工具查找、session gate、in-flight 创建和 gateway 调用留给 `core_async_cb()` 后的处理阶段。这样 transport/listener 只负责 I/O 边界，不需要知道 `tools/call` 的具体路由，也让四种入口共享同一套业务处理逻辑。

需要注意：这些 callback 和 core async handle 都运行在同一个 libuv loop 上；这里的队列首先是职责和调度边界，不应仅凭静态代码把它理解为一个带锁的跨线程队列。

### 3.10 错误发生在哪一层

| 错误/事件 | 发现模块 | 当前处理方式 |
| --- | --- | --- |
| stdio 行过长 | stdio transport | 停止输入并触发 exit 流程 |
| stdio EOF/读取失败 | stdio transport | 触发 `stdio_on_exit`，按配置关闭进程或仅 stdio |
| UDP 来源不允许 | core + network policy | 静默丢弃 |
| UDP 接收/发送错误 | UDP transport | 调用 `udp_on_error`；当前 core callback 不继续处理 |
| TCP 来源不允许 | framed listener + core policy callback | 关闭刚接受的连接 |
| TCP/pipe frame 长度为 0 或过大 | framed listener | 关闭该 connection |
| TCP/pipe 读写失败 | framed listener | 关闭该 connection |
| JSON 语法错误 | protocol | 构造 `-32700 Parse error`，按原 reply target 返回 |
| JSON-RPC 结构无效 | protocol | 构造 `-32600 Invalid Request`，按原 reply target 返回 |
| message node 分配失败 | core | 销毁已解析 message；当前实现不发送额外错误 |
| stream 连接关闭 | framed listener/core | 删除对应 session 和 snapshot |

这张表也展示了模块责任：消息边界错误由 transport/listener 处理；JSON-RPC 形态错误由 protocol 表达；来源准入由 network policy 决定；业务方法和 session 错误则在下一阶段由 core 处理。

### 3.11 四条完整输入路径对照

#### stdio

```text
stdin bytes
  → stdio read_cb 累积并按换行切行
  → stdio_on_line
  → parse JSON-RPC
  → reply target = STDIO
  → queue_message
  → core_async_cb
  → stdout 写队列返回带换行 JSON
```

#### UDP

```text
UDP datagram + peer sockaddr
  → udp recv_cb
  → udp_on_datagram
  → network policy
  → parse JSON-RPC
  → reply target = UDP peer
  → queue_message
  → core_async_cb
  → uv_udp_send 返回原 peer
```

#### TCP

```text
TCP connection bytes
  → connection 私有 rx buffer
  → 4-byte big-endian length 拆帧
  → framed_on_message
  → binary dispatch 或 parse JSON-RPC
  → reply target = STREAM connection
  → queue_message
  → core_async_cb
  → 加 4-byte length 后写回同一 connection
```

#### pipe

```text
local pipe/socket connection bytes
  → connection 私有 rx buffer
  → 4-byte big-endian length 拆帧
  → framed_on_message
  → binary dispatch 或 parse JSON-RPC
  → reply target = STREAM connection
  → queue_message
  → core_async_cb
  → 加 4-byte length 后写回同一 connection
```

### 3.12 本步中各模块在完整流程里的作用

- `src/main`：根据部署配置决定开放哪些入口。
- `src/transport/stdio_transport.c`：把 stdin 字节流恢复为行，把响应按顺序写到 stdout。
- `src/transport/udp_transport.c`：收发独立 datagram，并携带来源/目标地址。
- `src/listener/framed_listener.c`：管理 TCP/pipe 监听、多连接、每连接缓冲和长度 framing。
- `src/network`：为 UDP 来源和 TCP 新连接提供准入判断。
- `src/protocol/jsonrpc.c`：把完整 payload 转成统一消息，或构造协议错误。
- `src/core/server.c`：建立 reply target、处理入口特例、排队，并在统一调度点分派消息。
- `src/core/in_flight.h`：定义可跨异步调用保存的 reply target，使后续结果仍能回到原入口。

### 3.13 CodeGraph 与源码证据

源码证据：

- `src/main/main.c:200-241`：四种入口开关、地址、端口、pipe 路径和消息大小配置。
- `src/main/main.c:273-311`：按配置调用四个 `mcp_server_start_*()`。
- `src/core/server.c:55-84`：按 reply target 选择 stream、UDP 或 stdio 输出。
- `src/core/server.c:122-187`：reply target 相等规则以及 session 查找/创建。
- `src/core/server.c:238-263`：message 与 reply target 入队并唤醒 core。
- `src/core/server.c:696-712`：统一出队和 request/notification 分派。
- `src/core/server.c:714-879`：stdio、UDP、framed 的入口 callback、特例和连接关闭处理。
- `src/core/server.c:1009-1105`：四种入口的启动函数。
- `src/transport/stdio_transport.c:213-274`：stdio 累积、换行切分、长度限制和 callback。
- `src/transport/stdio_transport.c:350-395`：stdio 异步写队列入口。
- `src/transport/udp_transport.c:76-109`：datagram 接收边界及缓冲释放。
- `src/transport/udp_transport.c:182-225`：UDP bind 和开始接收。
- `src/listener/framed_listener.c:183-232`：4 字节长度解析、累计与多帧消费。
- `src/listener/framed_listener.c:273-308`：接受连接、TCP 地址准入和开始读取。
- `src/listener/framed_listener.c:344-455`：pipe/TCP 启动和 framed response 发送。
- `src/protocol/jsonrpc.c:135-210`：JSON-RPC 校验及 message 所有权建立。
- `src/core/in_flight.h:13-34`：reply transport、reply target 和 in-flight 保存关系。

CodeGraph 索引确认的静态调用关系包括：

```text
mcp_server_start_stdio → mcp_stdio_transport_open/start
mcp_server_start_udp   → mcp_udp_transport_open/start
mcp_server_start_pipe  → mcp_framed_listener_create/start_pipe
mcp_server_start_tcp   → mcp_framed_listener_create/start_tcp

stdio_on_line     → mcp_jsonrpc_parse_line → queue_message
udp_on_datagram   → mcp_jsonrpc_parse_line → queue_message
framed_on_message → mcp_jsonrpc_parse_line → queue_message

core_async_cb → handle_request / handle_notification
framed read_cb → process_rx
process_rx → read_u32_be / rx_consume
```

静态索引边界：transport 把 `stdio_on_line`、`udp_on_datagram`、`framed_on_message` 作为函数指针保存并在运行时回调，这类间接调用不一定在 CodeGraph 的 `calls` 边中完整呈现。因此本节用索引确认显式调用链，同时用回调注册源码补全真实路径；这仍是静态证据，不等于已经进行运行时抓包或端到端验证。

### 3.14 理解检查

后续更新阅读状态时，可以逐项记录：

- [ ] 我能解释为什么 stdio 按换行切分，而 TCP/pipe 必须使用长度前缀。
- [ ] 我能解释 UDP 为什么不需要接收缓冲重组，但必须保存来源地址。
- [ ] 我能说出四个入口分别使用哪一种 reply target。
- [ ] 我能解释原始接收缓冲释放后，排队消息为什么仍然有效。
- [ ] 我能解释为什么入口 callback 只解析和排队，不直接调用工具。
- [ ] 我能区分“非法 frame 长度”“非法 JSON”“来源地址不允许”三种错误由谁处理。
- [ ] 我能解释 TCP/pipe 普通 JSON-RPC 与 peer 二进制帧如何在同一 framed listener 上分流。

全部理解后，可把本节状态更新为“已阅读”，并在下面记录自己的理解。

### 3.15 阅读确认记录

尚未填写。

---

## 第 4 步：JSON-RPC 请求、通知与会话门控

状态：**未阅读**

### 4.1 本步定位

第 3 步结束时，core 队列里已经有统一的 `mcp_jsonrpc_message`。第 4 步解释这个消息如何被识别为 request 或 notification，以及同一输入通道对应的 session 如何从“尚未初始化”推进到“可调用工具”。

核心链路是：

```text
完整 payload
  → JSON 文本解析
  → JSON-RPC 结构校验
  → request / notification 分类
  → core 队列
  → session gate
  → method handler
```

### 4.2 protocol 层实际校验什么

`mcp_jsonrpc_parse_line()` 使用 Jansson 的 `json_loadb()` 读取指定长度的 payload，并启用 `JSON_REJECT_DUPLICATES`。当前校验规则是：

1. 文本必须是合法 JSON，否则返回 `-32700 Parse error`。
2. 根节点必须是 object，否则返回 `-32600 Invalid Request`。
3. `jsonrpc` 必须是字符串 `"2.0"`。
4. `method` 必须是字符串。
5. `id` 若存在，只允许整数或字符串。
6. `params` 若不存在，parser 创建空 object；若存在，parser 在这一层不限定其类型。

这说明 protocol 只验证通用 JSON-RPC 外形。比如 `tools/call.params.name` 是否存在、`arguments` 是否为 object，要到 core 的具体 method 处理中再验证。

### 4.3 request 与 notification 的分类

当前 parser 用 `id` 是否存在分类：

```text
存在合法 id
  → MCP_JSONRPC_REQUEST
  → 必须产生 result 或 error 响应

不存在 id
  → MCP_JSONRPC_NOTIFICATION
  → 不发送普通响应
```

`id: null` 不被接受，因为当前实现只允许整数或字符串。这是当前源码的明确约束，不应假设它接受 JSON-RPC 中所有可能的 ID 表达。

解析成功后，message 获得自己的数据所有权：method 被复制，id 和 params 增加引用计数。原始 transport 缓冲随后可以安全释放。

### 4.4 session 如何与输入来源绑定

`client_session_for_reply()` 使用第 3 步的 reply target 定位 session：

- stdio 始终返回 `server->stdio_session`；
- TCP/pipe 按 framed connection 指针查找；
- UDP 按来源 socket address 查找；
- request 第一次到达时，可按需创建非 stdio session。

因此 session 不是按 JSON-RPC id 创建的。ID 只标识一次 request；reply target 才代表客户端通道。

### 4.5 三态初始化状态机

每个 session 有三个状态：

```text
MCP_SESSION_NOT_INITIALIZED
          │ initialize request
          ▼
MCP_SESSION_AWAIT_CLIENT_INITIALIZED
          │ notifications/initialized
          ▼
MCP_SESSION_INITIALIZED
```

如果配置 `strict_initialized_notification=false`，`initialize` 响应发送后可直接进入 `INITIALIZED`，跳过中间等待态。

`gate_allows_method()` 的规则很小但非常关键：

- `ping` 在任何 session 状态都允许；
- `initialize` 只允许在 `NOT_INITIALIZED` 状态调用；
- 其他 request 只允许在 `INITIALIZED` 状态调用。

不满足条件时，core 返回 JSON-RPC `-32600 Session not initialized`。

### 4.6 initialize 的数据变化

收到 `initialize` request 后：

1. core 已经为 reply target 找到或创建 session。
2. 若 params 中带 `mcp_peer_identity` 且 discovery 已启用，discovery 记录 peer identity。
3. 若 identity 表示 `data_channel`，session 还会标记为数据通道。
4. server 构造 initialize result：协议版本、tools/resources/prompts capabilities 和 serverInfo。
5. 结果按原 reply target 返回。
6. session 进入等待 initialized notification 或直接进入 initialized。

普通 MCP 客户端通常不提供 `mcp_peer_identity`；这个字段是本项目节点互联扩展使用的。

### 4.7 notification 如何处理

`core_async_cb()` 按 message type 调用 `handle_notification()`。当前识别两种通知：

- `notifications/initialized`：把等待态 session 推进为 `INITIALIZED`。
- `notifications/cancelled`：根据 `request_id` 或 `requestId` 查找 in-flight request 并取消。

未知 notification 当前被忽略，不返回 method-not-found。这个行为符合 notification 不应得到普通响应的原则，但也意味着未知通知不会留下客户端可见错误。

一个重要的当前实现边界：notification 不经过 `gate_allows_method()`；`handle_notification()` 自己决定是否根据 session 状态采取动作。

### 4.8 request method 分派表

`handle_request()` 当前直接识别：

| method | 行为 |
| --- | --- |
| `ping` | 返回空 object |
| `initialize` | 返回能力并推进 session |
| `tools/list` | 刷新该 session 的工具快照 |
| `tools/call` | 进入第 6 步的调用生命周期 |
| `resources/list` | 返回空 resources 数组 |
| `resources/templates/list` | 返回空 resourceTemplates 数组 |
| `prompts/list` | 返回空 prompts 数组 |
| 其他 method | JSON-RPC `-32601 Method not found` |

capabilities 中目前声明 resources 和 prompts，但 list 结果为空；这表示协议入口存在，不代表项目已经实现资源或提示内容。

### 4.9 取消的当前语义

取消通知的数据生命周期是：

```text
request_id/requestId
  → mcp_jsonrpc_id_to_key()
  → in-flight map 查找
  → cancelled=true
  → 若有 op_ctx/op_free，释放下游异步上下文
  → 向原 request 的 reply target 返回 "Cancelled"
  → 从 in-flight 移除并释放
```

`mcp_jsonrpc_id_to_key()` 给整数和字符串加不同前缀，例如 `i:1` 与 `s:1`，避免数值 ID 1 和字符串 ID "1" 被当作同一个 key。

当前静态实现还有一个应当了解的边界：取消查找使用全局 `id_key`，没有把发出取消通知的 reply target 纳入 key，也没有在取消时核对通知来源；多客户端使用相同 request ID 时可能产生歧义。这是后续增强 Gateway 和多租户安全设计应修正的点。

### 4.10 本步的数据生命周期

```text
payload bytes
  → Jansson root
  → mcp_jsonrpc_message(method/id/params)
  → queue node
  → core_async_cb
  → 找到/创建 session
  → session gate
  → method-specific handler
  → result/error JSON
  → 原 reply target
  → message 和 queue node 释放
```

session 会跨多次消息长期存在；message 只活到本次 core 分派结束；tools/call 若转为异步，则另建 in-flight entry 延长“请求身份与回复路径”的生命周期。

### 4.11 源码与 CodeGraph 证据

- `src/protocol/jsonrpc.c:135-210`：解析、通用校验、request/notification 分类和所有权。
- `src/core/server.c:156-207`：reply target 到 session 的映射和 stream session 删除。
- `src/core/server.c:209-236`：session gate 与 initialize result。
- `src/core/server.c:445-481`：取消处理。
- `src/core/server.c:483-675`：request method 分派。
- `src/core/server.c:677-694`：notification 分派。
- `src/core/server_internal.h:22-35`：session 状态和 session 数据结构。
- CodeGraph 显式调用边确认 `core_async_cb → handle_request/handle_notification`、`handle_request → send_result_to/send_error_to`。

### 4.12 阅读检查

- [ ] 能说明 protocol 层和 method 参数校验层的边界。
- [ ] 能解释 request 与 notification 为什么由 `id` 区分。
- [ ] 能画出三态 session 初始化状态机。
- [ ] 能解释 reply target 与 session 的关系。
- [ ] 能说明未知 request 和未知 notification 的不同处理。
- [ ] 能指出当前取消 key 在多客户端场景下的歧义风险。

### 4.13 阅读确认记录

尚未填写。

---

## 第 5 步：工具注册表、版本与 session snapshot

状态：**未阅读**

### 5.1 registry 保存的不是函数名列表

registry 中每项是完整 `mcp_tool_descriptor`，包括：

- name、description、input schema；
- source、version、risk level、permission；
- idempotent、retryable、cancelable；
- enabled、timeout；
- route、handler、handler_data。

这些字段同时服务三个目的：对客户端描述工具、为 gateway 提供路由元数据、为未来授权/重试/超时策略提供输入。

### 5.2 descriptor 的所有权

注册时 `mcp_tool_registry_register()` 不保留调用者传入结构的裸引用，而是：

1. 复制字符串字段；
2. 对 input schema 增加 JSON 引用计数；
3. 复制布尔、timeout、route 和函数指针；
4. 将副本放入 registry 动态数组。

因此 built-in 注册函数可以在注册后释放临时 schema；插件也可以使用 ABI descriptor 注册而不要求原结构永久存活。

registry 销毁或 unregister 时负责释放这些副本。

### 5.3 唯一名称、enabled 与 version

工具名在 registry 内全局唯一。重复注册会失败，不能静默覆盖。

`version` 初值为 1，以下变化会递增：

- 注册工具；
- 注销工具；
- enabled 状态实际发生改变。

`find()` 只返回 enabled 工具。disabled 工具仍可出现在 internal list 中，但不会出现在 public list，也无法通过普通 gateway 查找执行。

### 5.4 public list 与 internal list

`mcp_tool_registry_public_list()` 生成客户端 `tools/list` 使用的对象：

```json
{
  "tools": [
    {
      "name": "...",
      "description": "...",
      "inputSchema": {},
      "annotations": {
        "source": "...",
        "route": "local_builtin",
        "risk_level": "L0",
        "permission": "system.read",
        "idempotent": true,
        "retryable": true,
        "cancelable": false,
        "timeout_ms": 1000
      }
    }
  ],
  "registryVersion": 1
}
```

internal list 还包含 disabled 工具及其 enabled/version 信息，供管理工具观察；它不是普通客户端 session snapshot 的来源。

### 5.5 built-in、plugin 与远端工具目前如何进入目录

- built-in：`mcp_register_builtin_tools()` 用 route=`LOCAL_BUILTIN` 注册并保存 handler。
- plugin：host API 把 ABI descriptor 转换为 route=`LOCAL_MODULE`，`handler_data` 保存 plugin ID。
- discovery 远端：当前不会把每个远端工具自动注册为 `REMOTE_SERVER` descriptor；主要通过内建 `gateway.proxy_tool` 显式传入 server ID 和远端工具名。
- `REMOTE_SERVER` 枚举已经存在，但普通 gateway route 分支尚未实现。

这也是外部 Gateway 增强的最清晰切入点：让经过命名和校验的远端工具成为真正的 registry descriptor，并实现该 route，而不是继续把所有远端调用藏在一个通用 proxy 参数里。

### 5.6 session snapshot 为什么存在

`tools/list` 不只是读取当前 registry，还把返回对象保存在 `session->tool_snapshot`：

```text
registry 当前 public list
  → 新 JSON snapshot
  → 替换该 session 的旧 snapshot
  → 同一个对象返回客户端
```

之后 `tools/call` 必须先在这个 snapshot 中找到工具名。这保证客户端不能突然调用一个从未在本 session 可见目录中出现的动态工具。

如果 session 尚无 snapshot，当前 `tools/call` 会自动从 registry 创建一次 snapshot。因此源码并不强制客户端必须显式先调用 `tools/list`；但一旦 snapshot 存在，它就成为该 session 的可见性边界。

### 5.7 registry 变化不会自动改写所有 session

registry 是全局实时目录，snapshot 是 session 局部历史视图。两者可以暂时不同：

```text
registry v10 ──tools/list──→ session A snapshot v10
registry 注册新工具，变成 v11
session A snapshot 仍为 v10
session B tools/list 得到 v11
```

session A 要再次 `tools/list` 才能看到一般新增工具。

插件 `insmod/rmmod` 是当前特例：调用完成后，core 主动刷新发起这次变更的 session snapshot，使该 session 立即看到加载/卸载结果。其他 session 不会同步刷新。

### 5.8 为什么 gateway 还要再检查一次 snapshot

core 在进入 gateway 前检查可见性，gateway 内又调用 `snapshot_contains_tool()`。这是纵深校验：即使未来增加其他 gateway 调用入口，gateway 仍不应绕过 session snapshot。

然后 gateway 才查询实时 registry。于是可能出现：

1. snapshot 中存在工具，但实时 registry 已禁用/卸载；gateway 返回“not loaded or disabled”。
2. registry 中有新工具，但旧 snapshot 没有；gateway 拒绝并要求刷新。

这两个判断分别回答“客户端是否见过”和“工具现在是否仍能执行”。

### 5.9 动态目录的一致性语义

当前模型可以总结为：

- registry version 表示全局目录变化；
- snapshot 固定某个 session 最近看见的目录内容；
- 执行时还要检查实时 descriptor；
- registry 变化不会偷偷赋予旧 session 新能力；
- 工具下线优先于旧 snapshot 的历史可见性。

对增强 Gateway 来说，这个语义应保留：远端 catalog 更新可以生成新 registry generation，但旧 session 不应被静默扩权；远端工具紧急下线则必须在执行时拒绝。

### 5.10 本步的数据生命周期

```text
builtin/plugin/未来 upstream descriptor
  → registry 复制并拥有 descriptor
  → version 增长
  → public_list 生成独立 JSON
  → session 持有 snapshot
  → tools/call 先查 snapshot
  → gateway 再查实时 registry
  → session 关闭时释放 snapshot
```

### 5.11 源码与 CodeGraph 证据

- `include/mcp/tools/tool.h:15-45`：route、descriptor、invocation。
- `src/registry/tool_registry.c:21-70`：descriptor 深拷贝和清理。
- `src/registry/tool_registry.c:119-203`：注册、注销、启停和查找。
- `src/registry/tool_registry.c:205-281`：public/internal list 与 registryVersion。
- `src/core/server.c:345-377`：snapshot 查询和刷新。
- `src/core/server.c:538-564`：`tools/list` 与首次 `tools/call` snapshot 行为。
- `src/gateway/gateway.c:49-69,175-195`：gateway 二次可见性及实时 registry 校验。
- `src/plugin/plugin_manager.c:374-437`：plugin descriptor 转 registry descriptor。
- CodeGraph 确认 builtin/plugin manager 都调用 `mcp_tool_registry_register()`，gateway 调用 `mcp_tool_registry_find()`。

### 5.12 阅读检查

- [ ] 能说明 descriptor 中“展示、策略、路由”三类字段。
- [ ] 能解释 registry version 与 session snapshot 的区别。
- [ ] 能解释旧 snapshot 中有工具但实时 registry 已卸载时为何仍会失败。
- [ ] 能说明插件变更后为什么只有当前 session 被主动刷新。
- [ ] 能指出当前远端工具尚未自动成为 `REMOTE_SERVER` descriptor。

### 5.13 阅读确认记录

尚未填写。

---

## 第 6 步：`tools/call`、Gateway 与 in-flight 完整生命周期

状态：**未阅读**

### 6.1 从 params 提取调用

`extract_tool_call()` 要求 params 是 object、`name` 是 string；参数优先读 `arguments`，兼容 `args`，缺失则创建空 object。参数最终必须是 object，否则返回 JSON-RPC `-32602 Invalid params`。

这里的 schema 元数据尚未由一个通用 validator 自动执行。具体 built-in/plugin 仍需自行检查字段；增强 Gateway 应补充统一的 schema/size 策略层。

### 6.2 为什么先创建 in-flight

在调用 gateway 前，core 把 JSON-RPC id 转为 typed key，然后创建 `mcp_in_flight_entry`：

```text
client id
  → i:<integer> 或 s:<string>
  → entry.id_key
  → entry.id（增加引用）
  → entry.reply_to（复制）
  → entry.invocation_id = srv_call_N
```

client ID 用于最终 JSON-RPC 响应；内部 invocation ID 用于插件等执行模块识别一次调用。两者分离，避免把外部 ID 直接当内部执行句柄。

### 6.3 Gateway 的统一输入和四种返回状态

core 调用 `mcp_gateway_call()` 时传入：server、id_key、invocation_id、tool name、arguments、session snapshot，以及 result/error 输出位置。

Gateway 返回值语义：

- `MCP_GATEWAY_OK`：同步成功，result 是 MCP tool result；
- `MCP_GATEWAY_TOOL_ERROR`：工具级失败，仍通过 JSON-RPC result 返回，内容标记 `isError`；
- `MCP_GATEWAY_PROTOCOL_ERROR`：协议级失败，使用 JSON-RPC error；
- `MCP_GATEWAY_PENDING`：下游异步持有调用，core 暂不回复，也不删除 in-flight。

工具执行失败与 JSON-RPC 协议失败是两个层次。模型可理解的工具错误通常仍是一个成功到达客户端的 JSON-RPC response，只是 tool result 表明 `isError=true`。

### 6.4 Gateway 的路由顺序

`mcp_gateway_call()` 当前按以下顺序：

```text
统计 calls_total
  → 再检查 session snapshot
  → 若是 gateway.proxy_tool：走 discovery 代理特例
  → 否则从 registry 查实时 descriptor
  → LOCAL_MODULE：plugin manager
  → LOCAL_BUILTIN：直接 handler
  → 其他 route：返回“已声明但未实现”
```

`gateway.proxy_tool` 是一个内建工具名，但 gateway 在 descriptor lookup 前识别它。它从 arguments 读取 `server_id`、`tool_name`、`args/arguments` 和可选 `proxy_timeout_ms`，然后交给 discovery。

`REMOTE_SERVER` 和 `EMBEDDED_ENDPOINT` 目前虽然在 enum 中存在，却没有普通执行分支。增强方案应实现 `REMOTE_SERVER`，并把 proxy 特例逐步降级为兼容/诊断接口。

### 6.5 本地 built-in 同步路径

Gateway 构造栈上的 `mcp_tool_invocation`：

```text
invocation_id
tool_name
arguments
descriptor
```

然后直接调用 `descriptor->handler(server, &invocation, out_result)`。handler 返回后：

1. gateway 转换为 OK 或 TOOL_ERROR；
2. core 发送 result；
3. core 从 in-flight map 删除 entry；
4. 释放 id、id_key、invocation_id 和 entry。

这是最短的完整调用生命周期。

### 6.6 plugin 同步与异步路径

route=`LOCAL_MODULE` 时，gateway 调用 plugin manager。manager 根据 descriptor 的 `handler_data` 找 plugin ID，确认 plugin ACTIVE 且确实拥有该工具，然后：

1. 序列化 arguments；
2. 增加 plugin `in_flight`；
3. 保存 `invocation_id ↔ id_key` pending mapping；
4. 调用 plugin ABI `invoke()`。

若插件同步返回 OK/ERROR，manager 立即删除 pending mapping 并减少计数；若返回 PENDING，mapping 和 core in-flight 都保留，等待 host API 完成。

插件可以从任意线程调用完成 API。非 loop 线程的结果先进入带 mutex 的 complete queue，再由 `uv_async_t` 回到 event loop；最终通过 `mcp_server_complete_async_ok()` 找到 core entry、按原 reply target 回复并释放。

### 6.7 discovery 远端 pending 路径

`gateway.proxy_tool` 成功创建 discovery pending proxy 后返回 `MCP_GATEWAY_PENDING`。此时同时存在两层状态：

```text
core in-flight
  id_key + client id + reply target

discovery pending proxy
  id_key + remote_id + server_id/peer + tool + args + timer
```

远端响应、超时或连接失败都会生成一个 tool result，并调用 `mcp_server_complete_async_ok()` 收口到原客户端。详细流程见第 8 步。

### 6.8 同步完成与异步完成的汇合点

```text
同步：gateway 返回
  → core send_result/send_error
  → remove in-flight

异步：gateway 返回 PENDING
  → 下游稍后 mcp_server_complete_async_ok(id_key)
  → 按 entry.reply_to 发送
  → remove in-flight
  → maybe_shutdown
```

`maybe_shutdown()` 说明 in-flight 不只是调用记录，也参与优雅关闭：只要还有异步请求，普通 shutdown 不会立即关闭所有入口。

### 6.9 当前 timeout、retry 和 cancel 的实际边界

descriptor 有 timeout/retryable/cancelable 元数据，但当前 gateway 没有通用 timer、重试器或取消分派器：

- 远端 proxy 自己创建 timer；
- 文件传输插件维护自己的 deadline/timer；
- 普通 built-in 没有统一 timeout wrapper；
- plugin descriptor 的 cancelable 没有自动转成 ABI cancel 回调；
- gateway 本身不执行 retry。

因此不能把 descriptor 元数据误读为“策略已实现”。它们目前主要是声明和展示信息，只有具体下游代码显式消费时才产生运行行为。

### 6.10 ID 与多 session 风险

in-flight map 当前只以 `id_key` 查找，没有把 session/reply target 纳入复合键；`mcp_in_flight_put()` 也没有拒绝重复 key。对单 stdio 客户端通常无冲突，但多 TCP/UDP 客户端可能都使用整数 ID 1。

这会影响完成、取消和远端 remote ID 映射。增强 Gateway 前应优先把内部 request key 改为服务器生成的唯一 token，或使用 `(session_id, typed_client_id)` 复合身份；客户端 id 只用于回包。

### 6.11 计数与可观测性

Gateway 维护饱和计数：总调用、本地调用、远端调用、拒绝调用。`gateway.status` 工具还会附带四种入口是否启用。

这些计数适合基本状态观察，但尚不能回答每个工具/upstream 的延迟、超时、排队、错误类别和并发。这些是增强 Gateway 的后续指标。

### 6.12 本步完整数据生命周期

```text
tools/call params
  → name + arguments
  → snapshot 可见性
  → typed id_key
  → core in-flight + invocation_id
  → gateway snapshot 再校验
  → registry descriptor / proxy 特例
  → builtin 或 plugin 或 discovery
  → 同步 result/error 或 pending
  → 原 reply target
  → 删除所有下游 mapping
  → 删除 core in-flight
```

### 6.13 源码与 CodeGraph 证据

- `src/core/server.c:410-443`：tools/call 参数提取。
- `src/core/server.c:547-650`：snapshot、in-flight、gateway 和完成分支。
- `src/core/in_flight.c:40-115`：entry 创建、查找与删除。
- `src/gateway/gateway.c:71-136`：proxy tool 参数及 pending 路径。
- `src/gateway/gateway.c:157-239`：统一 gateway 路由。
- `src/plugin/plugin_manager.c:1162-1220`：plugin 调用同步/异步判定。
- `src/plugin/plugin_manager.c:454-625`：异步完成回到 loop 并收口。
- `src/core/server.c:881-893`：异步结果最终返回客户端。
- CodeGraph 确认 `handle_request → mcp_gateway_call`、`mcp_gateway_call → mcp_plugin_manager_invoke`、built-in handler 和 discovery 代理调用关系。

### 6.14 阅读检查

- [ ] 能区分 client request ID、id_key 和 invocation ID。
- [ ] 能解释 tool error 与 JSON-RPC protocol error 的差异。
- [ ] 能画出 built-in 同步路径和 plugin/discovery 异步路径。
- [ ] 能说明 `MCP_GATEWAY_PENDING` 为什么不能立即删除 in-flight。
- [ ] 能指出 descriptor timeout/retry/cancel 元数据目前没有统一执行器。
- [ ] 能说明多 session 重复 request ID 的静态风险。

### 6.15 阅读确认记录

尚未填写。

---

## 第 7 步：插件生命周期与 MFT1 文件传输

状态：**未阅读**

### 7.1 为什么需要 plugin manager

registry 和 gateway 只需要理解统一 descriptor；plugin manager 负责把稳定的主程序世界与插件 ABI 连接起来。它解决四类问题：

1. 内建插件与动态库插件的统一生命周期；
2. 插件工具注册/注销到全局 registry；
3. 同步/异步调用结果回到 core；
4. 插件自定义 peer frame 与 capability 的注册。

插件不直接操作客户端 connection 或 reply target。它只使用 invocation ID 和 host API；最终响应归属仍由 core in-flight 决定。

### 7.2 plugin module 的长期状态

每个 `plugin_module` 保存：plugin ID、路径、状态、动态库句柄、init/invoke/shutdown 函数、host API、工具列表、frame handler、pending calls 和 in-flight 计数。

状态集合为：

```text
LOADING → ACTIVE → DRAINING → UNLOADING → UNLOADED
    └──────────────────────────────→ FAILED
```

动态卸载只接受 ACTIVE 的非内建插件，且 `in_flight` 必须为 0。这样不会在插件代码仍被调用时关闭动态库。

### 7.3 内建插件和动态插件

- 内建插件：server 初始化时用静态函数指针注册，文件传输在构建模式 `y` 时采用此方式。
- 动态插件：`plugin_tools.insmod` 使用 `uv_dlopen()`，查找 `mcp_plugin_init`、`mcp_plugin_invoke`、`mcp_plugin_shutdown` 三个 ABI 符号。
- 构建模式 `m`：只生成 `mcp_file_transfer_plugin` 动态模块，需要运行时 insmod。
- 构建模式 `n`：不把文件传输插件加入 server，也不生成模块。

无论装载方式如何，之后都走同一个 `init_linked_plugin()`、host API 和 invoke 路径。

### 7.4 host API 的边界

ABI 1.1 提供：日志、权限查询、时间、event loop、工具注册/注销、异步完成、peer frame 发送/handler 注册、peer capability 查询/设置。

插件注册工具时，manager：

1. 解析插件给出的 schema JSON；
2. 补齐默认 source/risk/permission/timeout；
3. 设置 route=`LOCAL_MODULE`；
4. 把 plugin ID 放入 `handler_data`；
5. 注册到 registry；
6. 在 plugin 自己的工具链表中记录名称。

第 4、5 步的全局 registry 与 snapshot 语义因此自然适用于插件工具。

### 7.5 插件异步完成为什么需要两层 mapping

plugin manager 保存 `invocation_id ↔ core id_key`；core in-flight 保存 `id_key ↔ client id/reply target`：

```text
plugin invocation_id
  → plugin_pending_call.id_key
  → core in-flight
  → client JSON-RPC id + reply target
```

插件只需回传 invocation ID。manager 取走 pending call 后调用 `mcp_server_complete_async_ok()`，core 再找到原客户端。

如果插件从工作线程完成，manager 复制 payload/message 到 thread-safe complete queue，用 mutex 保护，再以 `uv_async_send()` 让 loop 线程统一修改 plugin/core 状态。

### 7.6 unload 的资源顺序

`unload_plugin()` 的顺序是：

```text
plugin shutdown
  → state=UNLOADING
  → 先把所属工具 disabled
  → 从 registry 注销工具
  → 从 peer transport 注销 owner/handler
  → 释放 frame adapter 和 pending calls
  → uv_dlclose 动态库
  → state=UNLOADED
```

先 disable 再 unregister 可缩短并发可见窗口；真正 rmmod 前还会拒绝存在 in-flight 的插件。

### 7.7 文件传输插件在系统中的位置

MFT1 插件注册两个工具：

- `server.send`：把本地文件/目录发送给已发现节点；
- `server.recv`：从远端节点接收文件/目录。

它们仍通过普通 `tools/call → gateway → plugin manager` 启动，但大块数据不走 JSON-RPC result。插件向 peer transport 注册 magic `MFT1` 的二进制 frame handler，控制信息与数据块在 framed peer 连接上交换，最终只用 host async completion 完成原始 tools/call。

### 7.8 MFT1 能力协商

插件定义的能力包括 whole-file 校验、resume、block ACK、chunk window、CRC32 和 data channel。调用 `server.send/recv` 时：

1. 校验 server_id、local_path、remote_path、timeout；
2. 查询 peer 是否已有必要 capability；
3. 若没有，发送 HELLO 并保存 pending negotiation；
4. 返回 `MCP_PLUGIN_CALL_PENDING`；
5. 收到 HELLO 后记录能力并恢复等待调用；
6. 协商超时则通过 host API 完成错误。

能力按 server ID 保存在 peer transport；peer 关闭时会清除，避免重连后沿用过期能力。

### 7.9 一次文件发送的简化生命周期

```text
server.send
  → 构建文件/目录 manifest
  → 计算文件、逻辑块校验信息
  → MANIFEST/协商接收决策
  → 接收端返回 accept + resume_offset
  → sender 按块和 chunk 发送 DATA
  → receiver 异步落盘并校验
  → BLOCK ACK / NACK + WINDOW_UPDATE
  → 必要时有限重传
  → COMPLETE
  → receiver 做 whole-file SHA-256 校验
  → host.complete_async_ok/error
  → 原 tools/call 完成
```

插件有 session/stream window、最大并行 block、写队列和 high-watermark 控制，避免无限制把文件内容压入 event loop 写队列。

### 7.10 接收、临时文件和断点续传

接收端通常写入临时 `.part` 文件。resume offset 必须通过现有内容和逻辑 block 边界校验，不能只相信对端声明。已验证前缀可跳过；未验证或不对齐内容必须重新接收。

块级 CRC/ACK 保证传输阶段发现损坏，最终 whole-file SHA-256 再验证整体内容。只有最终校验通过才把调用标记成功。

文件/目录路径是高风险输入。当前测试覆盖 path 场景，但在部署前仍需结合允许根目录、符号链接策略、覆盖策略和权限模型审计；不能只依赖 JSON schema 中“这是 string”。

### 7.11 timer、I/O work 与 shutdown

插件维护 transfer deadline、block ACK deadline、receive deadline 和 capability negotiation deadline。文件准备/写入可使用 `uv_queue_work()`，避免大文件 I/O 全部堵塞 loop。

shutdown 会先检查仍在执行的 I/O job；有 job 时返回 busy，manager 因此拒绝立即卸载。无 job 后停止 timer thread、清空 negotiation/transfer 状态、注销 MFT1 handler 并销毁同步原语。

### 7.12 本步的数据生命周期

```text
tools/call
  → core in-flight
  → plugin pending call
  → transfer context
  → manifest + per-file/per-block state
  → peer MFT1 frames
  → work queue I/O + ACK/window/deadline
  → whole-file verification
  → plugin async completion
  → core response
  → transfer/pending/in-flight 逐层释放
```

### 7.13 源码与测试证据

- `include/mcp/plugin/plugin_abi.h:13-92`：ABI、descriptor、host API 与入口函数。
- `src/plugin/plugin_manager.c:18-89`：plugin 状态与内部对象。
- `src/plugin/plugin_manager.c:374-437`：工具注册/注销桥接。
- `src/plugin/plugin_manager.c:454-625`：跨线程异步完成。
- `src/plugin/plugin_manager.c:877-938`：卸载与销毁。
- `src/plugin/plugin_manager.c:954-1132`：内建/动态加载及 rmmod。
- `src/plugin/plugin_manager.c:1162-1220`：invoke 路径。
- `src/CMakeLists.txt:3-8,31,55-56,72-100`：文件传输 y/m/n 构建模式。
- `src/plugins/file_transfer/file_transfer_plugin.c:161-181`：协议能力和边界常量。
- `src/plugins/file_transfer/file_transfer_plugin.c:5491-5664`：工具注册、初始化和 invoke。
- `src/plugins/file_transfer/file_transfer_plugin.c:5667-5709`：插件 shutdown。
- `tests/file_transfer_*`：deadline、协商、offset、resume、协议边界、写队列和 smoke 覆盖。本文未运行这些测试。

CodeGraph 可确认 host 注册进入 registry、异步完成进入 core；ABI function pointer 和宏改名的调用边可能不完整，因此文件传输内部链路以直接源码和测试名交叉核对。

### 7.14 阅读检查

- [ ] 能说明 plugin manager 为什么不保存客户端 reply target。
- [ ] 能解释 plugin pending mapping 与 core in-flight 的两层关系。
- [ ] 能说明 `y/m/n` 三种构建模式的区别。
- [ ] 能解释 MFT1 为什么用 peer 二进制帧而不是把文件塞进 JSON-RPC result。
- [ ] 能说明 resume、block ACK、window 和 whole-file hash 分别解决什么问题。
- [ ] 能解释为什么有 I/O job 时插件不能安全卸载。

### 7.15 阅读确认记录

尚未填写。

---

## 第 8 步：服务发现、peer transport 与远端工具代理

状态：**未阅读**

### 8.1 三个容易混淆的角色

- discovery：发现节点、维护 peer 健康、建立 MCP 控制连接、缓存 tools/list、管理 proxy pending。
- peer transport：提供按 server ID 发送 frame、自定义 magic handler、capability 和连接事件抽象。
- gateway：决定本次 tools/call 是否走远端，并把请求交给 discovery。

当前 discovery 自己拥有外连 TCP connection，并通过 `set_external_sender()` 把发送能力接到 peer transport；插件因此只面向 peer transport，不必知道连接是 discovery 建立的。

### 8.2 节点发现数据面

discovery 启动后创建 UDP socket、announce timer 和 heartbeat timer。它向以下目标发送 announce/heartbeat：

- network policy 中配置的 peers；
-显式 hosts；
-允许时的广播地址。

收到合法 discovery packet 后，根据 instance/地址/port 更新 peer 记录。peer 拥有 server ID、在线状态、最近 heartbeat、TCP connection、tools cache 和 generation。

server ID 是本进程运行期分配的内部标识，不应当当作跨重启永久身份。

### 8.3 健康状态与连接

heartbeat timer 周期检查 peer 活性。超时会标记 peer timeout、关闭连接、使 tools cache 失效，并完成该 peer 的 pending proxy 错误。

建立 TCP 后，discovery 使用与第 3 步相同的 4 字节大端 framing。连接成功不等于可代理：还要完成 MCP initialize。

```text
TCP connected
  → send initialize(clientInfo=mcp_gateway + peer identity)
  → receive initialize result
  → send notifications/initialized
  → conn.mcp_initialized=true
  → 通知 peer transport connected
  → 处理等待中的 proxy
```

### 8.4 远端 tools/list 缓存状态机

每个 peer 的 tools cache 有 UNKNOWN、REFRESHING、READY 状态，并有 generation 防止旧响应覆盖新状态。

```text
UNKNOWN
  → 创建 tools_refresh + 唯一 remote ID + timer
  → REFRESHING
  → 收到匹配 ID 且 generation 未过期的 tools/list
  → deep copy tools list
  → READY
```

连接关闭、刷新失败或显式要求刷新时，cache 被释放、generation 增长并回到 UNKNOWN。等待调用不会绕过 cache；普通远端工具必须确实出现在该 peer 最近的 tools/list 中。

### 8.5 一次 `gateway.proxy_tool` 的生命周期

客户端 arguments 包含 server ID、远端 tool name、args 和可选 timeout。Gateway 校验后调用 discovery：

```text
本地 id_key
  → 查找在线 peer
  → 创建 pending_proxy
     - 本地 id_key
     - remote_id = "gateway:" + id_key
     - peer/tool/arguments
     - timeout timer
  → 确保 MCP initialize
  → 确保 tools cache READY
  → 确认远端公开该工具
  → 发送远端 tools/call
```

`tools_list` 作为特殊 tool name 时会代理/返回远端 tools/list，而普通调用会生成标准 `tools/call` params。

### 8.6 远端响应如何回到原客户端

discovery 从 frame 中解析 JSON object，按 remote response ID 查 pending proxy：

- 有 `result`：普通调用直接用远端 result 完成本地调用；
- 有 `error`：转换为本地 `isError=true` 的 tool result，避免把远端内部错误结构直接当本地协议错误；
- ID 不匹配：不消费为该 proxy 响应；
- 完成后更新 peer 活性。

`pending_proxy_finish_result()` 的关键动作：

```text
从 discovery pending 链表摘除
  → mcp_server_complete_async_ok(id_key, result)
  → core 按原 client reply target 回复
  → 关闭 timer
  → 释放 proxy
```

### 8.7 超时、断线、迟到响应

每个 proxy 有一次性 timer。超时会产生 `gateway_proxy_timed_out` tool error 并完成本地 in-flight。

peer 断开或 discovery 关闭时，属于该 peer 的 pending proxy 全部以明确错误完成。timer 关闭是异步的，discovery 用 `pending_proxy_closes` 计数延迟自身释放。

proxy 从 pending 链表移除后，迟到响应找不到 remote ID，不会完成已经释放或之后创建的 request。这是防止迟到响应误配的基本机制。

### 8.8 peer transport 的二进制扩展路径

peer payload 以 `{` 开头时走 JSON callback；否则至少要有 4 字节 magic，并查已注册 handler。未知/过短二进制 payload 会关闭连接。

handler 以 owner 注册，插件卸载时可以按 owner 批量注销。capability 也按 server ID 保存，peer close 时清除。

peer transport 还提供跨线程发送队列：非 loop 线程可通过 mutex/condition 把发送请求交给 `uv_async_t`，最终在 loop 上写连接。这正是文件传输 worker 与网络 I/O 解耦所需的桥。

### 8.9 背压与当前限制

discovery connection 跟踪 `write_queue_bytes`，超过 4 MiB high watermark 时发送返回特殊失败，避免无界积压。peer transport 也有自己的 configurable write high watermark。

但当前远端代理仍有这些限制：

- 只支持本项目 framed TCP peer 方式，不是通用 Streamable HTTP upstream adapter；
- 工具不自动聚合进本地 registry；
- remote ID 由本地 id_key 拼接，多 session 重复 ID 风险仍向远端映射扩散；
- 没有每 upstream 并发舱壁、rate limit、熔断和授权 pipeline；
- 客户端取消没有完整传播为远端取消；
- 没有通用幂等重试。

这些限制定义了增强 Gateway 的开发边界，而不是 discovery 自身“完全不可用”。它已经提供了可复用的连接、初始化、tools cache、timeout 和完成机制。

### 8.10 本步的数据生命周期

```text
UDP discovery packet
  → peer record + server ID
  → TCP framed connection
  → MCP initialize
  → remote tools/list cache + generation
  → pending proxy + remote ID + timer
  → remote tools/call
  → remote result/error
  → local core in-flight
  → original client reply target
  → proxy/timer/cache 按事件释放或失效
```

### 8.11 源码与 CodeGraph 证据

- `src/discovery/server_discovery.c:22-31`：周期、timeout 和 high watermark 常量。
- `src/discovery/server_discovery.c:37-175`：peer/cache/pending/discovery 状态。
- `src/discovery/server_discovery.c:988-1093`：连接和 framed 写队列。
- `src/discovery/server_discovery.c:1129-1211`：pending proxy 生命周期和 timeout。
- `src/discovery/server_discovery.c:1213-1541`：tools cache、generation 与刷新。
- `src/discovery/server_discovery.c:1544-1758`：MCP initialize、pending 发送和响应完成。
- `src/discovery/server_discovery.c:2304-2517`：创建、启动、关闭和资源释放。
- `src/discovery/server_discovery.c:2550-2622`：远端调用入口。
- `src/transport/peer_transport.c:209-276`：peer close 时 capability/handler 通知。
- `src/transport/peer_transport.c:344-395`：JSON 与 magic frame 分流。
- CodeGraph 确认 `gateway_proxy_tool_call → mcp_server_discovery_call_remote_tool`、`pending_proxy_finish_result → mcp_server_complete_async_ok`。

### 8.12 阅读检查

- [ ] 能区分 discovery、peer transport 和 gateway 的责任。
- [ ] 能解释“TCP 已连接”和“MCP 已初始化”为什么不是同一状态。
- [ ] 能画出 tools cache 的 UNKNOWN/REFRESHING/READY 流程。
- [ ] 能说明本地 id_key 与 remote ID 如何关联。
- [ ] 能解释 timeout/断线后为何迟到响应不会再次完成调用。
- [ ] 能列出当前远端代理与完整外部 Gateway 的主要差距。

### 8.13 阅读确认记录

尚未填写。

---

## 第 9 步：关闭、错误传播与资源清理

状态：**未阅读**

### 9.1 关闭不是一次 `free(server)`

libuv handle 的关闭回调通常在之后的 event loop tick 执行。如果先释放拥有 handle 的对象，会产生 use-after-free。因此项目把关闭分为：

1. 标记不再接受新工作；
2. 停止输入、timer、连接和后台任务；
3. 让 loop 驱动 close callback；
4. 最后释放长期对象内存。

### 9.2 关闭触发源

- SIGINT/SIGTERM（Windows 另有 SIGBREAK）：signal callback 关闭 signal handles 并请求 server shutdown。
- stdio EOF：根据配置关闭整个 server，或只结束 stdio session/output。
- 启动失败：直接进入 destroy 的强制清理路径。
- `mcp_server_destroy()`：最终兜底，关闭所有 runtime handle 并释放对象图。

### 9.3 正常 request-shutdown 的 drain 条件

`mcp_server_request_shutdown()` 只设置 `shutting_down=true` 并唤醒 core。入口 callback 看到该标志后不再排入新消息。

`maybe_shutdown()` 要等待：

```text
queue_head == NULL
且 in_flight.size == 0
```

满足后才关闭 discovery、UDP、pipe/TCP listener、stdio output 和 core async。这样已经排队或 pending 的请求有机会完成。

### 9.4 discovery 关闭如何处理 pending

discovery 关闭时：

1. 尝试向 peer 发送 offline；
2. 关闭 data connection；
3. 把所有 pending proxy 完成为“discovery is closing”错误；
4. 关闭 pending list timer；
5. 停止 announce/heartbeat timer；
6. 停止 UDP 接收；若还有 UDP send，则等 send callback 后关闭；
7. 等所有 connection/timer close callback 完成后才释放 discovery。

因此 discovery 不会简单丢弃本地 core in-flight，否则 server 的 drain 条件永远无法归零。

### 9.5 强制 destroy 路径

`close_runtime_handles()` 的范围更广：关闭 discovery、所有 ingress、stdio 输入输出、shell jobs、plugin async handle、peer transport 和 core async，然后反复运行 loop 直到不再 alive。

随后 `mcp_server_destroy()`：

1. 清空仍在队列的 message；
2. 清理 stdio/peer sessions 与 snapshot；
3. 再次确保 shell jobs shutdown；
4. 销毁 gateway、plugin manager、discovery、shell store；
5. 销毁 sandbox/policy、peer transport、registry、in-flight；
6. 销毁 UDP/framed/stdio transport 和 network policy；
7. 释放 server。

初始化失败也调用同一函数。由于 server 是 `calloc` 创建且 destroy 函数接受空成员，这条路径兼容半初始化对象。

### 9.6 shell job 与 plugin 的停止语义

shell job store shutdown 会标记 shutting_down，对未回收进程请求 kill，更新 job 状态，轮询一次并关闭 poll timer。

plugin manager close 会禁止新的异步完成入队并关闭 complete async；destroy 再卸载所有插件。文件传输插件如仍有 I/O job，单独 rmmod 会返回 busy；server 强制销毁路径仍需依赖插件 shutdown/loop 清理逻辑正确收敛。

### 9.7 错误传播的四个层次

| 层次 | 例子 | 对客户端表现 |
| --- | --- | --- |
| framing/transport | frame 过大、TCP read 失败 | 关闭连接或停止入口，可能无 JSON 响应 |
| JSON-RPC protocol | parse error、invalid request、method not found | JSON-RPC error object |
| MCP/tool | 工具不可见、插件失败、远端超时 | JSON-RPC result 中 `isError=true` |
| process/lifecycle | 初始化失败、资源关闭失败 | stderr/进程退出；不一定有客户端响应 |

理解错误属于哪层，才能判断应构造 JSON-RPC error、tool result，还是关闭 transport。

### 9.8 当前关闭路径的静态风险边界

以下是源码审阅发现、需要运行测试进一步证明的边界：

- 正常 `maybe_shutdown()` 调用 `mcp_stdio_transport_close_output()`，没有像强制 destroy 一样显式关闭 stdin handle。若 signal 触发时 stdin 仍保持打开，该 handle 是否会使 loop 继续存活需要专门运行验证；不能仅凭现有静态代码宣称 signal shutdown 一定退出。
- 普通 shutdown 无全局 drain deadline。若某个 plugin/in-flight 永不完成，`in_flight.size` 不归零，可能长期等待。
- core cancel 只清理 `entry->op_ctx`；当前 plugin/discovery pending 是否都绑定到 op_ctx 并传播下游取消并不统一。
- UDP error callback 当前不记录或升级错误，存在可观测性空洞。

增强 Gateway 应加入 server 级 drain deadline、每 upstream cancel/drain、pending 完成保证和关闭状态指标。

### 9.9 本步的数据生命周期

```text
shutdown trigger
  → shutting_down=true
  → 拒绝新输入
  → drain core queue
  → 等待或完成 in-flight
  → 停止 discovery/listener/timer/job/plugin/peer
  → loop 驱动 close callbacks
  → 清空 message/session/snapshot/pending
  → 销毁依赖对象
  → 销毁 transport/policy
  → free server
  → main 关闭 loop
```

### 9.10 源码与 CodeGraph 证据

- `src/main/main.c:69-175`：signal handle 生命周期。
- `src/main/main.c:338-354`：event loop 退出后的最终清理。
- `src/core/server.c:290-343`：request shutdown、drain 条件和强制关闭 handles。
- `src/core/server.c:965-1006`：server 对象图销毁顺序。
- `src/discovery/server_discovery.c:2459-2517`：discovery offline、pending、timer 和 UDP 关闭。
- `src/tools/shell_exec.c:3649-3689`：shell jobs shutdown/destroy。
- `src/plugin/plugin_manager.c:837-938`：plugin manager close/destroy。
- `src/transport/*`、`src/listener/framed_listener.c`：各 handle 的 close callback 和延迟释放。
- CodeGraph 确认 `close_runtime_handles` 与各 close 函数、`mcp_server_destroy` 与各 destroy 函数的显式关系。

### 9.11 阅读检查

- [ ] 能解释 libuv close callback 为什么要求“先关活动、后 free”。
- [ ] 能说出正常 shutdown 等待的两个 core 条件。
- [ ] 能说明 discovery 为什么必须完成 pending proxy 而不是直接丢弃。
- [ ] 能区分正常 drain 与 destroy 强制清理路径。
- [ ] 能按四个层次判断错误应如何对外呈现。
- [ ] 能指出正常关闭当前需要运行验证的边界。

### 9.12 阅读确认记录

尚未填写。

---

## 第 10 步：完整端到端串联、模块定位与扩展入口

状态：**未阅读**

### 10.1 场景一：stdio 调用本地 built-in

```text
客户端写入一行 tools/call JSON
  → stdio transport 按换行切分
  → protocol 解析 request
  → reply target=STDIO，message 入 core 队列
  → session gate
  → snapshot 可见性
  → core 创建 in-flight + invocation ID
  → gateway 查 registry descriptor
  → LOCAL_BUILTIN handler 同步执行
  → tool result
  → stdout 写队列输出带换行 JSON
  → 删除 in-flight/message/node
```

涉及模块：main → stdio transport → protocol → core → registry → gateway → tools → core → stdio transport。

### 10.2 场景二：TCP 调用异步 plugin

```text
TCP 4-byte framed tools/call
  → framed listener 按 connection 拆帧
  → reply target=STREAM connection
  → protocol/core/session/snapshot
  → core in-flight
  → gateway route=LOCAL_MODULE
  → plugin pending mapping
  → plugin 返回 PENDING
  → 工作线程完成并进入 plugin complete queue
  → uv_async 回到 loop
  → invocation ID 找 id_key
  → id_key 找 client ID + connection
  → framed response 写回原 TCP connection
  → 两层 pending/in-flight 清理
```

这里最值得记忆的是“业务异步完成”和“客户端连接归属”由两层 mapping 解耦。

### 10.3 场景三：客户端通过 Gateway 调远端工具

```text
客户端调用 gateway.proxy_tool
  → 本地 session/snapshot/in-flight
  → gateway 校验 server_id/tool/args/timeout
  → discovery 查在线 peer
  → 必要时完成 TCP + MCP initialize
  → 必要时刷新远端 tools/list cache
  → 确认远端工具已公开
  → 分配 remote ID 并发送远端 tools/call
  → 远端 response
  → discovery remote ID 找本地 id_key
  → core id_key 找原 reply target
  → 返回本地客户端
```

当前它是“显式 proxy 工具”，不是把远端工具透明聚合为本地工具名。增强方案将在不破坏这条链路的前提下加入 `REMOTE_SERVER` descriptor。

### 10.4 场景四：跨节点文件发送

```text
客户端 tools/call server.send
  → gateway/plugin manager
  → MFT1 capability negotiation
  → manifest + resume decision
  → peer transport 二进制 DATA
  → 接收端 work queue 落盘
  → ACK/window/retry
  → SHA-256 最终校验
  → plugin host async complete
  → core 返回原客户端
```

JSON-RPC 负责启动和最终结果，MFT1 数据面负责大文件。这是“控制面与数据面分离”的具体实例。

### 10.5 按问题寻找模块

| 想回答的问题 | 首先阅读 |
| --- | --- |
| 服务为什么启动了某个入口 | `src/main/main.c` |
| 字节怎样变成一条消息 | `src/transport/*`, `src/listener/framed_listener.c` |
| JSON-RPC 为什么被拒绝 | `src/protocol/jsonrpc.c` |
| 当前客户端是否已初始化 | `src/core/server.c`, `server_internal.h` |
| 工具为何看不见 | session snapshot + `tool_registry.c` |
| 工具为何走某个执行路径 | `src/gateway/gateway.c` |
| built-in 做了什么 | `src/tools/*` |
| plugin 如何注册和完成 | `src/plugin/plugin_manager.c`, ABI header |
| 远端节点为何不可用 | `src/discovery/server_discovery.c` |
| 二进制扩展帧到哪里 | `src/transport/peer_transport.c` |
| 文件传输为何暂停/重试 | `file_transfer_plugin.c` |
| 进程为何不能退出 | core in-flight、各 handle close callback、shell/plugin/discovery pending |

### 10.6 各类状态的最终归属

| 状态 | 所有者 | 生命周期 |
| --- | --- | --- |
| 启动配置 | main/server | 进程级 |
| event loop | main | 进程级 |
| registry descriptor | registry | 注册到注销/进程结束 |
| session state/snapshot | core session | 输入通道/peer session 级 |
| message node | core queue | 单次排队处理 |
| in-flight entry | core | 一次未完成调用 |
| plugin pending call | plugin manager | plugin 异步调用 |
| discovery pending proxy | discovery | 一次远端代理 |
| peer capability | peer transport | peer 连接期 |
| file transfer context | MFT1 plugin | 一次传输 |
| shell job | shell job store | job + retention 时间 |

### 10.7 当前已经实现与尚未实现

已经从源码确认存在：

- 四类 ingress 与统一 core 队列；
- JSON-RPC request/notification 和 session 初始化门控；
- registry、session snapshot、built-in/plugin route；
- plugin ABI、动态装卸和异步完成；
- discovery、peer 初始化、tools cache、显式远端 proxy；
- MFT1 文件传输的协商、resume、ACK/window、校验和 deadline；
- 分层 close/destroy 路径。

尚不能从当前源码声称已经完成：

- 通用外部 MCP upstream 配置和标准 transport adapter；
- 自动聚合远端工具为 `REMOTE_SERVER` descriptor；
- 多租户身份/授权和每次执行重新鉴权；
- 通用 schema validator、deadline、取消传播、重试、限流、熔断；
- 完整 metrics/trace/audit；
- 多 session request ID 隔离；
- 本轮文档对应的运行时、网络或构建验证。

### 10.8 推荐离线阅读顺序

1. 先读第 1～3 步，建立模块地图和输入边界。
2. 连读第 4～6 步，完整掌握普通 tools/call。
3. 根据需求选择第 7 步文件传输或第 8 步远端代理。
4. 读第 9 步理解对象为何按当前顺序销毁。
5. 回到本步四个场景，不看前文尝试自己复述。
6. 最后阅读 `docs/external_mcp_gateway_plan.md`，区分“现有事实”和“建议设计”。

### 10.9 CodeGraph 的正确使用方式

CodeGraph 适合确认：模块间显式调用、定义位置、文件依赖和高层主链。它对 callback、函数指针、ABI 符号、宏改名和运行时动态库绑定可能缺边或误配同名符号。

因此本项目的可靠阅读方法是：

```text
module_topology 提供地图
  + CodeGraph 找显式调用边
  + 直接源码补 callback/状态/所有权
  + tests 识别设计契约
  + 真正运行测试才形成运行时证据
```

本文件目前停留在前四类静态证据，没有执行最后一步。

### 10.10 外部 Gateway 后续开发入口

详细方案见 `docs/external_mcp_gateway_plan.md`。推荐从最小闭环开始：

```text
静态 upstream 配置
  → 复用现有 peer/discovery adapter
  → upstream manager
  → namespaced REMOTE_SERVER descriptor
  → gateway 普通 route 分支
  → unique internal request token + deadline
  → fake upstream 集成测试
```

先把 catalog 和一次远端调用生命周期做成稳定接口，再添加 Streamable HTTP、认证、动态配置、熔断和负载均衡。这样复杂功能是在确定的扩展点上叠加，而不是继续扩大 `server.c`/`gateway.c` 的条件分支。

### 10.11 全文理解检查

- [ ] 能从任一 transport 追踪到 response 返回。
- [ ] 能区分 process、session、request、plugin call、remote proxy 和 transfer 生命周期。
- [ ] 能说明 registry、snapshot、gateway、in-flight 各自回答什么问题。
- [ ] 能解释本地同步、插件异步、远端代理三种完成路径在哪里汇合。
- [ ] 能说明 discovery 与 peer transport 的边界。
- [ ] 能指出当前实现事实与 Gateway 增强建议的边界。
- [ ] 能基于模块定位表快速找到某类问题的首要源码。

### 10.12 阅读确认记录

尚未填写。
