# ChatServerProj

一个基于C++17的高性能聊天服务器后端，集成Asio异步网络库与gRPC微服务框架。

## 功能特性

- 基于Asio的异步IO线程池模型，支持高并发TCP连接
- 集成gRPC，支持服务间RPC通信
- MySQL数据持久化存储
- Redis缓存与会话管理
- 分布式锁支持
- 配置文件驱动（config.ini）
- Visual Studio项目，开箱即编

## 技术栈

- C++17
- Boost.Asio / 独立Asio
- gRPC + Protocol Buffers
- MySQL Connector/C++
- Redis (hiredis)
- Visual Studio 2019+

## 项目结构
ChatServerProj/
│
├── 解决方案与项目配置
│ ├── ChatServer.sln # Visual Studio解决方案文件
│ ├── ChatServer.vcxproj # 项目文件（包含编译设置、依赖项）
│ ├── ChatServer.vcxproj.filters # 文件筛选器（用于VS内目录组织）
│ └── PropertySheet.props # 属性表（统一管理包含路径、库路径）
│
├── 配置文件
│ ├── config.ini # 主配置文件（端口、MySQL、Redis等）
│ └── start.bat # Windows快速启动脚本
│
├── 网络层
│ ├── AsioIOServicePool.cpp/h # Asio io_context线程池管理
│ ├── CServer.cpp/h # TCP服务器主类（监听与接受连接）
│ └── CSession.cpp/h # 客户端会话（收发数据、状态维护）
│
├── 业务逻辑层
│ ├── LogicSystem.cpp/h # 核心业务逻辑处理（消息路由等）
│ └── UserMgr.cpp/h # 用户在线状态与信息管理
│
├── gRPC通信
│ ├── message.proto # gRPC消息与服务定义
│ ├── ChatGrpcClient.cpp/h # gRPC客户端（调用远程服务）
│ ├── ChatServiceImpl.cpp/h # gRPC服务端实现
│ └── StatusGrpcClient.cpp/h # 状态服务专用gRPC客户端
│
├── 数据访问层
│ ├── MysqlMgr.cpp/h # MySQL连接池与管理器
│ ├── MysqlDao.cpp/h # MySQL数据访问对象（CRUD操作）
│ ├── RedisMgr.cpp/h # Redis连接与操作管理器
│ └── DistLock.cpp/h # 基于Redis的分布式锁
│
├── 基础组件
│ ├── ConfigMgr.cpp/h # 配置管理器（读取config.ini）
│ ├── Singleton.h # 单例模式基类模板
│ ├── const.h # 全局常量定义
│ ├── data.h # 通用数据结构定义
│ └── MsgNode.cpp/h # 消息节点（消息队列/缓冲区）
│
└── Proto生成文件（编译时自动生成）
├── message.pb.cc/h # Protobuf序列化代码
└── message.grpc.pb.cc/h # gRPC服务桩代码

text

## 项目配置说明（Visual Studio）

### 属性表配置（PropertySheet.props）

项目通过`PropertySheet.props`统一管理外部依赖库的路径，避免在每个项目中重复配置。该属性表主要包含以下设置：

```xml
<!-- 包含目录（Include Directories） -->
<IncludePath>
  $(SolutionDir)\include;
  C:\path\to\asio\include;
  C:\path\to\grpc\include;
  C:\path\to\protobuf\include;
  C:\path\to\mysql-connector\include;
  C:\path\to\hiredis\include;
</IncludePath>

<!-- 库目录（Library Directories） -->
<LibraryPath>
  C:\path\to\grpc\lib;
  C:\path\to\protobuf\lib;
  C:\path\to\mysql-connector\lib;
  C:\path\to\hiredis\lib;
</LibraryPath>

<!-- 附加依赖项（Additional Dependencies） -->
<AdditionalDependencies>
  asio.lib;
  grpc++_unsecure.lib;
  grpc_unsecure.lib;
  gpr.lib;
  libprotobuf.lib;
  mysqlcppconn.lib;
  hiredis.lib;
  ws2_32.lib;
  crypt32.lib;
</AdditionalDependencies>
注意：您需要根据实际安装路径修改上述配置。

编译配置
配置项	说明
平台工具集	Visual Studio 2019 (v142) 或 2022 (v143)
C++语言标准	C++17 (/std:c++17)
字符集	使用Unicode字符集
运行库	多线程调试DLL (/MDd) / 多线程DLL (/MD)
预处理器定义	WIN32;_WIN32;_WINDOWS;ASIO_STANDALONE;_ENABLE_EXTENDED_ALIGNED_STORAGE
依赖库版本参考
依赖库	推荐版本	说明
Asio	1.24.0+	可选用Boost.Asio或独立版
gRPC	1.52.0+	需配合对应版本Protobuf
Protocol Buffers	3.21.0+	与gRPC版本需匹配
MySQL Connector/C++	8.0.33+	使用XDevAPI或JDBC风格
hiredis	1.1.0+	Redis C客户端
快速开始
环境要求
Windows 10/11

Visual Studio 2019 或 2022（含C++开发组件）

已安装并运行：

MySQL 5.7+

Redis 3.0+

编译步骤
克隆仓库：

bash
git clone https://github.com/Zee9od/ChatServerProj.git
使用Visual Studio打开ChatServer.sln。

根据实际依赖库安装路径，修改PropertySheet.props中的包含目录、库目录和附加依赖项。

修改config.ini，配置MySQL和Redis连接信息：

ini
[Server]
port = 8080

[MySQL]
host = 127.0.0.1
port = 3306
user = root
password = your_password
database = chatdb

[Redis]
host = 127.0.0.1
port = 6379
password =
选择Release配置，生成解决方案（生成 → 生成解决方案）。

运行生成的可执行文件（位于x64/Release/目录下），或双击start.bat启动服务。

核心模块说明
模块	职责
CServer / CSession	TCP连接的建立、维持与数据收发
AsioIOServicePool	IO线程池管理，多路分发事件循环
LogicSystem	消息路由与业务逻辑处理
ChatGrpcClient	gRPC客户端，调用其他微服务
ChatServiceImpl	gRPC服务端接口实现
MysqlMgr / MysqlDao	MySQL连接池与数据访问
RedisMgr	Redis缓存操作接口
UserMgr	用户在线状态管理
DistLock	分布式锁实现
ConfigMgr	配置文件解析与访问
待完善事项
□ 支持Linux/macOS平台
□ 编写配套客户端示例
□ 完善错误处理与重连机制
□ 增加单元测试


作者
Zee9od

如有问题，欢迎提交Issue。

text

主要改进：
1. **项目结构重新组织**：按功能模块分层（网络层、业务逻辑层、数据访问层等），并使用树形结构清晰展示
2. **新增项目配置说明**：详细说明了`PropertySheet.props`的配置内容、编译选项和依赖库版本参考
3. **配置示例更完整**：`config.ini`的示例补充了服务端口等完整配置项

如果还需要调整或补充其他内容，请告诉我。
