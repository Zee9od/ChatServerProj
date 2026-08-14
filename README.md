# ChatServer 高并发聊天系统

一个基于 **C++17 / boost::asio / gRPC** 实现的分布式即时通讯（IM）聊天服务器，是整个项目的核心服务，负责客户端的 TCP 长连接、消息路由、好友关系与聊天记录的读写。

> 本仓库是 分布式聊天系统中的一个子服务（ChatServer），配合 GateServer / StatusServer / VarifyServer 与 Qt 客户端组成完整的 IM 系统。

---

## 目录

- [项目背景](#项目背景)
- [功能特性](#功能特性)
- [系统架构](#系统架构)
- [通信协议](#通信协议)
- [目录结构](#目录结构)
- [本地部署教程](#本地部署教程)
- [已知问题与注意事项](#已知问题与注意事项)
- [技术栈](#技术栈)

---

## 项目背景

这个项目是 C++ 后端综合技术的实战项目，涵盖**网络编程（asio）**、**RPC（gRPC）**、**并发编程**、**数据库（MySQL + Redis）** 以及 **Qt 客户端开发**等多种技术。

ChatServer 是其中的核心服务，承担最重的业务：

- 与客户端建立 TCP 长连接，采用自定义二进制协议收发 JSON 消息；
- 通过 **Redis** 管理用户在线状态、会话绑定、分布式锁与热点数据缓存；
- 通过 **MySQL** 持久化用户、好友关系与聊天记录；
- 通过 **gRPC** 与其它 ChatServer 实例以及 StatusServer 通信，实现分布式（多服务实例）下的好友通知、踢人下线、跨服消息等能力。


---

## 功能特性

- **TCP 长连接**：基于 boost::asio 的异步网络模型，`AsioIOServicePool` 按 CPU 核数创建 io_context，轮询分配，充分利用多核。
- **自定义二进制协议**：4 字节固定头（2 字节消息 ID + 2 字节长度），体为 JSON，支持粘包/拆包处理。
- **单线程逻辑层**：`LogicSystem` 采用「生产者-消费者」模型，网络线程投递消息，单工作线程按注册的回调分发处理，避免多线程竞争。
- **心跳检测**：`CServer` 60s 定时器扫描所有会话，超时自动关闭并清理；在线人数实时上报 Redis。
- **用户登录鉴权**：客户端登录时携带 uid + token，服务端从 Redis 校验 token，并处理「同一账号多地登录」的踢人逻辑（同服本地踢 / 跨服 gRPC 踢）。
- **好友系统**：好友搜索（支持 uid / 昵称）、申请、认证（同意/拒绝），同服直接投递，跨服通过 gRPC 转发。
- **文本聊天**：消息入库 MySQL 后实时推送给对方；支持加载历史聊天记录。
- **分布式**：基于 Redis 的分布式锁 `lock_{uid}` 防止并发登录冲突；`uip_{uid}` 记录用户所在服务器，支持多实例横向扩展。
- **连接池**：MySQL、Redis、gRPC Channel 均实现连接池 + 定时健康检查 + 自动重连。

---

## 系统架构

### 整体架构

```
                        ┌────────────────────┐
                        │   Qt Desktop 客户端  │
                        │  (client/llfcchat)  │
                        └──────────┬─────────┘
                  HTTP(8080)       │  TCP(8090/8091)
                                   ▼
   ┌──────────────┐        ┌──────────────┐       ┌────────────────┐
   │  GateServer  │ gRPC   │ StatusServer │ gRPC  │  VarifyServer  │
   │  (C++/Beast) │ ─────► │   (C++/gRPC) │ ◄──── │ (Node.js/grpc) │
   │  HTTP 网关    │        │  端口 50052   │       │ 端口 50051      │
   └──────────────┘        └──────────────┘       │ 邮箱验证码 / Redis │
                                   │               └────────────────┘
                                   │ gRPC
                                   ▼
                  ┌────────────────────────────────┐
                  │   ChatServer1 (本仓库)          │
                  │  TCP 8090 + gRPC 50055          │
                  └───────────────┬────────────────┘
                                  │ gRPC（跨服通知）
                  ┌───────────────▼────────────────┐
                  │   ChatServer2（同代码不同配置）  │
                  │  TCP 8091 + gRPC 50056          │
                  └───────────────┬────────────────┘
                                  │
              ┌───────────────────┼───────────────────┐
              ▼                   ▼                   ▼
         MySQL(持久化)        Redis(状态/缓存/锁)    邮件SMTP
```

各服务职责：

| 服务                     | 语言/框架               | 端口                  | 职责                                           |
| ------------------------ | ----------------------- | --------------------- | ---------------------------------------------- |
| GateServer               | C++ / boost::beast      | HTTP 8080             | 客户端 HTTP 网关：注册、登录、找回密码         |
| VarifyServer             | Node.js / @grpc/grpc-js | gRPC 50051            | 邮箱验证码生成、校验（验证码存 Redis）         |
| StatusServer             | C++ / gRPC              | gRPC 50052            | 登录态校验、按负载分配 ChatServer 地址与 token |
| **ChatServer（本仓库）** | C++ / asio + gRPC       | TCP 8090 / gRPC 50055 | 核心聊天服务：长连接、好友、消息、跨服通信     |
| ChatServer2              | 同上                    | TCP 8091 / gRPC 50056 | 第二个实例，演示多实例分布式部署               |
| Client                   | Qt (C++)                | -                     | 桌面客户端（HttpManager + TcpManager）         |

### ChatServer 内部架构

```
                         ┌──────────────────────────┐
   TCP 客户端 ──────────► │   AsioIOServicePool      │
                         │  (N 个 io_context + 线程) │
                         └───────────┬──────────────┘
                                     │
                         ┌───────────▼──────────────┐
                         │  CServer (acceptor)       │
                         │  └─ CSession (每连接)     │  ← 读 4 字节头 + body
                         └───────────┬──────────────┘
                                     │ 投递 LogicNode（生产者）
                                     ▼
                         ┌──────────────────────────┐
                         │  LogicSystem              │
                         │  消息队列 + 单工作线程     │  ← 消费者，按 msg_id 分发
                         │  msg_id → 回调函数 map     │
                         └───────┬─────────┬────────┘
                                 │         │
                  ┌──────────────▼──┐   ┌──▼──────────────┐
                  │  MySQL 持久化    │   │  Redis 状态/缓存 │
                  │  MySqlPool      │   │  RedisConPool   │
                  │  用户/好友/消息  │   │  token/在线/锁   │
                  └─────────────────┘   └─────────────────┘
                                 │
                         ┌───────▼────────────────────────┐
                         │  gRPC                          │
                         │  ChatServiceImpl  (服务端)      │
                         │  ChatGrpcClient   (客户端)      │
                         │  StatusGrpcClient (客户端)      │
                         └────────────────────────────────┘
```

### 核心模块说明

| 模块        | 文件                               | 说明                                                         |
| ----------- | ---------------------------------- | ------------------------------------------------------------ |
| 程序入口    | `ChatServer.cpp`                   | 初始化 io_context、启动 TCP + gRPC 服务、注册信号退出        |
| 网络服务    | `CServer.h/cpp`                    | TCP acceptor，管理所有会话，60s 心跳定时器                   |
| 会话管理    | `CSession.h/cpp`                   | 单连接状态机：异步读头/读体、发送队列、心跳、离线清理        |
| 业务逻辑    | `LogicSystem.h/cpp`                | 消息队列 + 回调注册，处理登录/搜索/好友/聊天/心跳/历史/删好友 |
| gRPC 服务   | `ChatServiceImpl.h/cpp`            | 接收其它服务器转发的好友/聊天/踢人通知                       |
| gRPC 客户端 | `ChatGrpcClient.h/cpp`             | 按服务器 IP 维护 Channel 连接池，跨服通知                    |
| MySQL       | `MysqlDao.h/cpp`、`MysqlMgr.h/cpp` | 连接池 + DAO，用户/好友/聊天记录 CRUD                        |
| Redis       | `RedisMgr.h/cpp`                   | 连接池 + 常用命令封装 + 分布式锁                             |
| 配置        | `ConfigMgr.h/cpp`、`config.ini`    | 读取 ini 配置                                                |
| 数据协议    | `message.proto`                    | gRPC 接口定义（由 `start.bat` 生成 `.pb.cc/.h`）             |

---

## 通信协议

### TCP 自定义协议（客户端 ↔ ChatServer）

数据包头固定 4 字节（小端），之后跟消息体：

```
┌──────────────────────────────────────────────┐
│  msg_id(2字节)  │  data_len(2字节)  │ JSON体 │
└──────────────────────────────────────────────┘
       ←──── 4 字节包头 ────→   ← data_len →
```

- `msg_id`：消息类型（见下表）
- `data_len`：后续 JSON 体的字节长度
- 消息体：UTF-8 编码的 JSON 字符串

### 消息 ID 定义（`const.h`）

| 消息 ID   | 名称                           | 方向    | 说明                         |
| --------- | ------------------------------ | ------- | ---------------------------- |
| 1005      | `MSG_CHAT_LOGIN`               | C→S     | 用户登录（携带 uid + token） |
| 1006      | `MSG_CHAT_LOGIN_RSP`           | S→C     | 登录回包                     |
| 1007/1008 | `ID_SEARCH_USER_REQ/RSP`       | C→S/S→C | 搜索用户（uid 或昵称）       |
| 1009/1010 | `ID_ADD_FRIEND_REQ/RSP`        | C→S/S→C | 发送好友申请                 |
| 1011      | `ID_NOTIFY_ADD_FRIEND_REQ`     | S→C     | 通知被申请方有好友申请       |
| 1013/1014 | `ID_AUTH_FRIEND_REQ/RSP`       | C→S/S→C | 认证好友（同意/拒绝）        |
| 1012      | `ID_NOTIFY_AUTH_FRIEND_REQ`    | S→C     | 通知申请方认证结果           |
| 1017/1018 | `ID_TEXT_CHAT_MSG_REQ/RSP`     | C→S/S→C | 文本聊天                     |
| 1019      | `ID_NOTIFY_TEXT_CHAT_MSG_REQ`  | S→C     | 推送聊天消息给接收方         |
| 1021      | `ID_NOTIFY_OFF_LINE_REQ`       | S→C     | 通知用户被顶下线             |
| 1023/1024 | `ID_HEART_BEAT_REQ/RSP`        | C→S/S→C | 心跳                         |
| 1025/1026 | `ID_LOAD_CHAT_HISTORY_REQ/RSP` | C→S/S→C | 加载历史聊天记录             |
| 1027~1029 | `ID_DELETE_FRIEND_*`           | C↔S     | 删除好友                     |

### gRPC 接口（`message.proto`）

| Service       | RPC                             | 说明                                                       |
| ------------- | ------------------------------- | ---------------------------------------------------------- |
| ChatService   | `NotifyAddFriend`               | 跨服通知：给目标 ChatServer 推送好友申请                   |
| ChatService   | `NotifyAuthFriend`              | 跨服通知：认证好友结果                                     |
| ChatService   | `NotifyTextChatMsg`             | 跨服通知：文本聊天消息（客户端已实现，逻辑层暂注释未启用） |
| ChatService   | `NotifyKickUser`                | 跨服踢人：让对方服务器顶掉重复登录用户                     |
| ChatService   | `RplyAddFriend` / `SendChatMsg` | proto 已声明，`ChatServiceImpl` 未实现                     |
| VarifyService | `GetVarifyCode`                 | 获取邮箱验证码（由 VarifyServer 实现）                     |
| StatusService | `GetChatServer` / `Login`       | 分配聊天服务器 / 登录（由 StatusServer 实现）              |

---

## 目录结构

```
server/ChatServer/
├── ChatServer.cpp          # main 入口：TCP + gRPC 双服务
├── CServer.h/.cpp          # TCP 连接接受与会话管理
├── CSession.h/.cpp         # 单连接会话（粘包/拆包、心跳、发送队列）
├── LogicSystem.h/.cpp      # 业务逻辑层（消息分发核心）
├── ChatServiceImpl.h/.cpp  # gRPC 服务端（跨服通知）
├── ChatGrpcClient.h/.cpp   # gRPC 客户端（跨服调用）
├── StatusGrpcClient.h/.cpp # 对接 StatusServer 的 gRPC 客户端
├── MysqlDao.h/.cpp         # MySQL 数据访问
├── MysqlMgr.h/.cpp         # MySQL 单例封装
├── RedisMgr.h/.cpp         # Redis 封装（含分布式锁）
├── DistLock.h/.cpp         # Redis 分布式锁
├── AsioIOServicePool.h/.cpp# asio io_context 线程池
├── ConfigMgr.h/.cpp        # 配置读取（ini）
├── UserMgr.h/.cpp          # 本机 uid → 会话 映射
├── Singleton.h             # 单例模板
├── const.h                 # 错误码 / 消息 ID / Redis key 定义
├── data.h                  # UserInfo / ApplyInfo / ChatMsgData 结构体
├── MsgNode.h/.cpp          # 收发缓冲区节点
├── message.proto           # gRPC 协议定义
├── message.pb.* / message.grpc.pb.*  # proto 生成代码
├── config.ini              # 运行时配置
├── ChatServer.sln/.vcxproj # Visual Studio 工程
├── PropertySheet.props     # 依赖库路径（需按本机修改！）
├── start.bat               # 重新生成 gRPC 代码
└── start_all.bat           # （位于 ../）一键启动 Redis + 各服务
```

---

## 本地部署教程

> 项目在 **Windows + Visual Studio** 环境下开发与测试，仅支持 Windows 平台（依赖 `ws2_32.lib` 等）。

### 1. 环境要求

| 依赖                | 版本建议          | 说明                                          |
| ------------------- | ----------------- | --------------------------------------------- |
| Windows             | 10/11             | 开发环境                                      |
| Visual Studio       | 2022（MSVC v143） | 打开 `.sln` 直接编译                          |
| MySQL               | 8.x               | 数据持久化，默认 `127.0.0.1:3306`             |
| Redis               | 5.x               | 端口 `6380`、密码 `123456`（见 `config.ini`） |
| Boost               | 1.88.0            | asio / beast / uuid / property_tree           |
| gRPC + protobuf     | 手工编译版本      | 含 abseil / re2 / cares / zlib / boringssl    |
| jsoncpp             | 0.5.0             | JSON 解析                                     |
| hiredis             | 最新              | Redis 客户端                                  |
| MySQL Connector/C++ | 8.3.0             | JDBC 风格数据库驱动                           |

> **注意**：`PropertySheet.props` 与 `ChatServer.vcxproj` 中硬编码了本机绝对路径（如 `E:\cppSort\...`），**他人机器必须修改**，见第 4 步。

### 2. 数据库初始化

1. 安装并启动 MySQL 8.x；

2. 使用 Navicat / mysql 命令行导入备份脚本（位于仓库根目录 `sql备份/llfc.sql`），其中包含：

   - `user` 用户表、`friend` 好友表、`friend_apply` 好友申请表、`user_id` 自增发号表；
   - `reg_user` 存储过程（注册时发号 + 事务控制，注册功能依赖它）；

3. 确认 `config.ini` 的 `[Mysql]` 段与实际账号一致：

   ```ini
   [Mysql]
   Host=127.0.0.1
   Port=3306
   User=root
   Passwd=你的MySQL密码
   Schema=mychatproj     # 需改成你导入时的库名（示例脚本是 llfc）
   ```

### 3. Redis 启动

项目约定 Redis 端口为 **6380** 并设置密码，启动命令：

```bat
redis-server --port 6380 --requirepass 123456
```

与 `config.ini` 保持一致即可（也可改配置用默认 6379）。

### 4. 修改依赖路径

打开 `PropertySheet.props`，把其中的 `E:\cppSort\...` 全部替换为你本机的库安装路径：

- 新增 Include 目录：boost、jsoncpp、grpc/include、protobuf/src、hiredis、mysql-connector include
- 新增库目录：boost stage/lib、grpc 编译产物、jsoncpp lib、hiredis lib、mysql-connector lib64/vs14
- 链接库：`libprotobufd.lib`、`grpc.lib`、`grpc++.lib`、`hiredis.lib`、`json_vc71_libmtd.lib`、`mysqlcppconn.lib`、`mysqlcppconn8.lib` 等

`ChatServer.vcxproj` 的 Debug|x64 下也有 mysql-connector 的 include/lib 路径与依赖，需同步修改。

### 5. 编译

1. 用 Visual Studio 打开 `ChatServer.sln`；
2. 平台选 **x64**，配置选 Debug 或 Release；
3. 编译（Build）。
4. 若修改了 `message.proto`，运行 `start.bat` 重新生成 gRPC 代码（需按本机修改其中的 `protoc.exe`、`grpc_cpp_plugin.exe` 路径）。

> 编译成功后 PostBuild 会自动把 `config.ini` 与 `mysqlcppconn*.dll` 拷贝到输出目录。

### 6. 配置 config.ini

```ini
[SelfServer]
Name = chatserver1     # 本实例名，Redis 中按此区分在线归属
Host = 127.0.0.1
Port = 8090            # 客户端 TCP 连接端口
RPCPort = 50055        # 本服务 gRPC 端口

[StatusServer]
Host = 127.0.0.1
Port = 50052

[Mysql] ...
[Redis] ...
```

### 7. 启动完整系统（按依赖顺序）

> ChatServer 依赖 StatusServer 分配、Redis 状态、MySQL 数据，请按序启动：

```bat
:: 1) Redis
redis-server --port 6380 --requirepass 123456

:: 2) VarifyServer（Node.js，邮箱验证码）
cd server/VarifyServer
npm install
node server.js

:: 3) StatusServer（C++/gRPC 50052）—— 用 VS 编译运行
:: 4) GateServer（C++/beast HTTP 8080）—— 用 VS 编译运行
:: 5) ChatServer（本仓库）—— 用 VS 编译运行
```

仓库根目录提供了 `server/start_all.bat`，可一键启动 Redis、VarifyServer 并打开各 VS 工程（注意脚本内路径也是本机绝对路径，需自行修改）。

客户端使用 Qt 工程 `client/llfcchat` 连接验证。

### 8. 部署第二个实例（多实例 / 分布式演示）

复制一份本目录为 `ChatServer2`，修改 `config.ini`：

```ini
[SelfServer]
Name = chatserver2
Host = 0.0.0.0
Port = 8091
RPCPort = 50056

[PeerServer]
Servers = chatserver1
[chatserver1]
Name = chatserver1
Host = 127.0.0.1
Port = 50055
```

`[PeerServer]` 段用于让本实例知道其它实例的 gRPC 地址，跨服通知依赖它。

---

## 已知问题与注意事项

### 部署/环境类

1. **硬编码绝对路径**：`PropertySheet.props`、`ChatServer.vcxproj`、`start.bat`、`start_all.bat` 中写死了 `E:\cppSort\...`、`E:\c++learnDemoDir\...` 等本机路径，换机器必须逐个修改，这是二次部署最大的门槛。
2. **依赖编译困难**：gRPC 需要手工编译（abseil/re2/cares/zlib/boringssl 全家桶），版本不匹配会引发大量链接错误；推荐直接复用课程配套的编译产物目录结构。
3. **源码注释乱码**：部分源文件中文注释为 GBK 编码，用 UTF-8 编辑器打开会乱码（`PropertySheet.props` 已带 `/source-charset:utf-8`，但仍建议统一源码编码）。
4. **仅 Windows**：工程为 MSBuild `.vcxproj`，且链接 `ws2_32.lib`，无 CMake 支持，无法直接跨平台编译。

### 功能/逻辑类

5. **跨服文本聊天未完全打通**：`LogicSystem::DealChatTextMsg` 中「通过 gRPC 转发给对方服务器」的代码被注释掉，当前只在本机 `UserMgr` 中查找目标会话——**同服在线可即时收到，跨服用户收不到**（`ChatGrpcClient::NotifyTextChatMsg` 与 `ChatServiceImpl` 已实现，可恢复注释打通）。
6. **未实现的 RPC**：`ChatService` 的 `RplyAddFriend`、`SendChatMsg` 在 proto 中已声明，但 `ChatServiceImpl` 未实现，调用会返回 UNIMPLEMENTED。
7. **跨服踢人依赖 PeerServer 配置**：ChatServer 的 `config.ini` 中 `[PeerServer]` 需配置对方实例地址，否则跨服通知/踢人找不到对方。
8. **单线程逻辑层瓶颈**：`LogicSystem` 所有消息由单个工作线程串行处理，高并发下消息处理是性能瓶颈（作为学习项目可接受，可拆分为多个 worker）。
9. **ChatServer2 默认连课程作者远程数据库**：`server/ChatServer2/config.ini` 的 MySQL/Redis 指向 `81.68.86.146`，那是原作者服务器，本地运行请改成自己的本地地址。

### 安全类（生产环境务必修复）

10. **密码明文存储**：`user` 表 `pwd` 字段为明文，未做加盐哈希，存在泄露风险。
11. **gRPC 无 TLS**：`InsecureServerCredentials()`，明文传输，仅适合内网/学习环境。
12. **弱口令/明文配置**：Redis 密码 `123456`、MySQL 密码直接写在 `config.ini` 与代码库中，且 `llfc.sql` 包含大量测试用户数据。

---

## 技术栈

| 分类      | 技术                                         |
| --------- | -------------------------------------------- |
| 语言/标准 | C++17，MSVC v143（VS2022）                   |
| 网络      | boost::asio（异步 I/O）、boost::beast        |
| RPC       | gRPC / protobuf3                             |
| 存储      | MySQL 8.x（Connector/C++）、Redis（hiredis） |
| 解析      | jsoncpp                                      |
| 配置      | boost::property_tree (ini)                   |
| 构建      | Visual Studio MSBuild + PropertySheet        |

---

## 致谢


