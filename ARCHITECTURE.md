# libzmq 项目架构分析报告

## 1. 项目概览

**libzmq** 是 ZeroMQ 的核心 C++ 实现（版本 4.3.3），是一个高性能异步消息库，扩展了标准 socket 接口，提供异步消息队列、多种消息模式、消息过滤（订阅）、无缝多传输协议支持等功能。

- **语言**: 主要使用 C++98 编写，部分可选 C++11 特性
- **许可证**: LGPL v3+（含特殊链接例外）
- **代码规模**: `src/` 目录约 56,700 行代码（.cpp/.hpp/.c/.h）

---

## 2. 顶层目录结构

```
libzmq/
├── include/          # 公共 API 头文件 (zmq.h, zmq_utils.h)
├── src/              # 核心源码 (~160+ 文件)
├── tests/            # 集成/功能测试 (~126 个测试文件)
├── unittests/        # 单元测试 (8 个测试文件)
├── doc/              # API 文档 (76 个 asciidoc 格式文件)
├── builds/           # 多平台构建配置
│   ├── cmake/        # CMake 模块
│   ├── android/      # Android NDK 构建
│   ├── mingw32/      # MinGW 构建
│   ├── cygwin/       # Cygwin 构建
│   ├── vxworks/      # VxWorks 构建
│   └── ...           # 其他平台
├── external/         # 第三方依赖
│   ├── sha1/         # 内建 SHA1 实现
│   ├── unity/        # Unity 测试框架
│   └── wepoll/       # Windows epoll 模拟
├── perf/             # 性能基准测试工具
├── tools/            # 辅助工具 (curve_keygen 等)
├── CMakeLists.txt    # CMake 构建主文件
├── configure.ac      # Autotools 构建配置
├── Makefile.am       # Automake 配置
└── Dockerfile        # Docker 构建支持
```

---

## 3. 构建系统

libzmq 支持两套构建系统：

| 构建系统 | 入口文件 | 适用场景 |
|---------|---------|---------|
| **CMake** | `CMakeLists.txt` | 跨平台，Windows/macOS/Linux |
| **Autotools** | `configure.ac` + `Makefile.am` | Unix/Linux 系统 |

### 关键构建选项

- `ENABLE_DRAFTS` — 构建 Draft API（开发版特性）
- `ENABLE_WS` — WebSocket 传输支持
- `ENABLE_CURVE` — CURVE 加密支持
- `WITH_LIBSODIUM` — 使用 libsodium（默认）或 tweetnacl
- `WITH_TLS` — WSS (WebSocket over TLS) 支持，依赖 GnuTLS
- `WITH_OPENPGM` — OpenPGM 多播支持
- `ENABLE_ASAN` — Address Sanitizer 支持
- `POLLER` — I/O 轮询机制选择（epoll/kqueue/poll/select/devpoll/pollset）

---

## 4. 核心架构

### 4.1 分层架构

libzmq 采用清晰的分层架构：

```
┌─────────────────────────────────────────────────┐
│            公共 C API 层 (zmq.h)                 │
│  zmq_ctx_new, zmq_socket, zmq_send, zmq_recv... │
├─────────────────────────────────────────────────┤
│              Socket 类型层                       │
│  REQ/REP, PUB/SUB, PUSH/PULL, DEALER/ROUTER... │
├─────────────────────────────────────────────────┤
│            会话层 (Session)                       │
│  session_base_t — 管理连接的消息路由              │
├─────────────────────────────────────────────────┤
│          引擎层 (Engine)                         │
│  stream_engine, raw_engine, ws_engine, udp_engine│
├─────────────────────────────────────────────────┤
│        安全机制层 (Mechanism)                     │
│  NULL, PLAIN, CURVE, GSSAPI                      │
├─────────────────────────────────────────────────┤
│        传输层 (Transport)                        │
│  TCP, IPC, inproc, PGM, TIPC, UDP, WebSocket    │
├─────────────────────────────────────────────────┤
│        I/O 线程 & 轮询层                         │
│  io_thread_t + epoll/kqueue/poll/select          │
└─────────────────────────────────────────────────┘
```

### 4.2 核心类继承体系

