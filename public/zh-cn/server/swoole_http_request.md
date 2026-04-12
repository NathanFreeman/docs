# Swoole\Http\Request

`HTTP`请求对象封装了客户端发起的`HTTP`请求信息，包含`GET`参数、`POST`数据、`Cookie`以及`Header`等关键内容。


## 属性

### fd
- 当前请求的客户端文件描述符，是一个`int`类型的整数。

* **示例**

```php
<?php
use Swoole\Http\Server;
use Swoole\Http\Request;
use Swoole\Http\Response;

$http = new Server('127.0.0.1', 9501);
$http->on('request', function(Request $request, Response $response) {
  var_dump($request->fd);
  $response->end("<h1>Hello Swoole</h1>");
});

$http->start(); 
```

### streamId
- HTTP/2 在一个 `TCP` 连接上可以同时传输多个请求和响应，每个独立的“请求-响应”交互被称为一个流。`streamId` 就是用来区分这些不同流的 ID 号。

* **示例**

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
  var_dump($request->streamId);
  $response->end("<h1>Hello Swoole</h1>");
});

$http->start();
```

### header
- `HTTP`请求的头部信息。类型为数组，所有`key`均为小写。

* **示例**

```php
<?php
use Swoole\Http\Server;
use Swoole\Http\Request;
use Swoole\Http\Response;

$http = new Server('127.0.0.1', 9501);
$http->on('request', function(Request $request, Response $response) {
  $name = $request->header['server'] ?? 'Swoole';
  $response->end("<h1>Hello $name</h1>");
});

$http->start(); 
```

### server

- `HTTP`请求相关的服务器信息，相当于`PHP`的`$_SERVER`数组。包含了`HTTP`请求的方法，`URL`路径，客户端`IP`等信息。

* **示例**

```php
<?php
use Swoole\Http\Server;
use Swoole\Http\Request;
use Swoole\Http\Response;

$http = new Server('127.0.0.1', 9501);
$http->on('request', function(Request $request, Response $response) {
  $method = $request->server['request_method'];
  $response->end("<h1>Hello $method</h1>");
});

$http->start(); 
```

---

* **参数信息**

| 键名                   | 说明                                                                                                                                     |
|----------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| `query_string`       | 请求的 `GET` 参数字符串（如 `id=1&cid=2`）；若不存在则省略                                                                                                |
| `request_method`     | HTTP 请求方法（如 `GET`、`POST`）                                                                                                              |
| `request_uri`        | 不含 `GET` 参数的请求路径（如 `/favicon.ico`）                                                                                                     |
| `path_info`          | 同 `request_uri`                                                                                                                        |
| `request_time`       | 请求开始处理的时间戳（秒）。<br>在 `SWOOLE_PROCESS` 模式下因存在 `dispatch` 过程，可能滞后于实际收包时间；高负载时偏差更明显。可通过 `$server->getClientInfo()` 中的 `last_time` 获取精确收包时间 |
| `request_time_float` | 请求开始处理的微秒级时间戳，`float` 类型（如 `1576220199.2725`）                                                                                          |
| `server_protocol`    | 服务器协议版本：`HTTP/1.0`、`HTTP/1.1` 或 `HTTP/2`                                                                                               |
| `server_port`        | 服务器监听的端口                                                                                                                               |
| `remote_port`        | 客户端端口                                                                                                                                  |
| `remote_addr`        | 客户端 IP 地址                                                                                                                              |
| `server_addr`        | 接收请求的服务器网卡地址，Swoole 6.2+才有该键值                                                                                                          |
| `master_time`        | 连接最后一次通信的时间                                                                                                                            |

### cookie
- `HTTP`请求携带的`COOKIE`信息，相当于`PHP`中的`$_COOKIE`，格式为数组。

* **示例**

```php
<?php
use Swoole\Http\Server;
use Swoole\Http\Request;
use Swoole\Http\Response;

$http = new Server('127.0.0.1', 9501);
$http->on('request', function(Request $request, Response $response) {
  $name = $request->cookie['name'] ?? 'Swoole';
  $response->end("<h1>Hello $name</h1>");
});

$http->start(); 
```


### get
- `HTTP`请求的`GET`参数，相当于`PHP`中的`$_GET`，格式为数组。

* **示例**

```php
<?php
use Swoole\Http\Server;
use Swoole\Http\Request;
use Swoole\Http\Response;

$http = new Server('127.0.0.1', 9501);
$http->on('request', function(Request $request, Response $response) {
  $name = $request->get['name'] ?? 'Swoole';
  $response->end("<h1>Hello $name</h1>");
});

