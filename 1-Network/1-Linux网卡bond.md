# Linux网卡Bond

Linux 网卡绑定（bonding）共有 **7 种标准模式**：

| 模式 | 名称 | 原理 | 需要交换机支持 |
|------|------|------|:--:|
| **mode=0** | balance-rr | 轮询，按包交替走不同网卡 | ✅ 需要 |
| **mode=1** | active-backup | 主备，一根断了切另一根 | ❌ 不需要 |
| **mode=2** | balance-xor | 按源/目标 MAC/IP 做 XOR 哈希选路 | ❌ 不需要 |
| **mode=3** | broadcast | 所有包从所有网卡同时发出 | ❌ 不需要 |
| **mode=4** | 802.3ad (LACP) | 动态链路聚合，标准协议 | ✅ 需要 |
| **mode=5** | balance-tlb | 按负载选发送口，接收只走主口 | ❌ 不需要 |
| **mode=6** | balance-alb | TLB + ARP 协商实现收发都负载均衡 | ❌ 不需要 |

我们可以通过如下命令来查看一个bond网卡具体使用的模式:

```bash
# cat /proc/net/bonding/bond1
...
Bonding Mode: IEEE 802.3ad Dynamic link aggregation
```
