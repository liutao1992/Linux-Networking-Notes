# Nginx 长连接（keep-alive）使用教程

在现代 Web 服务中，**长连接（Keep-Alive）** 是提升性能和减少资源浪费的重要手段。本文将介绍 Nginx 中如何配置和使用 HTTP 长连接。

---

## 一、什么是长连接？

长连接（HTTP persistent connection）是一种允许 **多个 HTTP 请求/响应在一个 TCP 连接上复用** 的机制。它避免了为每一个请求都建立一次新的 TCP 连接，从而显著提升性能。

### 优点：

* 减少 TCP 握手/挥手的开销
* 降低连接建立延迟
* 节省资源（连接复用）

---

## 二、Nginx 中的长连接实现原理

在 Nginx 中，长连接主要通过以下两个配置参数实现：

* `keepalive_timeout`
* `keepalive_requests`

此外，还可能涉及 `connection` 头部和 `proxy_http_version` 等代理相关参数。

---

## 三、长连接配置示例（客户端与 Nginx）

客户端与 Nginx 之间启用长连接非常简单，只需设置：

```nginx
http {
    server {
        listen 80;

        location / {
            root /usr/share/nginx/html;

            keepalive_timeout 65;      # 设置保持连接的超时时间（单位：秒）
            keepalive_requests 100;    # 一个连接最多处理多少个请求
        }
    }
}
```

### 参数说明：

* `keepalive_timeout 65;`
  设置连接在空闲多少秒后关闭，默认是 75 秒。设置为 `0` 表示禁用 keep-alive。

* `keepalive_requests 100;`
  每个连接允许处理的最大请求数。超过这个数后，连接会被关闭。

### HTTP 头的影响：

如果客户端发送头部 `Connection: keep-alive`，并且服务器支持，那么连接会保持不关闭。

---

## 四、长连接配置示例（Nginx 作为反向代理）

当 Nginx **作为反向代理** 时，你还需要配置 Nginx 与后端服务器之间的连接为 keep-alive。

```nginx
http {
    upstream backend {
        server 127.0.0.1:8080;

        keepalive 32;   # upstream 中连接池大小（即可复用的空闲连接数）
    }

    server {
        listen 80;

        location / {
            proxy_pass http://backend;
            proxy_http_version 1.1;              # 使用 HTTP/1.1 才支持 keep-alive
            proxy_set_header Connection "";      # 清空 connection 头，保持连接
        }
    }
}
```

### 注意事项：

* `proxy_http_version 1.1`：HTTP/1.1 才支持长连接，HTTP/1.0 默认不支持。
* `proxy_set_header Connection ""`：防止 Nginx 添加 `Connection: close` 头。

---


## 五、验证是否启用长连接

启用 Nginx 的 Keep-Alive 后，我们需要验证它是否**真正生效**。以下是几种常用的验证方法：

---

### ✅ 方法一：查看响应头（初步验证）

使用 `curl` 查看响应头中是否包含 `Connection: keep-alive` 字段：

```bash
curl -I http://localhost/ -H "Connection: keep-alive"
```

示例输出：

```
HTTP/1.1 200 OK
Server: nginx
Connection: keep-alive
...
```

说明：

* 如果响应中包含 `Connection: keep-alive`，表示服务器**支持并响应**客户端的 Keep-Alive 请求。
* 但这只能说明**连接允许保持打开**，并不能证明**连接是否真的被复用**。

---

### ✅ 方法二： 使用压测工具

#### ✅ 工具一：使用 `curl -v` 查看连接状态

发送两次请求，观察连接是否关闭或复用：

```bash
curl -v --keepalive-time 60 http://localhost/
```

在输出中查找这些字段：

* `Connection: keep-alive`（响应头）
* `Re-using existing connection`（仅出现在连接复用时）

如果只发一次请求则无法判断复用，建议使用如下脚本测试复用：

```bash
for i in {1..5}; do curl -v --keepalive-time 60 http://localhost/; done
```

---

#### ✅ 工具二：使用 ApacheBench（`ab`）进行并发压测

```bash
ab -n 100 -c 10 -k http://localhost/
```

参数说明：

* `-n 100`：总共请求 100 次
* `-c 10`：10 个并发用户
* `-k`：启用 HTTP Keep-Alive（连接复用）

观察输出：

* 如果 Keep-Alive 生效，输出中会显示 `Connection: Keep-Alive`
* 请求速度、延迟等会比未启用 Keep-Alive 更优

---

