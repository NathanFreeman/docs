# HTTP/HTTPS/HTTP2服务器

- 要创建一个 `HTTP`、`HTTPS` 或 `HTTP2` 服务器，只需实例化 `Swoole\Http\Server` 对象即可。这也是 `Swoole` 全系列服务器中使用最广泛的一种。

- `Swoole\Http\Server`、`Swoole\Http\Request` 和 `Swoole\Http\Response` 三者共同构成了一个完整的 HTTP 服务器。其中，[Request对象](/server/swoole_http_requqest)封装了客户端的请求信息，[Response对象](/server/swoole_http_response)则用于向客户端返回响应。以下示例展示了它们的简单用法。

> `Swoole\Http\Server` 是 `Swoole\Server` 的子类。

#### 示例

- 示例1：创建一个简单的 HTTP 服务器

- 在服务器上监听 `127.0.0.1` 的 `9501` 端口，实现一个基本的 HTTP 服务器：

```php
<?php
use Swoole\Http\Server;
use Swoole\Http\Request;
use Swoole\Http\Response;

$http = new Server('127.0.0.1', 9501);
$http->on('request', function(Request $request, Response $response) {
  $name = $request->get['name'];
  $response->end("<h1>Hello $name</h1>");
});

$http->start();
```

- 测试服务器打开另一个终端，使用 `curl` 发送 HTTP 请求：

```bash
curl http://127.0.0.1:9501/?name=Swoole
```

- 我们会看到终端会输出：

```
<h1>Hello Swoole</h1>
```

- 示例2：创建一个简单的 HTTPS 服务器

- 在服务器上监听 `127.0.0.1` 的 `443` 端口，实现一个基本的 HTTPS 服务器：

```php
<?php
use Swoole\Http\Server;
use Swoole\Http\Request;
use Swoole\Http\Response;

$http = new Server('127.0.0.1', 443, SWOOLE_BASE, SWOOLE_SOCK_TCP | SWOOLE_SSL);
// 设置SSL证书
$http->set([
  'ssl_cert_file' =>  '/server.crt', 
  'ssl_key_file' =>  '/server.key',
]);
$http->on('request', function(Request $request, Response $response) {
  $name = $request->get['name'];
  $response->end("<h1>Hello $name</h1>");
});

$http->start();
```

- 测试服务器打开另一个终端，使用 `curl` 发送 HTTPS 请求：

```bash
curl -k https://127.0.0.1/?name=Swoole
```

- 我们会看到终端会输出：

```
<h1>Hello Swoole</h1>
```

!> `HTTPS` 服务器需要在编译`Swoole`的时候开启`--enable-openssl`。`Swoole 6.2`开始，已经默认内置对`openssl`的支持。


- 示例3：创建一个简单的 HTTP2 服务器

- 在服务器上监听 `127.0.0.1` 的 `443` 端口，实现一个基本的 HTTP2 服务器：

```php
<?php
use Swoole\Http\Server;
use Swoole\Http\Request;
use Swoole\Http\Response;

$http = new Server('127.0.0.1', 443, SWOOLE_BASE, SWOOLE_SOCK_TCP | SWOOLE_SSL);
// 设置SSL证书
$http->set([
  'ssl_cert_file' => '/server.crt',
  'ssl_key_file' => '/server.key',
  'open_http2_protocol' => true  // 开启这个以便于解析http2协议
]);
$http->on('request', function(Request $request, Response $response) {
  $name = $request->get['name'];
  $response->end("<h1>Hello $name</h1>");
});

$http->start();
```

- 测试服务器打开另一个终端，使用 `curl` 发送 HTTP2 请求：

```bash
curl -k https://127.0.0.1/
```

- 我们会看到终端会输出：

```
<h1>Hello Swoole</h1>
```

!> `HTTPS` 服务器需要在编译`Swoole`的时候开启`--enable-openssl`和`--enable-http2`。`Swoole 5.0`开始，已经默认内置对`http2`的支持。`Swoole 6.2`开始，已经默认内置对`openssl`的支持。


#### Nginx + Swoole

网络世界中的报文格式千变万化，而 `Swoole` 内置的 `HTTP` 协议支持仅实现了基础功能，能够满足常规应用场景。

因此，**建议将 `Swoole` 仅作为应用服务器**，专注于处理动态请求；而在其前端增加 `Nginx` 作为代理服务器，负责处理静态资源、负载均衡等更复杂的 Web 服务器职责。

以下是一个简单的 `Nginx` 代理配置示例：

```nginx
server {
    listen 80;
    server_name swoole.test;

    location / {
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        proxy_pass http://127.0.0.1:9501;
    }
}
```

通过这样的架构，可以同时发挥 `Nginx` 强大的网络处理能力与 `Swoole` 高性能的动态请求处理能力。
