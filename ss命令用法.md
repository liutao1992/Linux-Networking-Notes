以下是一个\*\*`ss` 命令教程\*\*，面向日常运维与开发中的常见需求。适合你查端口、分析连接状态、排查网络问题等用途。

---

# 🧪 `ss` 命令教程：Linux 网络连接查看利器

## 一、为什么用 `ss`

* `ss` 是 `netstat` 的升级替代品
* 启动快，信息详细
* 能查看 TCP、UDP、UNIX 等连接
* 能显示进程、连接状态、端口占用

---

## 二、基本语法

```bash
ss [参数]
```

---

## 三、常见用法场景

### ✅ 1. 查看所有 TCP 连接

```bash
ss -t
```

### ✅ 2. 查看所有 UDP 连接

```bash
ss -u
```

### ✅ 3. 查看所有监听中的端口（TCP + UDP）

```bash
ss -tuln
```

* `-t`: TCP
* `-u`: UDP
* `-l`: LISTEN 状态（监听端口）
* `-n`: 不解析端口名，直接显示数字（如 80 而不是 http）

**示例输出**：

```
Netid State  Local Address:Port  Peer Address:Port
tcp   LISTEN 0.0.0.0:22           0.0.0.0:*
tcp   LISTEN 0.0.0.0:80           0.0.0.0:*
```

---

### ✅ 4. 查看监听端口及其所属进程（需 sudo）

```bash
sudo ss -tulnp
```

* `-p`: 显示进程名和 PID

**示例输出**：

```
tcp  LISTEN 0.0.0.0:80  0.0.0.0:*  users:(("nginx",pid=1234,fd=6))
```

---

### ✅ 5. 查看某个端口是否被占用

比如查看 8080 是否被监听：

```bash
ss -tuln | grep :8080
```

---

### ✅ 6. 查看某个状态的连接（如 TIME\_WAIT）

```bash
ss -o state time-wait
```

其他常见状态还有：

| 状态            | 含义          |
| ------------- | ----------- |
| `established` | 已建立连接       |
| `listen`      | 正在监听        |
| `time-wait`   | 等待超时关闭的连接   |
| `syn-recv`    | 正在接收连接（半连接） |

---

### ✅ 7. 显示连接统计信息（快速概览）

```bash
ss -s
```

输出示例：

```
Total: 150
TCP:   20 (estab 15, closed 2, orphaned 0, timewait 3)
```

---

### ✅ 8. 查看指定程序的网络连接（如 nginx）

```bash
sudo ss -tulpn | grep nginx
```

---

### ✅ 9. 查看来自某个 IP：80 的连接状态

```bash
watch -d -n 1 "ss -tan | grep 10.211.55.2 | grep :80"
```

---

## 四、进阶用法：按条件筛选

### 🌐 过滤本地或远程 IP 和端口

```bash
ss -tan sport = :443        # 源端口为 443
ss -tan dport = :8080       # 目的端口为 8080
ss -tan dst 192.168.1.10    # 目的 IP 为 192.168.1.10
ss -tan src 127.0.0.1       # 源 IP 为 127.0.0.1
```

---

## 五、快捷组合命令推荐

| 场景                  | 命令                      |              |
| ------------------- | ----------------------- | ------------ |
| 查看所有监听端口 + 进程       | `sudo ss -tulnp`        |              |
| 查看 TIME\_WAIT 连接    | `ss -o state time-wait` |              |
| 查看某端口是否被占用          | \`ss -tuln              | grep :端口号\`  |
| 统计连接状态              | `ss -s`                 |              |
| 查某个进程的网络连接（如 nginx） | \`sudo ss -tulpn        | grep nginx\` |

---

## 六、总结

| 参数   | 作用           |
| ---- | ------------ |
| `-t` | TCP          |
| `-u` | UDP          |
| `-l` | 监听端口（LISTEN） |
| `-n` | 不解析主机名和端口名   |
| `-p` | 显示使用该连接的进程   |
| `-a` | 显示所有状态（含监听）  |
| `-s` | 显示统计信息       |
| `-o` | 显示 TCP 计时器信息 |

---

太好了，这个正好是 `ss` 的强项！我们分两种情况详细讲解一下：

---

# 🎯 场景一：调试 HTTP Keep-Alive（连接复用）

### 💡 目标

* 查看客户端是否复用了连接
* 查看同一个客户端 IP 是否通过同一连接发送多个请求

### 🔧 操作方法

#### ✅ 1. 查看所有已建立的连接

```bash
ss -tn state established
```

#### ✅ 2. 过滤目标端口（如 nginx 监听的 80 或 8080 端口）

```bash
ss -tn state established '( sport = :80 or sport = :8080 )'
```

#### ✅ 3. 查看连接中是否有多个来自同一 IP

```bash
ss -tn state established | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -nr
```

> 这会统计每个客户端 IP 建立了多少个连接。
> 如果 Keep-Alive 生效，IP 数少、连接数少。

#### ✅ 4. 查看某 IP 是否复用了连接

```bash
ss -tnp state established | grep <客户端IP>
```

---

### 📌 附加技巧

你也可以从 nginx 日志中看到连接是否复用：

* nginx 中可以加日志变量 `$connection`, `$connection_requests`：

```nginx
log_format with_conn '$remote_addr:$remote_port '
                    'conn=$connection reqs=$connection_requests';
```

会显示连接 ID 和请求数，如：

```
192.168.0.5:56732 conn=205 reqs=3
```

表示连接 205 已复用 3 次（典型的 keep-alive）。

---

# 🎯 场景二：查看 TIME\_WAIT 是否异常增多（连接泄露/端口耗尽）

### 💡 目标

* TIME\_WAIT 数量是否异常
* 哪些连接产生了大量 TIME\_WAIT
* 是否连接未正确复用或提前关闭

### 🔧 操作方法

#### ✅ 1. 查看 TIME\_WAIT 连接总数

```bash
ss -s
```

输出示例：

```
TCP: 105 (estab 10, closed 2, orphaned 0, timewait 93)
```

#### ✅ 2. 只看 TIME\_WAIT 的连接

```bash
ss -tan state time-wait
```

#### ✅ 3. 统计哪些远程 IP 导致 TIME\_WAIT

```bash
ss -tan state time-wait | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -nr
```

---

### 🧠 进阶分析建议

如果 TIME\_WAIT 非常多，可能是：

| 原因                           | 解释                       |
| ---------------------------- | ------------------------ |
| HTTP 未开启 keep-alive          | 每次请求都断开，造成 TIME\_WAIT 爆炸 |
| 客户端主动关闭连接                    | 常见于压力测试或爬虫               |
| nginx `keepalive_timeout` 太短 | 连接复用不充分                  |
| 上游服务连接池小，频繁断开                | 导致下游反复重连                 |

---

### 🧰 实用建议

| 动作                  | 建议配置                                            |
| ------------------- | ----------------------------------------------- |
| nginx 开启 keep-alive | `keepalive_timeout 60; keepalive_requests 100;` |
| 反向代理上游连接            | `upstream` 中加 `keepalive 32;`                   |
| 避免 TIME\_WAIT 过多    | 调整客户端或服务端连接关闭策略                                 |
| 临时诊断                | 使用 `ss` + `tcpdump` 联合分析                        |

---



