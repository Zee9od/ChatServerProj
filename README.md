
```markdown
# ChatServerProj - 高性能C++聊天服务器

[![C++](https://img.shields.io/badge/C++-17-blue.svg)](https://isocpp.org/)
[![Platform](https://img.shields.io/badge/Platform-Windows-blue.svg)]()
[![License](https://img.shields.io/badge/License-MIT-green.svg)]()

## 📖 项目简介

**ChatServerProj** 是一个基于现代C++（C++17）开发的高性能聊天服务器后端。它采用**异步I/O**与**多线程**模型，结合**微服务**架构思想，实现了稳定、高并发的实时通信服务。该项目不仅支持自定义的TCP长连接协议，还集成了**gRPC**，能够灵活地与其他服务（如状态服务、网关服务）进行交互。

## ✨ 核心特性

*   **高性能网络通信**：基于 **Boost.Asio** 库和 `io_context` 线程池，高效处理成千上万的并发连接。
*   **双重通信机制**：
    *   支持自定义的**TCP私有协议**，用于客户端与服务器的核心消息交互。
    *   集成 **gRPC**，用于服务间的高效、跨语言通信（例如，与状态服务器同步）。
*   **完善的业务逻辑层**：独立的 `LogicSystem` 模块负责消息路由、好友管理、群组聊天等核心业务逻辑。
*   **数据持久化与缓存**：
    *   使用 **MySQL** 作为主要数据库，存储用户资料、聊天记录等持久化数据。
    *   引入 **Redis** 作为高性能缓存，用于管理用户会话、在线状态及实现分布式锁，提升系统响应速度。
*   **模块化与可扩展性**：清晰的分层架构（网络层、业务层、数据访问层），各组件间耦合度低，便于功能扩展与维护。
*   **配置化管理**：通过 `config.ini` 文件集中管理服务端口、数据库连接、Redis地址等配置，方便部署与运维。

## 🛠️ 技术栈

| 技术领域 | 使用技术 |
| :--- | :--- |
| **核心语言** | C++17 |
| **网络框架** | Boost.Asio (或独立的Asio) |
| **RPC框架** | gRPC + Protocol Buffers |
| **关系型数据库** | MySQL (Connector/C++) |
| **缓存数据库** | Redis (hiredis 或类似客户端) |
| **构建工具** | Visual Studio (`.sln`/`.vcxproj`) |
| **设计模式** | 单例模式 (Singleton)、工厂模式、RAII |

## 📁 项目结构

```
ChatServerProj/
├── ChatServer.sln              # Visual Studio 解决方案文件
├── ChatServer.vcxproj          # 项目文件
├── ChatServer.vcxproj.filters  # 项目文件筛选器
├── PropertySheet.props         # 属性表（包含库路径等配置）
├── config.ini                  # 主配置文件
├── start.bat                   # Windows 快速启动脚本
├── const.h                     # 全局常量定义
├── data.h                      # 数据结构定义
├── message.proto               # gRPC 消息与服务定义
├── Singleton.h                 # 单例模式基类
├── AsioIOServicePool.cpp/h     # Asio io_context 线程池
├── CServer.cpp/h               # TCP 服务器主类（监听与接受连接）
├── CSession.cpp/h              # 客户端会话管理（收发与状态）
├── LogicSystem.cpp/h           # 核心业务逻辑处理
├── ChatGrpcClient.cpp/h        # gRPC 客户端（调用远程服务）
├── ChatServiceImpl.cpp/h       # gRPC 服务端实现（响应远程调用）
├── StatusGrpcClient.cpp/h      # 状态服务 gRPC 客户端
├── ConfigMgr.cpp/h             # 配置管理器
├── UserMgr.cpp/h               # 用户在线状态与信息管理
├── DistLock.cpp/h              # 基于 Redis 的分布式锁
├── MysqlDao.cpp/h              # MySQL 数据访问对象
├── MysqlMgr.cpp/h              # MySQL 连接池与管理器
├── RedisMgr.cpp/h              # Redis 连接与操作管理器
├── MsgNode.cpp/h               # 消息节点（可能用于存储转发）
└── ...                         # 其他生成文件 (gRPC/Proto)
```

## 🚀 快速开始

### 环境与依赖

*   **操作系统**：Windows (项目已配置好，其他平台需适配)
*   **编译器**：Visual Studio 2019 或更高版本 (支持C++17)
*   **必需库**：
    *   [Boost.Asio](https://www.boost.org/) 或 [独立的Asio](https://think-async.com/Asio/)
    *   [gRPC](https://grpc.io/) + [Protocol Buffers](https://protobuf.dev/)
    *   [MySQL Connector/C++](https://dev.mysql.com/downloads/connector/cpp/)
    *   [Redis Client (hiredis)](https://github.com/redis/hiredis) (或 Windows 兼容版本)

### 编译与运行步骤

1.  **克隆代码**：
    ```bash
    git clone https://github.com/Zee9od/ChatServerProj.git
    cd ChatServerProj
    ```
2.  **配置环境**：
    *   使用 Visual Studio 打开 `ChatServer.sln` 解决方案文件。
    *   在 `PropertySheet.props` 或项目属性中，确保所有依赖库的 **包含目录**、**库目录** 和 **附加依赖项** 已正确配置到您的本地路径。
    *   根据您的实际环境，修改根目录下的 `config.ini` 文件，正确设置 MySQL、Redis 等服务的连接信息。

3.  **编译项目**：
    *   在 Visual Studio 中，选择 `Release` 或 `Debug` 配置，然后生成解决方案。

4.  **启动服务**：
    *   **方式一**：直接运行编译生成的可执行文件（例如 `ChatServer.exe`）。
    *   **方式二**：双击项目根目录下的 `start.bat` 脚本（如有需要，请先编辑其中的路径）。

## 🔌 未来规划与贡献

*   [ ] 增加更完善的日志系统 (如 spdlog)。
*   [ ] 支持更多平台 (Linux/macOS)。
*   [ ] 开发配套的客户端示例 (如 Qt、WebSocket)。
*   [ ] 实现更丰富的聊天功能 (文件传输、语音视频信令)。

欢迎提交 Issue 或 Pull Request，共同完善这个项目！

## 📄 许可证

本项目采用 [MIT 许可证](https://opensource.org/licenses/MIT)。您可以自由地使用、修改和分发。

## 📧 联系方式

如有任何问题或建议，欢迎通过 [GitHub Issues](https://github.com/Zee9od/ChatServerProj/issues) 与我联系。
```

这份README涵盖了项目介绍、技术架构、快速开始指南等核心部分。您可以根据实际代码的具体实现细节，对“核心特性”和“目录结构”的描述进行微调。如果还需要补充其他部分（例如详细的API文档或部署指南），随时可以告诉我。
