# 开始使用 nginx

nginx（发音同 engine x）是异步框架的 Web 服务器，也可以用作[反向代理](https://zh.wikipedia.org/wiki/%E5%8F%8D%E5%90%91%E4%BB%A3%E7%90%86)，[负载平衡器](https://zh.wikipedia.org/wiki/%E8%B4%9F%E8%BD%BD%E5%9D%87%E8%A1%A1)和 [HTTP 缓存](https://zh.wikipedia.org/wiki/Web%E7%BC%93%E5%AD%98)。

## 安装 nginx

### nginx 版本

- Mainline： 包括最新功能和错误修复，并始终是最新的。它是可靠的，但它可能包括一些实验模块，它也可能有一些新的bug。
- Stable：不包含所有最新功能，但具有关键错误修复，始终向后移植到 Mainline 版本。建议生产服务器使用稳定版本。

### 安装方式

两种版本都支持一下两种方式安装 nginx，使用 Prebuilt Package 或者 Compiling from Source。

- 使用编译好的 2 进制包是最简单最快捷的方式安装 nginx，这个包包含了几乎所有的 nginx 官方模块并适应大部分的流行操作系统。
- 从源代码编译，这种方式更灵活，你可以添加一些特殊的包，包括一些第三方的包，或应用最新的安全补丁。

### macOS 环境安装

最简单的方式是使用 homebrew 安装，

```bash
brew install nginx
```

验证安装

```bash
sudo nginx
```

打开 http://localhost:8080。

## nginx 控制命令

```bash
nginx -s <SIGNAL>
```
<SIGNAL> 可以为以下值：

- quit – 优雅地关闭
- reload – 重载配置文件
- reopen – 重新打开日志文件
- stop – 直接关闭 (快速关闭)

`kill` 命令也能够直接发送信号给主进程，主进程的 ID 默认被写在 `nginx.pid` 文件，`nginx.pid` 文件的位置可以通过 `nginx.conf` 配置文件进行查看和修改。

```bash
/usr/local/etc/nginx/nginx.conf
```

假设 nginx 主进程 pid 为 1628，给主进程发送 `QUIT` 信令能够使 nginx 优雅地关闭。

```bash
kill -s QUIT 1628
```

要获取所有正在运行的 nginx 进程的列表，可以使用 `ps` 命令。

```bash
ps -ax | grep nginx
```

## 服务静态内容

nginx 的配置通过 `nginx.conf` 文件来设置，为了方便开发可以在 `nginx.conf` include 更多的配置文件。

```
http {
  ...
  include /Users/yuzi/Documents/workspace/www/nginx-conf/*.conf;
}
```

然后就可以在 `nginx-conf` 目录下新建一个 `my-react-demo.conf` 文件用于配置用于单独服务 **my-react-demo** 相关的配置。内容如下：

```
server {
  listen       8000;
  server_name  localhost;

  location / {
    root   /Users/yuzi/Documents/workspace/jscodes/my-react-demo/build;
    index  index.html;
  }
}
```

如果有相对路径例如 `/images/` 但 build 下不存在 images 目录，图片实际存放在另一个目录下，可以添加 location 的方式来将 `/images/` 的请求代理到其他目录。

```
server {
  listen       8000;
  server_name  localhost;

  location / {
    root   /Users/yuzi/Documents/workspace/jscodes/my-react-demo/build;
    index  index.html;
  }

  location /images/ {
    root   /path-to-images;
  }
}
```

另外需要注意的是 **my-react-demo/build** 的权限问题。这里 **my-react-demo/build** 放置在用户所属目录下，默认情况下 nginx 启用用户默认是 nginx 的，对目录没有读的权限，这样浏览器会报 403 错误。可以选择修改目录的权限或者把 nginx 启动用户改为目录的所属用户，推荐直接修改 nginx 的启动用户这样不用每次修改目录权限。

通过 ls 可以看到目录所属用户 yuzi，所属群组是 staff

```bash
> ls -l
total 48
-rw-r--r--  1 yuzi  staff   779 11 20 08:55 asset-manifest.json
-rw-r--r--  1 yuzi  staff  3870 11 20 08:54 favicon.ico
-rw-r--r--  1 yuzi  staff  2057 11 20 08:55 index.html
-rw-r--r--  1 yuzi  staff   306 11 20 08:54 manifest.json
-rw-r--r--  1 yuzi  staff   606 11 20 08:55 precache-manifest.8f533e011155365bd9730298efbc27b8.js
-rw-r--r--  1 yuzi  staff  1041 11 20 08:55 service-worker.js
drwxr-xr-x  5 yuzi  staff   160 11 20 08:55 static
```

向根 `nginx.conf` 文件添加下面一行内容，就能够以将这个用户设置为启动用户，然后 sudo nginx -s reload 重启服务。

```
user yuzi staff
```

> 如果遇到其他的错误，请查看 access.log 和 error.log

## 反向代理

nginx 一个常用的方式是作为 proxy server 来接收客户端的请求，转发到不同的 server，然后将 response 返回给客户端。这个过程被称之为反向代理，而通过客户端代理的过程则被称为正向代理。

通过 nginx 的反向代理，可以实现了在前端层面上的跨域。除了代理本地的服务之外，我们还可以代理到其他线上的服务。例如我们可以将 **/github/** 这个路由代理到 **https://api.github.com**。

```bash
server {
  listen       8001;
  server_name  localhost;
  root   /Users/yuzi/Documents/workspace/jscodes/my-react-demo/build;
}

server {
  # 将默认路由 / 代理到 localhost:8001
  location / {
    proxy_pass http://localhost:8001/
  }

  # 将 /github/ 路径转发到 api.github.com
  location /github/ {
    proxy_pass https://api.github.com/;
  }
}
```

现在我们在访问 http://localhost/github/ 时会自动代理到 https://api.github.com 的页面。同时因为代理是通过 nginx 实现的，浏览器并没有经历跨域，因此也不会出现跨域的问题，这也是反向代理的一个作用。

## HTTPS

先使用 [mkcert](https://github.com/FiloSottile/mkcert) 创建一个本地的 HTTPS 证书，按照说明安装好。
使用 mkcert 创建一个本地证书。

```bash
> mkcert -install
Created a new local CA at "/Users/yuzi/Library/Application Support/mkcert" 💥
Password:
The local CA is now installed in the system trust store! ⚡️
```

nginx 配置 HTTPS 需要在 listen 的端口后面加 `ssl` 标志。同时必须配置 `server_certificate` 和 `private_key`。

```bash
  listen              443 ssl;
  server_name  localhost;
  ssl_certificate     "/Users/yuzi/Library/Application Support/mkcert/localhost+2.pem";
  ssl_certificate_key "/Users/yuzi/Library/Application Support/mkcert/localhost+2-key.pem";
  ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
  ssl_ciphers         HIGH:!aNULL:!MD5;
```

服务器证书 `certificate` 是一个公开的证书，它会被发送给每个连接到服务器的客户端。私钥 `private key` 是一个安全的实体应存储在具有受限访问权限的文件中，但是 nginx 主进程要能读取到它。私钥也可被存放在和证书同样的文件里，这个文件的访问也是受限的：

```bash
  ssl_certificate     www.example.com.cert;
  ssl_certificate_key www.example.com.cert;
```

这种情况下，虽然私钥和证书在同一文件下，只有证书会被发送给客户端。

`ssl_protocols` 和 `ssl_ciphers` 用于限制连接使用 SSL / TLS 的强版本和密码。默认情况下 nginx 使用 “ssl_protocols TLSv1 TLSv1.1 TLSv1.2” 和 “ssl_ciphers HIGH:!aNULL:!MD5”，所以没有必要显式配置它们，这两项的默认值也曾经几次被改变过。

### HTTPS 优化

SSL 操作会消耗 CPU 资源。在多处理器系统上，应运行多个工作进程，最 CPU 密集型的操作是 SSL 握手，有两种方法可以最大限度地减少每个客户端的这些操作数量。第一种是通过启用 keepalive 连接来通过一个连接发送多个请求，第二个是重用 SSL session 参数以避免并行 SSL 握手和后续的连接。

session 存储在工作线程之间共享的 SSL session 高速缓存中，并由 `ssl_session_cache` 指令配置。一兆字节的缓存包含大约 4000 个会话。默认的 cache 过期时间是 5 分钟，可以使用 `ssl_session_timeout` 增加，

```bash
worker_processes auto;

http {
  ssl_session_cache   shared:SSL:10m;
  ssl_session_timeout 10m;

  server {
    listen              443 ssl;
    server_name         www.example.com;
    keepalive_timeout   70;

    ssl_certificate     www.example.com.crt;
    ssl_certificate_key www.example.com.key;
    ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers         HIGH:!aNULL:!MD5;
    ...
```

## HTTP2

nginx 1.9.5 开始提供了 `ngx_http_v2_module` 模块，开启配置如下：

```bash
server {
  listen 443 ssl http2;
  ssl_certificate     "/Users/yuzi/Library/Application Support/mkcert/localhost+2.pem";
  ssl_certificate_key "/Users/yuzi/Library/Application Support/mkcert/localhost+2-key.pem";
}
```

此时在 chrome 的 devtool 里可以看到请求的 protocal 就是 h2。

### 其他的 http2 配置

```bash
# 在开始处理每个请求的请求主体之前，可以保存请求主体的缓冲区大小
http2_body_preread_size size;

# 响应主体被分块的最大大小。值太低会导致更高的开销。太高的值会由于 HOL 阻塞损害优先级
http2_chunk_size size;

# 关闭连接之后的不活动超时时间
http2_idle_timeout time;

# 限制连接中的最大并发推送请求数
http2_max_concurrent_pushes number;

# 连接中的最大并发 HTTP2 流数
http2_max_concurrent_streams number;

# 限制 HPACK 压缩请求标头字段的最大大小
http2_max_field_size size;

# 限制整个请求头在 HPACK 解压缩后的最大大小，默认值通常是足够的
http2_max_header_size size

# 预先地将请求推送到指定的 uri 以及对原始请求的响应。仅处理绝对路径的 URI，例如：http2_push /static/css/main.css;
http2_push uri | off;

# 允许将 Link 标签中指定的预加载链接自动转换为推送请求。
http2_push_preload on | off;

# 每个工作进程输入缓冲区的大小
http2_recv_buffer_size size;

# 从客户端获取更多数据的超时时间，之后关闭连接
http2_recv_timeout time;
```