$http->start(); 
```

### post

- `HTTP`请求的`POST`参数，相当于`PHP`中的`$_POST`，格式为数组。

* **示例**

```php
<?php
use Swoole\Http\Server;
use Swoole\Http\Request;
use Swoole\Http\Response;

$http = new Server('127.0.0.1', 9501);
$http->on('request', function(Request $request, Response $response) {
  $name = $request->post['name'] ?? 'Swoole';
  $response->end("<h1>Hello $name</h1>");
});

$http->start(); 
```

### files
- 上传文件信息，类型为以`form`名称为`key`的二维数组。与`PHP`的`$_FILES`相同。最大文件尺寸不得超过[package_max_length](/server/setting?id=package_max_length)设置的值。因为Swoole在解析报文的时候是会占用内存的，报文越大，内存占用越大，因此请勿使用`Swoole\Http\Server`处理大文件上传或者由用户自行设计断点续传的功能。

* **示例**

```php
<?php
use Swoole\Http\Server;
use Swoole\Http\Request;
use Swoole\Http\Response;

$http = new Server('127.0.0.1', 9501);
$http->on('request', function(Request $request, Response $response) {
  var_dump($request->files);
  $response->end("<h1>Hello $name</h1>");
});

$http->start(); 
```

---

* **参数信息**

| 键名                   | 说明                               |
|----------------------|----------------------------------|
| `name`       | 浏览器上传时传入的文件名称                    |
| `type`     | MIME类型                           |
| `tmp_name`        | 上传的临时文件，文件名以/tmp/swoole.upfile开头 |
| `error`          | 错误码                              |
| `size`       | 上传文件大小                           |

## 方法

### getContent

- 获取原始 HTTP 请求主体（Body），主要用于处理 `application/x-www-form-urlencoded` 之外的 `POST` 请求类型（如 `application/json`、`text/xml` 等）。

- 此方法等效于 PHP 中的 `fopen('php://input')`。

```php
Swoole\Http\Request->getContent(): string|false
```

  * **返回值**

    * 成功时返回原始的请求主体字符串
    * 若当前请求的上下文连接已不存在，返回 `false`

---

* **示例**

```php
<?php
use Swoole\Http\Server;
use Swoole\Http\Request;
use Swoole\Http\Response;

$http = new Server('127.0.0.1', 9501);
$http->on('request', function(Request $request, Response $response) {
  var_dump(json_decode($request->getContent(), true));
  $response->end("<h1>Hello Swoole</h1>");
});

$http->start(); 
```

- 新开一个终端，使用`curl`发送`application/json`请求。

```shell
curl -X POST 127.0.0.1:9501 -H "Content-Type: application/json" -d '{"name":"张三","age":25}'
```

- 服务器终端输出结果为

```shell
array(2) {
  ["name"]=>
  string(6) "张三"
  ["age"]=>
  int(25)
}
```

---

!> Swoole版本 >= `v4.5.0` 可用, 在低版本可使用别名`rawContent` (此别名将永久保留, 即向下兼容)

### getData

- 获取完整的原始`Http`请求报文，注意`Http2`下无法使用。

```php
Swoole\Http\Request->getData(): string|false
```

* **返回值**

  * 执行成功返回报文，如果上下文连接不存在或者在`Http2`模式下返回`false`

---

* **示例**

```php
<?php
use Swoole\Http\Server;
use Swoole\Http\Request;
use Swoole\Http\Response;

$http = new Server('127.0.0.1', 9501);
$http->on('request', function(Request $request, Response $response) {
  $protocol = $request->getData();
  echo $protocol . PHP_EOL;
  $response->end("<h1>Hello Swoole</h1>");
});