```
object_t                    # 所有参与线程间通信的对象基类
├── own_t                   # 所有权层级管理基类
│   ├── socket_base_t       # Socket 基类（也继承 i_poll_events, i_pipe_events）
│   │   ├── req_t / rep_t           # 请求/应答模式
│   │   ├── pub_t / sub_t           # 发布/订阅模式
│   │   ├── push_t / pull_t         # 管道模式
│   │   ├── dealer_t / router_t     # 异步请求/路由模式
│   │   ├── pair_t                  # 对等模式
│   │   ├── xpub_t / xsub_t        # 扩展发布/订阅
│   │   ├── stream_t                # 原始 TCP 流
│   │   ├── server_t / client_t     # 客户端/服务器 (Draft)
│   │   ├── radio_t / dish_t        # 组播 (Draft)
│   │   ├── scatter_t / gather_t    # 散射/收集 (Draft)
│   │   ├── dgram_t                 # 数据报 (Draft)
│   │   └── peer_t                  # 对等 (Draft)
│   ├── session_base_t              # 会话管理（也继承 io_object_t）
│   ├── stream_connecter_base_t     # 连接器基类
│   │   ├── tcp_connecter_t
│   │   ├── ipc_connecter_t
│   │   ├── ws_connecter_t
│   │   └── socks_connecter_t
│   ├── stream_listener_base_t      # 监听器基类
│   │   ├── tcp_listener_t
│   │   ├── ipc_listener_t
│   │   └── ws_listener_t
│   └── reaper_t                    # 资源回收线程
├── io_thread_t                     # I/O 线程（也继承 i_poll_events）
└── pipe_t                          # 消息管道

i_engine (接口)             # 引擎抽象接口
├── stream_engine_base_t    # 流式引擎基类（也继承 io_object_t）
│   ├── zmtp_engine_t       # ZMTP 协议引擎
│   ├── raw_engine_t        # 原始流引擎
│   ├── ws_engine_t         # WebSocket 引擎
│   └── wss_engine_t        # WebSocket over TLS 引擎
├── udp_engine_t            # UDP 引擎
├── norm_engine_t           # NORM 多播引擎
└── pgm_sender_t/receiver_t # PGM 多播引擎

mechanism_t (安全机制基类)
├── null_mechanism_t        # 无安全
├── plain_client_t / plain_server_t    # 明文用户名/密码
├── curve_client_t / curve_server_t    # CURVE 椭圆曲线加密
└── gssapi_client_t / gssapi_server_t  # GSSAPI/Kerberos
```

---

## 5. 关键组件详解

### 5.1 Context (`ctx_t`)
全局上下文对象，封装了所有库级全局状态：
- 管理 **I/O 线程池**（默认 1 个 I/O 线程）
- 管理所有 **Socket 实例**的生命周期
- 维护 **inproc 端点** 注册表（进程内传输）
- 管理 **Reaper 线程**（负责回收关闭的 Socket）
- 继承自 `thread_ctx_t`，管理线程调度参数

### 5.2 Socket (`socket_base_t`)
所有 Socket 类型的基类，提供：
- `bind()` / `connect()` — 绑定/连接地址
- `send()` / `recv()` — 发送/接收消息
- 管理与多个 `pipe_t` 的连接
- 通过 Mailbox 与 I/O 线程进行异步通信
- 各具体 Socket 类型（REQ/REP/PUB/SUB 等）通过重写 `xsend()`/`xrecv()`/`xhas_in()`/`xhas_out()` 实现不同的消息分发策略

### 5.3 Pipe (`pipe_t`)
线程安全的消息管道，用于 Socket 与 Session 之间的消息传递：
- 总是成对创建 (`pipepair()`)
- 基于无锁队列 (`ypipe_t`) 实现高性能
- 支持 HWM（高水位标记）流控
- 通过 `i_pipe_events` 接口通知 Socket 和 Session

### 5.4 Session (`session_base_t`)
管理单个连接的会话：
- 连接 Socket（通过 Pipe）和 Engine
- 处理消息在 Socket 和网络之间的路由
- 管理重连逻辑
- 运行在 I/O 线程中

### 5.5 Engine (`i_engine`)
协议引擎，负责实际的网络 I/O：
- `stream_engine_base_t` — 流式传输的通用引擎（TCP/IPC/WS）
- 负责消息的编码/解码（Encoder/Decoder）
- 集成安全机制（Mechanism）进行握手和加密
- 处理 ZMTP 协议的版本协商

### 5.6 I/O 线程 (`io_thread_t`)
独立的 I/O 工作线程：
- 每个线程拥有一个 **Poller**（epoll/kqueue/poll/select）
- 通过 **Mailbox** 接收来自应用线程的命令
- 负责所有网络 I/O 的非阻塞处理
- Context 会选择负载最低的 I/O 线程分配给新连接

### 5.7 线程间通信机制
libzmq 内部使用 **命令模式** 进行线程间通信：
- `command_t` — 定义所有内部命令类型（plug, bind, attach, term 等）
- `mailbox_t` — 线程安全的命令信箱（基于 Signaler + 管道/socketpair）
- `object_t::send_*()` — 发送命令到目标对象
- `object_t::process_command()` — 处理接收到的命令

### 5.8 消息 (`msg_t`)
高效的消息表示：
- 小消息（≤ 若干字节）内联存储，避免堆分配
- 大消息引用计数管理
- 支持多帧消息（multi-part）
- 零拷贝设计

---

## 6. 传输协议

