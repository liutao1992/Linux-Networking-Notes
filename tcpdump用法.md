`tcpdump` 是一个强大的命令行工具，用于截获和分析网络数据包。它广泛应用于网络故障排除、性能分析和安全检测等方面。

### tcpdump表达式

![这是图片](./image/tcpdump表达式.png "tcp表达式")


### 1. 基本抓包命令

抓取所有流经 `eth0` 接口的数据包：

```shell
sudo tcpdump -i eth0
```

### 2. 抓取指定数量的数据包

抓取10个数据包：

```shell
sudo tcpdump -i eth0 -c 10
```

### 3. 抓取并保存到文件

将抓取的数据包保存到 `capture.pcap` 文件中：

```shell
sudo tcpdump -i eth0 -w capture.pcap
```

### 4. 从文件读取数据包

从文件 `capture.pcap` 中读取并显示数据包：

```shell
sudo tcpdump -r capture.pcap
```

### 5. 抓取指定主机的数据包

#### 5.1. 抓取与 `192.168.1.1` 相关的数据包：

```shell
sudo tcpdump -i eth0 host 192.168.1.1

等价于：sudo tcpdump -i eth0 src host 192.168.1.1 or dst host 192.168.1.1
```

> 含义：

- 抓取所有 源地址或目标地址为 192.168.1.1 的数据包

#### 5.2. 抓取从 `192.168.1.1` 发出的数据包：

```shell
sudo tcpdump -i eth0 src host 192.168.1.1
```

> 含义：
- 抓取源地址（Source Address）为 192.168.1.1 的数据包。

- 即这些数据包是“从 192.168.1.1 发出来”的。

> 应用场景：

想看 192.168.1.1 这台设备正在发送什么数据。

#### 5.4. 抓取发送到 `192.168.1.1` 的数据包：

```shell
sudo tcpdump -i eth0 dst host 192.168.1.1
```

> 含义：

- 抓取目标地址（Destination Address）为 192.168.1.1 的数据包。

- 即这些数据包是“发往 192.168.1.1”的。

> 应用场景：

想看哪些设备/主机正在向 192.168.1.1 发数据。


> 举例理解：

假设你有两台主机 A（192.168.1.2）和 B（192.168.1.1），你在 A 上运行 `tcpdump`：

- 抓取发往 B 的包：

```bash
sudo tcpdump -i eth0 dst host 192.168.1.1
```

捕捉的是 A 向 B 发的数据。

- 抓取来自 B 的包：

```bash
sudo tcpdump -i eth0 src host 192.168.1.1
```

捕捉的是 B 发给 A 的数据。

- 如果你想抓 双向通信：

```bash
sudo tcpdump -i eth0 host 192.168.1.1 

或者

sudo tcpdump -i eth0 src host 192.168.1.1 or dst host 192.168
```

### 6. 抓取指定端口的数据包

抓取与端口 80（HTTP）相关的数据包：

```shell
sudo tcpdump -i eth0 port 80
```

抓取源端口为 80 的数据包：

```shell
sudo tcpdump -i eth0 src port 80
```

抓取目标端口为 80 的数据包：

```shell
sudo tcpdump -i eth0 dst port 80
```

### 7. 抓取指定协议的数据包

抓取所有 TCP 数据包：

```shell
sudo tcpdump -i eth0 tcp
```

抓取所有 UDP 数据包：

```shell
sudo tcpdump -i eth0 udp
```

抓取所有 ICMP 数据包：

```shell
sudo tcpdump -i eth0 icmp
```

### 8. 抓取并显示数据包内容

显示抓取数据包的完整内容：

```shell
sudo tcpdump -i eth0 -A
```

显示抓取数据包的十六进制和ASCII内容：

```shell
sudo tcpdump -i eth0 -X
```

### 9. 使用表达式进行复杂过滤

抓取从 `192.168.1.1` 发出的且目标端口为 80 的 TCP 数据包：

```shell
sudo tcpdump -i eth0 src host 192.168.1.1 and tcp dst port 80
```

### 10. 只抓取ARP数据包

抓取所有 ARP 数据包：

```shell
sudo tcpdump -i eth0 arp
```

### 11. 抓取特定时间段的数据包

设置抓包持续时间为 30 秒：

```shell
sudo timeout 30 tcpdump -i eth0
```

### 12. 抓取 VLAN 数据包

抓取 VLAN ID 为 10 的数据包：

```shell
sudo tcpdump -i eth0 vlan 10
```

### 13. 两个指定IP地址之间传输的数据包

``` bash
sudo tcpdump -i eth0-ent'(dst 192.168.1.109 and src 192.168.1.108)or
(dst 192.168.1.108 and src 192.168.1.109)'
```

> 含义：

抓取在网卡 eth0 上，192.168.1.108 和 192.168.1.109 两台主机之间的所有双向通信数据包，并显示链路层信息、纯 IP、无时间戳。即抓取这两台主机之间的所有数据包（双向）

### 14. 抓两主机间的 HTTP 流量（端口 80）：

``` bash
sudo tcpdump -i eth0 -ent '((src 192.168.1.108 and dst 192.168.1.109) or (src 192.168.1.109 and dst 192.168.1.108)) and tcp port 80'
```

### 示例总结

`tcpdump` 的强大功能使其成为网络分析的重要工具。通过灵活使用各种过滤条件和选项，用户可以精确地截获并分析特定类型的数据包，帮助快速定位网络问题并进行深入的网络研究和分析。