#### ✅ 工具三：使用 `wrk` 工具进行高并发模拟（推荐）

```bash
wrk -t4 -c10 -d30s --header "Connection: keep-alive" http://10.211.55.3
```

说明：

* `-t4`：4 个线程
* `-c10`：10 个并发连接
* `-d30s`：持续 30 秒

观察结果中的吞吐量（requests/sec）和连接数变化，若连接数较少且响应快，说明连接被复用。

---

### ✅ 方法四：在 nginx 中启用日志：

```bash
log_format with_conn '$remote_addr:$remote_port conn=$connection reqs=$connection_requests';
```

```
tail -f /var/log/nginx/access.log | grep 10.211.55.2
```

```
<客户端IP>:<客户端端口> conn=<连接编号> reqs=<当前连接内第几个请求>
```

| 变量                     | 含义                                            |
| ---------------------- | --------------------------------------------- |
| `$remote_addr`         | 客户端 IP 地址                                     |
| `$remote_port`         | 客户端端口（每次建立 TCP 连接时由客户端操作系统临时分配）               |
| `$connection`          | **连接编号**，Nginx 进程内分配的标识，同一条 TCP 连接这个值固定       |
| `$connection_requests` | **连接内处理的请求次数**，通常用于观察**HTTP Keep-Alive 复用情况** |


输出示例：

```
10.211.55.2:50774 conn=46 reqs=1
10.211.55.2:50774 conn=46 reqs=2
10.211.55.2:50774 conn=46 reqs=3
```

- 说明这个客户端（10.211.55.2）用端口 50774 建立了一个连接（conn=46）

- 在这个连接上已经复用了 3 次请求




### ✅ 方法五：使用 `ss` 查看连接状态

在压测过程中执行以下命令，实时监控 TCP 连接：

```bash
 watch -d -n 1 "ss -tan | grep 10.211.55.2 | grep :80"
```


如果 Keep-Alive 启用，连接数会相对较少，并保持稳定（不会每个请求都建立新连接）。

---

### ✅ 方法六（可选）：查看 `TIME_WAIT` 数量

连接未复用时，每次断开都进入 `TIME_WAIT` 状态。使用：

```bash
watch -n 1 "netstat -an | grep TIME_WAIT | wc -l"
```

或者观察连接是否在 TIME_WAIT
```
watch -n 1 "ss -tan | grep 10.211.55.2 | awk '{print \$1}' | sort | uniq -c"
```

如果 Keep-Alive 正常工作，`TIME_WAIT` 的数量会显著减少。

---

## ✅ 小结：验证方法对比

| 验证方法                | 是否能验证 keep-alive 开启 | 是否能验证连接复用    |
| ------------------- | ------------------- | ------------ |
| `curl -I`           | ✅ 是（响应头）            | ❌ 否          |
| `curl -v` 多次请求      | ✅ 是                 | ✅ 是（看复用信息）   |
| `ab -k` 压测工具        | ✅ 是                 | ✅ 是（间接）      |
| `wrk` 并发测试          | ✅ 是                 | ✅ 是（间接）      |
| `netstat/ss` 监控连接状态 | ✅ 是                 | ✅ 是（看连接是否重用） |
| `TIME_WAIT` 状态数量观察  | ✅ 是                 | ✅ 是（间接）      |

---

## 六、调优建议

| 项目                   | 建议值           | 说明               |
| -------------------- | ------------- | ---------------- |
| `keepalive_timeout`  | 30 \~ 75 秒    | 过短会频繁断开，过长浪费资源   |
| `keepalive_requests` | 100 \~ 1000   | 太小复用效果差，太大可能连接过久 |
| `upstream keepalive` | 与后端服务并发数成比例设置 | 控制空闲连接数，防止资源占满   |

---

## 七、注意事项

* 如果后端服务（如 Tomcat）不支持 keep-alive，Nginx 设置也无效。
* 连接数受 `worker_connections` 和系统 `ulimit` 限制。
* keep-alive 对于静态资源服务器、大量短请求场景效果最好。

---

## 八、小结

| 场景               | 关键配置                                                       |
| ---------------- | ---------------------------------------------------------- |
| 客户端 → Nginx      | `keepalive_timeout`, `keepalive_requests`                  |
| Nginx → 后端（反向代理） | `upstream keepalive`, `proxy_http_version`, `Connection` 头 |

使用得当的长连接策略可以极大地提升 Nginx 的吞吐量和资源利用率，是高性能 Web 系统中不可忽视的优化方式。

---



