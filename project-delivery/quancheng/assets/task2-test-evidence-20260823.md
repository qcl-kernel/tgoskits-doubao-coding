# 任务二测试证据摘录（2026-08-23 UTC）

## 1. 固定环境

```text
implementation commit: 2a891bbe949f59c7bfd037107b3615de4f57c704
host: Linux 4.19.90-89.11.v2401.ky10 x86_64
QEMU: 10.0.11
rustc/cargo: 1.99.0-nightly
cc: GCC 14.2.0
Python: 3.13.5
topology: Axvisor + StarryOS VM1 + ArceOS VM2
network: 10.0.42.1/24 <-> 10.0.42.2/24, TCP 4242
```

## 2. 构建、静态质量与底座测试

| 检查项 | 结果 |
| --- | --- |
| `cargo test -p guest-ip-protocol` | 5/5 PASS |
| `cargo test -p axvirtio-net` | 32/32 PASS |
| `cargo test -p axvirtio-blk` | 38/38 PASS |
| guest-ip-protocol Clippy | PASS，0 warning |
| arceos-guest-ip-server check/Clippy | PASS，0 warning |
| StarryOS C client `-Werror` 编译 | PASS |
| Python 验证脚本语法检查 | PASS |

## 3. StarryOS/ArceOS 双 guest 端到端结果

端到端运行的关键 marker 如下：

```text
VM[1] boot success
VM[2] boot success
GIPC_RTOS_READY
GIPC_RTOS_LISTEN ip=10.0.42.2 port=4242
GIPC_STARRY_NET_READY interface=eth0 address=10.0.42.1/24 peer=10.0.42.2
GIPC_RTOS_CONNECTED peer=10.0.42.1:<ephemeral-port>
GIPC_STARRY_STATUS seq=1 payload=8 attempts=1 timeouts=0
GIPC_STARRY_METRIC requests=1 success=1 success_rate=1 errors=0 timeouts=0 attempts=1 reconnects=0 recovery=0 rtt_ns=79471 throughput_bps=100665
GIPC_METRICS_OK
```

| 用例 | 请求/成功 | 成功率 | app errors | timeouts | reconnects | RTT | 有效吞吐 | 结论 |
| --- | ---: | ---: | ---: | ---: | ---: | --- | ---: | --- |
| E01 网络与 listener | — | — | 0 | 0 | 0 | — | — | PASS |
| E02 单请求闭环 | 1/1 | 100% | 0 | 0 | 0 | 79,471 ns | 100,665 B/s | PASS |
| E03 100 请求序列 | 100/100 | 100% | 0 | 0 | 0 | 平均 83,162 ns | 96,196 B/s | PASS |
| 30×100 正常基线 | 3,000/3,000 | 100% | 0 | 0 | 0 | P50 84,210 ns；P95 97,284 ns | 平均 95,620 B/s | PASS |

聚合输出：

```text
GIPC_AGGREGATE requests=3000 success=3000 success_rate=1.000000 app_errors=0 timeouts=0 reconnects=0 recoveries=0 rtt_p50_ns=84210 rtt_p95_ns=97284 throughput_avg_bps=95620
```

运行日志通过 QEMU 在线成功判据，runner、`verify_metrics.py` 与 `aggregate_metrics.py` 均以退出码 0 结束。

## 4. 首次断连恢复

故障注入使第一次连接在响应前关闭，客户端使用相同 sequence 建立第二次连接并成功收到 STATUS：

```text
GIPC_STARRY_STATUS seq=1 payload=8 attempts=2 timeouts=1
GIPC_STARRY_METRIC requests=1 success=1 success_rate=1 errors=0 timeouts=1 attempts=2 reconnects=1 recovery=1 rtt_ns=84449 throughput_bps=94731
GIPC_METRICS_OK
GIPC_AGGREGATE requests=1 success=1 success_rate=1.000000 app_errors=0 timeouts=1 reconnects=1 recoveries=1 rtt_p50_ns=84449 rtt_p95_ns=84449 throughput_avg_bps=94731
```

恢复场景最终请求成功率为 100%，应用错误为 0，并明确记录一次 timeout、一次 reconnect 和一次成功 recovery，结论为 PASS。

## 5. 总体结论

任务二的协议、VirtIO-net/IP 链路、StarryOS 客户端、ArceOS 服务端、正常请求序列、指标统计和断连恢复均通过验收。业务数据经 VirtIO-net 上的 IPv4/TCP 传输；vsock、共享内存、HyperCall 和裸 MMIO 均未作为主数据通道。
