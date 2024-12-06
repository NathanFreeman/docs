`Swoole\Server` 是 Swoole 扩展中用于创建和管理异步风格服务器的基类。它提供了一系列方法、属性、配置项以及事件，使得开发者能够构建高性能的网络服务器。以下是`Swoole\Server`类的相关信息：

### 方法

- **构造方法**：`__construct()` 用于初始化服务器对象，设置监听地址和端口。
- **事件回调**：`on()` 方法用于注册事件回调函数，如连接建立、接收数据、连接关闭等。
- **启动服务器**：`start()` 方法用于启动服务器，开始接收客户端连接。

### 属性

- `$host`：监听的IP地址。
- `$port`：监听的端口号。
- `$workers`：工作进程数。
- `$settings`：服务器运行时参数配置。

### 配置项

- `worker_num`：指定启动的工作进程数。
- `worker_max_request`：每个工作进程允许处理的最大请求数。
- `max_connections`：服务器允许维持的最大TCP连接数。
- `dispatch_mode`：数据包分发策略，包括轮循模式、固定模式和抢占模式。
- `task_worker_num`：异步任务工作进程数。
- `log_file`：指定日志文件路径。
- `log_level`：日志级别。

### 事件

- **onStart**：服务器启动时触发。
- **onWorkerStart**：工作进程启动时触发。
- **onConnect**：有新连接接入时触发。
- **onReceive**：接收到数据时触发。
- **onClose**：连接关闭时触发。
- **onTask**：异步任务完成时触发。
- **onError**：发生错误时触发。

通过这些配置和事件处理，`Swoole\Server` 类能够灵活地应对各种网络服务需求，提供高效、稳定的服务。