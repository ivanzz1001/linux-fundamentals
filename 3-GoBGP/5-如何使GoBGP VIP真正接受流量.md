# 如何使GoBGP VIP真正接收流量

要让 VIP 真正接收流量，需要同时完成三件事：

1. Linux 将 VIP 视为本机地址。
1. 应用监听 VIP 对应端口。
1. GoBGP 把 VIP 路由发布给上游。

下面以推荐的“VIP 绑定 Loopback”方式说明。

## 1. 示例拓扑

```text
服务器物理地址：192.0.2.10/24，eth0
上游 ToR：      192.0.2.1
VIP：           10.20.0.100/32
业务端口：      TCP 443
```

最终流量路径：

```text
客户端
  → 上游路由器
  → 查到 10.20.0.100/32 经由 192.0.2.10
  → 数据包从服务器 eth0 进入
  → Linux 发现 10.20.0.100 是本地地址
  → 投递给监听 10.20.0.100:443 的应用
```

## 2. 方式1-把 VIP 配置到 Loopback（推荐）

### 2.1 添加VIP

```bash
# sudo ip addr add 10.20.0.100/32 dev lo
```

检查：
```bash
# ip addr show dev lo
# ip route show table local
```
应该看到类似:

```bash
local 10.20.0.100 dev lo proto kernel scope host
```
`local路由`表示目的地址属于本机，数据包会被交给本地协议栈，而不是继续转发。
> 参看 [Linux ip-route 文档](https://man7.org/linux/man-pages/man8/ip-route.8.html)

这里不需要打开:

```bash
net.ipv4.ip_forward
```
因为数据包的目的地址是本机 VIP，不是让服务器充当普通路由器。

### 2.2 让应用监听 VIP

- 应用可以只监听 VIP

  ```bash
  10.20.0.100:443
  ```
  例如`Nginx`:
  
  ```bash
  server {
      listen 10.20.0.100:443 ssl;
      server_name _;
  }
  ```
  检查监听情况应该可以看到:

  ```bash
  # ss -lntp
  LISTEN 0 511 10.20.0.100:443
  ```


- 也可以监听所有本地地址

  ```bash
  0.0.0.0:443
  ```
   检查监听情况应该可以看到:

  ```bash
  # ss -lntp
  LISTEN 0 511 0.0.0.0:443
  ```

先在服务器本机测试：

```bash
# curl -k https://10.20.0.100/
```
如果本机都不能访问，先不要发布 BGP 路由。

## 2.2 通过 GoBGP 发布 VIP

如果服务器与 ToR 建立的是直连 eBGP，通常可以直接添加：

```bash
gobgp global rib add 10.20.0.100/32
```