| 传输协议 | 描述 | 关键文件 |
|---------|------|---------|
| **TCP** | 标准 TCP 连接 | `tcp_*.cpp/hpp` |
| **IPC** | Unix 域套接字（进程间通信） | `ipc_*.cpp/hpp` |
| **inproc** | 进程内传输（零拷贝） | 通过 `ctx_t` 端点注册 |
| **PGM/EPGM** | 可靠多播（需 OpenPGM） | `pgm_*.cpp/hpp` |
| **TIPC** | 集群间通信 (Linux) | `tipc_*.cpp/hpp` |
| **UDP** | 数据报 (Draft) | `udp_*.cpp/hpp` |
| **WebSocket** | WebSocket (Draft) | `ws_*.cpp/hpp` |
| **WSS** | WebSocket over TLS (Draft) | `wss_*.cpp/hpp` |
| **VMCI** | VMware VMCI | 可选构建 |
| **NORM** | NORM 多播协议 | `norm_engine.cpp/hpp` |

---

## 7. 安全架构

libzmq 支持多种安全机制，通过 ZAP（ZeroMQ Authentication Protocol）框架集成：

| 机制 | 描述 | 依赖 |
|-----|------|------|
| **NULL** | 无安全（默认） | 无 |
| **PLAIN** | 明文用户名/密码 | 无 |
| **CURVE** | 基于 CurveZMQ 的椭圆曲线加密 | libsodium 或 tweetnacl |
| **GSSAPI** | Kerberos 认证 | libgssapi |

---

## 8. 数据流概览

```
应用程序线程                          I/O 线程
┌──────────┐                    ┌──────────────┐
│  zmq_send │                    │  io_thread_t │
│     ↓     │                    │      ↑       │
│ socket_t  │                    │   poller_t   │
│     ↓     │                    │   (epoll)    │
│  pipe_t ══╪════ 无锁队列 ══════╪═► session_t  │
│           │                    │      ↓       │
│           │                    │  engine_t    │
│           │                    │      ↓       │
│           │                    │  encoder_t   │
│           │                    │      ↓       │
│           │                    │   TCP/IPC    │
└──────────┘                    └──────────────┘
```

1. 应用线程调用 `zmq_send()` → 消息写入 `pipe_t` 的无锁队列
2. I/O 线程的 Poller 检测到管道可写信号
3. `session_base_t` 从管道读取消息
4. `engine_t` 通过 `encoder_t` 编码消息为 ZMTP 帧
5. 数据通过实际的网络 socket 发送

接收过程是相反方向。

---

## 9. 消息模式 (Socket Types)

### 稳定模式 (Stable)
| 模式 | 发送端 | 接收端 | 描述 |
|------|--------|--------|------|
| **Request/Reply** | REQ | REP | 同步请求-应答 |
| **Pub/Sub** | PUB | SUB | 发布-订阅 |
| **Pipeline** | PUSH | PULL | 单向管道/负载均衡 |
| **Exclusive Pair** | PAIR | PAIR | 双向一对一 |
| **Extended Pub/Sub** | XPUB | XSUB | 带订阅转发的发布-订阅 |
| **Router/Dealer** | ROUTER | DEALER | 异步扩展的请求-应答 |
| **Stream** | STREAM | STREAM | 原始 TCP 流 |

### Draft 模式
| 模式 | 发送端 | 接收端 | 描述 |
|------|--------|--------|------|
| **Client/Server** | CLIENT | SERVER | 异步客户端-服务器 |
| **Radio/Dish** | RADIO | DISH | 基于组的发布-订阅 |
| **Scatter/Gather** | SCATTER | GATHER | 散射-收集 |
| **Dgram** | DGRAM | DGRAM | 数据报 |
| **Peer** | PEER | PEER | 对等通信 |

---

## 10. 测试结构

- **集成测试** (`tests/`): 126 个测试文件，覆盖各种 Socket 类型、传输协议、安全机制、边界条件等
- **单元测试** (`unittests/`): 8 个文件，针对核心数据结构的独立测试
  - `unittest_ip_resolver.cpp` — IP 解析器
  - `unittest_poller.cpp` — Poller 轮询器
  - `unittest_mtrie.cpp` — 消息 Trie 树
  - `unittest_radix_tree.cpp` — 基数树
  - `unittest_ypipe.cpp` — 无锁管道
  - `unittest_udp_address.cpp` — UDP 地址
  - `unittest_curve_encoding.cpp` — CURVE 编码
- **性能测试** (`perf/`): 吞吐量和延迟的基准测试工具

---

## 11. 设计亮点

1. **无锁数据结构**: `ypipe_t`（无锁管道）、`atomic_counter_t`、`atomic_ptr_t` 实现高性能线程间通信
2. **对象所有权模型**: 通过 `own_t` 实现清晰的对象生命周期管理和线程安全的资源释放
3. **命令模式通信**: 所有跨线程交互通过类型化命令（`command_t`）和 Mailbox 完成，避免共享状态
4. **I/O 多路复用抽象**: 通过 `poller_t` 类型别名统一不同平台的 I/O 多路复用机制
5. **协议引擎可插拔**: 通过 `i_engine` 接口实现传输协议与 Socket 逻辑的解耦
6. **安全机制可插拔**: 通过 `mechanism_t` 基类支持多种安全方案的无缝切换
7. **订阅过滤**: 支持 Trie 树和 Radix 树两种实现来高效管理 PUB/SUB 订阅