$http->start(); 
```

- 新开一个终端，使用`curl`发送`application/json`请求。

```shell
curl -X POST 127.0.0.1:9501 -H "Content-Type: application/json" -d '{"name":"张三","age":25}'
```


- 输出结果为。

```shell
POST / HTTP/1.1
Host: 127.0.0.1:9501
User-Agent: curl/8.5.0
Accept: */*
Content-Type: application/json
Content-Length: 26

{"name":"张三","age":25}
```

---

### create

创建一个 `Swoole\Http\Request` 对象。

```php
Swoole\Http\Request::create(array $options = []): Swoole\Http\Request
```

* **参数**

  * **`array $options`**
    * **功能**：可选参数，用于设置 `Request` 对象的配置

| 参数                  | 默认值 | 说明                                                   |
| --------------------- | ------ | ------------------------------------------------------ |
| `parse_cookie`        | `true` | 是否解析 Cookie                                        |
| `parse_body`          | `true` | 是否解析 HTTP Body                                     |
| `parse_files`         | `true` | 是否解析上传的文件                                     |
| `enable_compression`  | `false`（若服务器不支持压缩报文） | 是否启用压缩                                           |
| `compression_level`   | `1`    | 压缩级别（1-9），值越高压缩越小，但 CPU 消耗越大       |
| `upload_tmp_dir`      | `/tmp` | 上传文件的临时存储目录                                 |

* **返回值**

  * 返回一个`Swoole\Http\Request`对象

---


* **示例：解析 HTTP 报文**

假设你有一段 HTTP 原始报文，需要从中提取关键信息，可以这样操作：

```php
<?php
use Swoole\Http\Request;

$protocol = "POST / HTTP/1.1\r\nHost: 127.0.0.1:9501\r\nUser-Agent: curl/8.5.0\r\nAccept: */*\r\nContent-Type: application/json\r\nContent-Length: 26\r\n\r\n{\"name\":\"张三\",\"age\":25}";

$request = Request::create(['parse_body' => true]);
$request->parse($protocol);
$data = $request->getContent();
echo $data;
```

---

!> Swoole版本 >= `v4.6.0` 可用

### parse
- 解析一个`HTTP`报文。

```php
Swoole\Http\Request->parse(string $data): int|false
```

* **参数**

  * **`string $data`**
    * 要解析的报文

  * **返回值**

    * 解析成功返回解析的报文长度，连接上下文不存在或者上下文已经结束返回`false`

---

* **示例**

```php
<?php
use Swoole\Http\Request;

$protocol1 = "POST / HTTP/1.1\r\nHost: 127.0.0.1:9501\r\nUser-Agent: curl/8.5.0\r\nAccept: */*\r\n";
$protocol2 = "Content-Type: application/json\r\nContent-Length: 26\r\n\r\n";
$protocol3 = "{\"name\":\"张三\",\"age\":25}";

$request = Request::create(['parse_body' => true]);
$request->parse($protocol1);
$request->parse($protocol2);
$request->parse($protocol3);
$data = $request->getContent();
echo $data;
```

> 支持将一个完整的 `HTTP` 报文拆分成多个片段，并分多次调用 `parse` 方法进行解析。底层会自动维护解析状态，将新传入的数据与之前未解析完的数据拼接，并从上次解析中断的位置继续解析。

!> 只能解析`HTTP/1.1`的报文，无法解析`HTTPS`和`HTTP2`的报文。 


### isCompleted

- 获取当前的`HTTP`请求数据包是否已解析到结尾。

```php
Swoole\Http\Request->isCompleted(): bool
```

* **返回值**

  * `true`表示已经是结尾，`false`表示连接上下文已经结束或者未到结尾

---

* **示例**

```php
use Swoole\Http\Request;

$data = "GET /index.html?hello=world&test=2123 HTTP/1.1\r\n";
$data .= "Host: 127.0.0.1\r\n";
$data .= "Connection: keep-alive\r\n";
$data .= "Pragma: no-cache\r\n";
$data .= "Cache-Control: no-cache\r\n";
$data .= "Upgrade-Insecure-Requests: \r\n";
$data .= "User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/86.0.4240.75 Safari/537.36\r\n";
$data .= "Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9\r\n";
$data .= "Accept-Encoding: gzip, deflate, br\r\n";
$data .= "Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,zh-TW;q=0.7,ja;q=0.6\r\n";
$data .= "Cookie: env=pretest; phpsessid=fcccs2af8673a2f343a61a96551c8523d79ea; username=hantianfeng\r\n";

/** @var Request $req */
$req = Request::create(['parse_cookie' => false]);
var_dump($req);

var_dump($req->isCompleted());
var_dump($req->parse($data));

var_dump($req->parse("\r\n"));
var_dump($req->isCompleted());

var_dump($req);
// 关闭了解析cookie，所以会是null
var_dump($req->cookie);
```

---

!> Swoole版本 >= `v4.6.0` 可用


### getMethod

- 获取当前的`HTTP`请求的请求方式。

```php
Swoole\Http\Request->getMethod(): string|false
```
* **返回值**

  * 成功返回大写的请求方式，`false`表示连接上下文不存在

--- 

* **示例**

```php
<?php
use Swoole\Http\Server;
use Swoole\Http\Request;
use Swoole\Http\Response;

$http = new Server('127.0.0.1', 9501);
$http->on('request', function(Request $request, Response $response) {
  var_dump($request->getMethod());
  $response->end("<h1>Hello Swoole</h1>");
});

$http->start(); 
```
