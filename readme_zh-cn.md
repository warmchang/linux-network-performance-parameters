[🇷🇺](/README_RU.md "俄语")

# 目录

* [简介](#简介)
* [Linux 网络队列概述](#linux-网络队列概述)
* [将 sysctl 参数融入 Linux 网络处理流程](#将-sysctl-参数融入-linux-网络处理流程)
  * 入站路径 - 数据进来时
  * 出站路径 - 数据出去时
  * 如何观察 - perf
* [网络与 sysctl 参数：是什么、为什么、怎么调](#网络与-sysctl-参数是什么为什么怎么调)
  * 环形缓冲区 - rx, tx
  * 中断合并（IC）- rx-usecs, tx-usecs, rx-frames, tx-frames（硬件 IRQ）
  * 中断合并（软 IRQ / NAPI）与入站 QDisc
  * 出站 QDisc - txqueuelen 和 default_qdisc
  * TCP 读写缓冲区 / 队列
  * 补充：TCP 状态机与拥塞控制算法
* [网络测试与监控工具](#网络测试与监控工具)
* [参考资料](#参考资料)

# 简介

有时候，人们会去寻找一组可以“照抄即用”的 [sysctl](https://www.kernel.org/doc/Documentation/networking/ip-sysctl.txt) 参数，希望它在任何场景下都能同时带来高吞吐、低延迟，而且没有任何权衡。这种想法并不现实。较新的内核版本通常已经有相当不错的默认调优；事实上，随意改动默认值反而可能[损害性能](https://medium.com/@duhroach/the-bandwidth-delay-problem-c6a2a578b211)。

这篇简短教程的目标，是把**一些最常见、也最常被提到的 sysctl / 网络参数放回到 Linux 网络路径中去理解**。内容主要受 [Linux 网络栈图解指南](https://blog.packagecloud.io/eng/2016/10/11/monitoring-tuning-linux-networking-stack-receiving-data-illustrated/) 和 [Marek Majkowski 的多篇文章](https://blog.cloudflare.com/how-to-achieve-low-latency/) 启发。

> #### 欢迎提出更正和建议！ :)

# Linux 网络队列概述

<img width="563" height="339" alt="linux network flow" src="https://github.com/user-attachments/assets/59e65d53-2175-40c9-8805-44ccd3b3daed" />


# 将 sysctl 参数融入 Linux 网络处理流程

## 入站路径 - 数据进来时
1. 数据包到达网卡（NIC）。
2. NIC 会校验 `MAC` 地址（如果没有开启混杂模式）和 `FCS`，然后决定丢弃还是继续处理。
3. NIC 会通过 [DMA](https://en.wikipedia.org/wiki/Direct_memory_access) 把数据包搬运到 RAM 中，而这块 DMA 映射区域事先由驱动准备好。
4. NIC 会把这些数据包的引用加入接收[环形缓冲区](https://en.wikipedia.org/wiki/Circular_buffer)队列 `rx`，直到 `rx-usecs` 超时或达到 `rx-frames` 的阈值。
5. NIC 触发一次`硬中断`（hard IRQ）。
6. CPU 运行`IRQ 处理程序`，执行驱动中的中断处理代码。
7. 驱动会`调度 NAPI`，清除`硬中断`，然后返回。
8. 驱动触发`软中断（NET_RX_SOFTIRQ）`。
9. NAPI 开始从接收环形缓冲区轮询数据，直到 `netdev_budget_usecs` 超时，或处理的数据包数量达到 `netdev_budget` / `dev_weight` 的限制。
10. Linux 还会为 `sk_buff`（socket buffer，套接字缓冲区）分配内存。
11. Linux 填充相关元数据：协议、接口信息、设置 MAC 头并剥离以太网头。
12. Linux 把 `skb` 交给内核协议栈（`netif_receive_skb`）。
13. 内核设置网络层头部，把 `skb` 复制给 tap 监听点（例如 `tcpdump`），并交给 tc ingress。
14. 数据包进入入站队列规则（ingress QDisc），排队上限由 `netdev_max_backlog` 控制，队列规则算法由 `default_qdisc` 定义。
15. 内核调用 `ip_rcv`，把数据包交给 IP 层。
16. 内核调用 netfilter（`PREROUTING`）。
17. 内核查看路由表，决定是转发还是本地处理。
18. 如果是本地处理，就继续调用 netfilter（`LOCAL_IN`）。
19. 内核调用四层协议处理函数（例如 `tcp_v4_rcv`）。
20. 内核找到对应的套接字。
21. 数据包进入 TCP 有限状态机。
22. 数据包进入接收缓冲区，其大小受 `tcp_rmem` 规则控制。
    1. 如果启用了 `tcp_moderate_rcvbuf`，内核会自动调优接收缓冲区大小。
23. 内核通知应用程序有数据可读（例如通过 epoll 或其他轮询机制）。
24. 应用程序被唤醒并读取数据。

## 出站路径 - 数据出去时
1. 应用程序发送消息（`sendmsg` 或其他接口）。
2. TCP 为待发送消息分配 `sk_buff`（socket buffer，套接字缓冲区）。
3. `skb` 被放入套接字发送缓冲区，其大小由 `tcp_wmem` 控制。
4. 构造 TCP 头部（源端口、目标端口、校验和等）。
5. 调用三层处理逻辑（这里以 `ipv4` 路径中的 `tcp_write_xmit` 和 `tcp_transmit_skb` 为例）。
6. 三层函数 `ip_queue_xmit` 继续处理：构造 IP 头，并调用 netfilter（`LOCAL_OUT`）。
7. 执行输出路由决策。
8. 调用 netfilter（`POST_ROUTING`）。
9. 对数据包执行分片（`ip_output`）。
10. 调用二层发送函数（`dev_queue_xmit`）。
11. 按 `default_qdisc` 指定的算法，把数据包送入出站 QDisc 队列；该队列长度由 `txqueuelen` 控制。
12. 驱动把数据包加入发送`环形缓冲区 tx`。
13. 驱动会在 `tx-usecs` 超时或达到 `tx-frames` 阈值后触发`软中断（NET_TX_SOFTIRQ）`。
14. 重新启用 NIC 的硬中断。
15. 驱动把所有待发送数据包映射到 DMA 区域。
16. NIC 通过 DMA 从 RAM 取走数据包并发送出去。
17. 发送完成后，NIC 触发一次`硬中断`通知传输结束。
18. 驱动处理这次中断（并将其关闭）。
19. 然后驱动继续调度 NAPI 轮询系统（通过`软中断`）。
20. NAPI 处理接收完成通知并释放相关 RAM。

## 如何观察 - perf

如果你想观察 Linux 内部的网络跟踪事件，可以使用 [perf](https://man7.org/linux/man-pages/man1/perf-trace.1.html)。

```bash
docker run -it --rm --cap-add SYS_ADMIN --entrypoint bash ljishen/perf
apt-get update
apt-get install iputils-ping

# 在执行 ping 的同时，跟踪 net:* 子系统下的所有事件（不包含系统调用）
perf trace --no-syscalls --event 'net:*' ping globo.com -c1 > /dev/null
```
![perf trace network](https://user-images.githubusercontent.com/55913/147019725-69624e67-b3ca-48b4-a823-10521d2bed83.png)


# 网络与 sysctl 参数：是什么、为什么、怎么调

## 环形缓冲区 - rx, tx
* **是什么** - 驱动层的收发队列，可以是一个或多个固定大小的队列，通常按 FIFO（先进先出）方式实现，位于 RAM 中。
* **为什么** - 这类缓冲区可以在突发流量到来时平滑接收数据，减少丢包。如果你观察到丢包或 overrun，说明数据包到达速度已经超过内核的处理能力，这时可能需要增大队列；代价通常是更高的延迟。
* **怎么做：**
  * **检查命令：** `ethtool -g ethX`
  * **修改命令：** `ethtool -G ethX rx value tx value`
  * **如何监控：** `ethtool -S ethX | grep -e "err" -e "drop" -e "over" -e "miss" -e "timeout" -e "reset" -e "restar" -e "collis" -e "over" | grep -v "\: 0"`

## 中断合并（IC）- rx-usecs, tx-usecs, rx-frames, tx-frames（硬件 IRQ）
* **是什么** - 在触发一次硬中断之前，NIC 需要等待的微秒数或帧数。从网卡视角看，就是在达到这个时间或帧数阈值之前持续 DMA 数据包。
* **为什么** - 可以减少 CPU 占用和硬中断次数，通常有助于提升吞吐，但代价可能是更高的延迟。
* **怎么做：**
  * **检查命令：** `ethtool -c ethX`
  * **修改命令：** `ethtool -C ethX rx-usecs value tx-usecs value`
  * **如何监控：** `cat /proc/interrupts`

## 中断合并（软 IRQ / NAPI）与入站 QDisc
* **是什么** - 这里讨论的是软中断侧的“中断合并”行为：`netdev_budget_usecs` 表示一次 [NAPI](https://en.wikipedia.org/wiki/New_API) 轮询周期允许消耗的最大时间（微秒）。当轮询周期已经持续到 `netdev_budget_usecs`，或者处理的数据包数量达到 `netdev_budget` 时，本轮轮询就会退出。
* **为什么** - 驱动通过持续轮询来代替频繁响应大量软中断。重点关注 `dropped`（因为超过 `netdev_max_backlog` 而被丢弃的数据包数）和 `squeezed`（ksoftirq 在还有任务未处理时就耗尽 `netdev_budget` 或时间片的次数）。
* **怎么做：**
  * **检查命令：** `sysctl net.core.netdev_budget_usecs`
  * **修改命令：** `sysctl -w net.core.netdev_budget_usecs value`
  * **如何监控：** `cat /proc/net/softnet_stat`；或者使用[更好的工具](https://raw.githubusercontent.com/majek/dump/master/how-to-receive-a-packet/softnet.sh)
* **是什么** - `netdev_budget` 表示一次轮询周期（NAPI poll）里，从所有接口合计最多处理多少个数据包。一个轮询周期内，已注册到轮询机制的接口会按轮询方式逐个探测。同时，即使 `netdev_budget` 还没耗尽，单次轮询也不能超过 `netdev_budget_usecs` 微秒。
* **怎么做：**
  * **检查命令：** `sysctl net.core.netdev_budget`
  * **修改命令：** `sysctl -w net.core.netdev_budget value`
  * **如何监控：** `cat /proc/net/softnet_stat`；或者使用[更好的工具](https://raw.githubusercontent.com/majek/dump/master/how-to-receive-a-packet/softnet.sh)
* **是什么** - `dev_weight` 是内核在一次 NAPI 中断处理中最多能处理的数据包数量，它是按 CPU 维度生效的变量。对于支持 LRO 或 GRO_HW 的驱动，硬件聚合后的数据包在这里按一个数据包计算。
* **怎么做：**
  * **检查命令：** `sysctl net.core.dev_weight`
  * **修改命令：** `sysctl -w net.core.dev_weight value`
  * **如何监控：** `cat /proc/net/softnet_stat`；或者使用[更好的工具](https://raw.githubusercontent.com/majek/dump/master/how-to-receive-a-packet/softnet.sh)
* **是什么** - `netdev_max_backlog` 是接口接收数据包速度高于内核处理速度时，允许在输入侧（也就是入站 QDisc）排队的最大数据包数。
* **怎么做：**
  * **检查命令：** `sysctl net.core.netdev_max_backlog`
  * **修改命令：** `sysctl -w net.core.netdev_max_backlog value`
  * **如何监控：** `cat /proc/net/softnet_stat`；或者使用[更好的工具](https://raw.githubusercontent.com/majek/dump/master/how-to-receive-a-packet/softnet.sh)

## 出站 QDisc - txqueuelen 和 default_qdisc
* **是什么** - `txqueuelen` 是输出侧允许排队的最大数据包数。
* **为什么** - 这是应对突发流量的一个缓冲 / 队列，同时也是应用 [tc（流量控制）](http://tldp.org/HOWTO/Traffic-Control-HOWTO/intro.html) 的基础。
* **怎么做：**
  * **检查命令：** `ip link show dev ethX`
  * **修改命令：** `ip link set dev ethX txqueuelen N`
  * **如何监控：** `ip -s link`
* **是什么** - `default_qdisc` 是网络设备默认使用的队列规则（queuing discipline）。
* **为什么** - 不同应用负载对流量控制的需求不同，它也常用于缓解[缓冲区膨胀（bufferbloat）](https://www.bufferbloat.net/projects/codel/wiki/)。
* **怎么做：**
  * **检查命令：** `sysctl net.core.default_qdisc`
  * **修改命令：** `sysctl -w net.core.default_qdisc value`
  * **如何监控：** `tc -s qdisc ls dev ethX`

## TCP 读写缓冲区 / 队列

> 用于定义什么叫作[内存压力](https://web.archive.org/web/20200315112330/wwwx.cs.unc.edu/~sparkst/howto/network_tuning.php)的策略，由 `tcp_mem` 和 `tcp_moderate_rcvbuf` 共同指定。

* **是什么** - `tcp_rmem` 定义 TCP 套接字接收缓冲区的最小值（内存压力下使用的大小）、默认值（初始大小）和最大值（允许增长到的上限）。
* **为什么** - 它直接影响 TCP 接收侧的缓存能力；[理解这类缓冲区参数带来的后果会很有帮助](https://blog.cloudflare.com/the-story-of-one-latency-spike/)，尤其是在分析延迟尖峰、吞吐和内存占用之间的权衡时。
* **怎么做：**
  * **检查命令：** `sysctl net.ipv4.tcp_rmem`
  * **修改命令：** `sysctl -w net.ipv4.tcp_rmem="min default max"`；修改默认值时，记得重启用户空间应用（例如 Web 服务器、nginx 等）
  * **如何监控：** `cat /proc/net/sockstat`
* **是什么** - `tcp_wmem` 定义 TCP 套接字发送缓冲区的最小值（内存压力下使用的大小）、默认值（初始大小）和最大值（允许增长到的上限）。
* **怎么做：**
  * **检查命令：** `sysctl net.ipv4.tcp_wmem`
  * **修改命令：** `sysctl -w net.ipv4.tcp_wmem="min default max"`；修改默认值时，记得重启用户空间应用（例如 Web 服务器、nginx 等）
  * **如何监控：** `cat /proc/net/sockstat`
* **是什么** - `tcp_moderate_rcvbuf` 用于控制 TCP 接收缓冲区自动调优；开启后，TCP 会尝试自动调整缓冲区大小。
* **怎么做：**
  * **检查命令：** `sysctl net.ipv4.tcp_moderate_rcvbuf`
  * **修改命令：** `sysctl -w net.ipv4.tcp_moderate_rcvbuf value`
  * **如何监控：** `cat /proc/net/sockstat`

## 补充：TCP 状态机与拥塞控制算法

> Accept 队列和 SYN 队列由 `net.core.somaxconn` 和 `net.ipv4.tcp_max_syn_backlog` 管理。[现在 `net.core.somaxconn` 会同时限制这两个队列的大小。](https://blog.cloudflare.com/syn-packet-handling-in-the-wild/#queuesizelimits)

* `sysctl net.core.somaxconn` - 为传给 [`listen()` 函数](https://eklitzke.org/how-tcp-sockets-work) 的 backlog 参数设置上限；它在用户空间里对应 `SOMAXCONN`。如果你修改这个值，也应该把应用配置改到兼容范围内（例如 [nginx backlog](http://nginx.org/en/docs/http/ngx_http_core_module.html#listen)）。
* `cat /proc/sys/net/ipv4/tcp_fin_timeout` - 指定在套接字被强制关闭前，等待最后一个 FIN 报文的秒数。这严格来说违反了 TCP 规范，但对于防止拒绝服务攻击是必要的。
* `cat /proc/sys/net/ipv4/tcp_available_congestion_control` - 显示当前已注册、可用的拥塞控制算法。
* `cat /proc/sys/net/ipv4/tcp_congestion_control` - 设置新连接默认使用的拥塞控制算法。
* `cat /proc/sys/net/ipv4/tcp_max_syn_backlog` - 设置尚未收到客户端确认的连接请求队列上限；超过这个数量后，内核会开始丢弃请求。
* `cat /proc/sys/net/ipv4/tcp_syncookies` - 启用 / 禁用 [SYN Cookie](https://en.wikipedia.org/wiki/SYN_cookies)，可用于防御 [SYN Flood 攻击](https://www.cloudflare.com/learning/ddos/syn-flood-ddos-attack/)。
* `cat /proc/sys/net/ipv4/tcp_slow_start_after_idle` - 启用 / 禁用 TCP 慢启动。

**如何监控：**
* `netstat -atn | awk '/tcp/ {print $6}' | sort | uniq -c` - 按状态汇总
* `ss -neopt state time-wait | wc -l` - 统计特定状态的数量：`established`、`syn-sent`、`syn-recv`、`fin-wait-1`、`fin-wait-2`、`time-wait`、`closed`、`close-wait`、`last-ack`、`listening`、`closing`
* `netstat -st` - TCP 统计摘要
* `nstat -a` - 更易读的 TCP 统计摘要
* `cat /proc/net/sockstat` - 汇总后的套接字统计
* `cat /proc/net/tcp` - 详细统计；各字段含义可参考[内核文档](https://www.kernel.org/doc/Documentation/networking/proc_net_tcp.txt)
* `cat /proc/net/netstat` - 重点关注 `ListenOverflows` 和 `ListenDrops`
  * `cat /proc/net/netstat | awk '(f==0) { i=1; while (i<=NF) {n[i] = $i; i++ }; f=1; next} \
(f==1) { i=2; while (i<=NF) { printf "%s = %d\n", n[i], $i; i++ }; f=0 }' | grep -v "= 0"`；也可以参考这份[更易读的 `/proc/net/netstat` 说明](https://sa-chernomor.livejournal.com/9858.html)

![tcp finite state machine](https://upload.wikimedia.org/wikipedia/commons/a/a2/Tcp_state_diagram_fixed.svg "TCP 有限状态机图示")
来源：https://commons.wikimedia.org/wiki/File:Tcp_state_diagram_fixed_new.svg

# 网络测试与监控工具

* [iperf3](https://iperf.fr/) - 网络吞吐测试
* [vegeta](https://github.com/tsenart/vegeta) - HTTP 压测工具
* [netdata](https://github.com/firehol/netdata) - 分布式实时性能与健康监控系统
* [prometheus](https://prometheus.io/) + [grafana](https://grafana.com/) + [node exporter 完整仪表板](https://grafana.com/grafana/dashboards/1860-node-exporter-full/) - 用于绘制详细系统行为图表的监控栈

# 参考资料

* https://www.kernel.org/doc/Documentation/sysctl/net.txt
* https://www.kernel.org/doc/Documentation/networking/ip-sysctl.txt
* https://www.kernel.org/doc/Documentation/networking/scaling.txt
* https://www.kernel.org/doc/Documentation/networking/proc_net_tcp.txt
* https://www.kernel.org/doc/Documentation/networking/multiqueue.txt
* http://man7.org/linux/man-pages/man7/tcp.7.html
* http://man7.org/linux/man-pages/man8/tc.8.html
* http://cseweb.ucsd.edu/classes/fa09/cse124/presentations/TCPlinux_implementation.pdf
* https://netdevconf.org/1.2/papers/bbr-netdev-1.2.new.new.pdf
* https://blog.cloudflare.com/how-to-receive-a-million-packets/
* https://blog.cloudflare.com/how-to-achieve-low-latency/
* https://people.redhat.com/pladd/MHVLUG_2017-04_Network_Receive_Stack.pdf
* https://blog.packagecloud.io/eng/2016/06/22/monitoring-tuning-linux-networking-stack-receiving-data/
* https://www.youtube.com/watch?v=6Fl1rsxk4JQ
* https://oxnz.github.io/2016/05/03/performance-tuning-networking/
* https://www.intel.com/content/dam/www/public/us/en/documents/reference-guides/xl710-x710-performance-tuning-linux-guide.pdf
* https://access.redhat.com/sites/default/files/attachments/20150325_network_performance_tuning.pdf
* https://medium.com/@matteocroce/linux-and-freebsd-networking-cbadcdb15ddd
* https://blogs.technet.microsoft.com/networking/2009/08/12/where-do-resets-come-from-no-the-stork-does-not-bring-them/
* https://www.intel.com/content/dam/www/public/us/en/documents/white-papers/multi-core-processor-based-linux-paper.pdf
* http://syuu.dokukino.com/2013/05/linux-kernel-features-for-high-speed.html
* https://www.bufferbloat.net/projects/codel/wiki/Best_practices_for_benchmarking_Codel_and_FQ_Codel/
* https://software.intel.com/en-us/articles/setting-up-intel-ethernet-flow-director
* https://courses.engr.illinois.edu/cs423/sp2014/Lectures/LinuxDriver.pdf
* https://www.coverfire.com/articles/queueing-in-the-linux-network-stack/
* http://vger.kernel.org/~davem/skb.html
* https://www.missoulapubliclibrary.org/ftp/LinuxJournal/LJ13-07.pdf
* https://opensourceforu.com/2016/10/network-performance-monitoring/
* https://www.yumpu.com/en/document/view/55400902/an-adventure-of-analysis-and-optimisation-of-the-linux-networking-stack
* https://lwn.net/Articles/616241/
* https://medium.com/@duhroach/tools-to-profile-networking-performance-3141870d5233
* https://www.lmax.com/blog/staff-blogs/2016/05/06/navigating-linux-kernel-network-stack-receive-path/
* https://fasterdata.es.net/host-tuning/linux/100g-tuning/
* http://tcpipguide.com/free/t_TCPOperationalOverviewandtheTCPFiniteStateMachineF-2.htm
* http://veithen.github.io/2014/01/01/how-tcp-backlog-works-in-linux.html
* https://people.cs.clemson.edu/~westall/853/tcpperf.pdf
* http://tldp.org/HOWTO/Traffic-Control-HOWTO/classless-qdiscs.html
* https://fasterdata.es.net/assets/Papers-and-Publications/100G-Tuning-TechEx2016.tierney.pdf
* https://www.kernel.org/doc/ols/2009/ols2009-pages-169-184.pdf
* https://devcentral.f5.com/articles/the-send-buffer-in-depth-21845
* http://packetbomb.com/understanding-throughput-and-tcp-windows/
* https://www.speedguide.net/bdp.php
* https://www.switch.ch/network/tools/tcp_throughput/
* https://www.ibm.com/support/knowledgecenter/en/SSQPD3_2.6.0/com.ibm.wllm.doc/usingethtoolrates.html
* https://blog.tsunanet.net/2011/03/out-of-socket-memory.html
* https://unix.stackexchange.com/questions/12985/how-to-check-rx-ring-max-backlog-and-max-syn-backlog-size
* https://serverfault.com/questions/498245/how-to-reduce-number-of-time-wait-processes
* https://unix.stackexchange.com/questions/419518/how-to-tell-how-much-memory-tcp-buffers-are-actually-using
* https://eklitzke.org/how-tcp-sockets-work
* https://www.linux.com/learn/intro-to-linux/2017/7/introduction-ss-command
* https://staaldraad.github.io/2017/12/20/netstat-without-netstat/
* https://loicpefferkorn.net/2016/03/linux-network-metrics-why-you-should-use-nstat-instead-of-netstat/
* http://assimilationsystems.com/2015/12/29/bufferbloat-network-best-practice/
* https://wwwx.cs.unc.edu/~sparkst/howto/network_tuning.php
* https://medium.com/@tom_84912/the-alphabet-soup-of-receive-packet-steering-rss-rps-rfs-and-arfs-c84347156d68
