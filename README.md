# GOST 弱网 SOCKS5 指南

本文档说明在弱网（例如 20% 丢包、~25ms RTT）场景下，如何用 GOST 的 UDP 传输通道承载 SOCKS5 代理，并给出推荐配置与调优思路。

## 可选传输通道

- **KCP**：基于 UDP，具备前向纠错和拥塞/重传调优，弱网首选。参考 `docs/en/docs/reference/listeners/kcp.md` 与 `docs/en/docs/reference/dialers/kcp.md`。
- **QUIC**：标准 UDP 协议，握手快，有拥塞控制，但在重度丢包下通常不如 KCP 激进。参考 `docs/en/docs/reference/listeners/quic.md`。

## 基本拓扑

- 服务器：在 UDP 端口上监听 KCP/QUIC，handler 选 `socks` (SOCKS5)。
- 客户端：本地监听 SOCKS5 端口（如需转发 UDP 流量在 handler 元数据加 `udp=true`），通过 dialer 把流量送到服务器的 KCP/QUIC。

## KCP 推荐配置（20% 丢包 / 25ms RTT）

服务器：
```bash
gost -L socks5+kcp://:8388?
  key=mykey&crypt=aes&mode=fast2&mtu=1350&
  sndwnd=512&rcvwnd=512&
  datashard=10&parityshard=4&
  nodelay=1&interval=10&resend=2&nc=1&
  sockbuf=16777216&smuxbuf=8388608
```

客户端：
```bash
gost -L socks5://:1080?udp=true -F socks5+kcp://server.example.com:8388?
  key=mykey&crypt=aes&mode=fast2&mtu=1350&
  sndwnd=512&rcvwnd=512&
  datashard=10&parityshard=4&
  nodelay=1&interval=10&resend=2&nc=1&
  sockbuf=16777216&smuxbuf=8388608
```

### 参数说明与调优

- `mode`: `fast2` 平衡延迟/带宽；更激进可试 `fast3`，更保守用 `fast`。
- `nodelay=1, interval=10, resend=2, nc=1`: 减少等待、提升恢复速度。
- `sndwnd/rcvwnd`: 发送/接收窗口，512 起步，吞吐不够可升至 1024（占用内存增加）。
- `datashard/parityshard`: FEC 配置。20% 丢包建议 `10/4`，更差链路可提高 parity（如 5），但带宽占用增大。
- `mtu`: 常用 1350；若路径 MTU 较小需调低以避免分片。
- `sockbuf/smuxbuf`: 内核与多路复用缓冲，弱网下放大会更平滑。
- `crypt/key`: 传输加密与密钥，自行设置强随机 key；可选 `aes`, `chacha20` 等。
- `udp=true`（SOCKS5 handler 元数据）: 开启 SOCKS5 的 UDP 转发。

## QUIC 快速示例（弱网备选）

服务器：
```bash
gost -L socks5+quic://:8443
```

客户端：
```bash
gost -L socks5://:1080?udp=true -F socks5+quic://server.example.com:8443
```

如需更稳，可调低 `handshakeTimeout`、`maxIdleTimeout`，并配合 TLS 证书。

## 验证与排查

- 本地出网：`curl -x socks5h://127.0.0.1:1080 https://ipinfo.io`
- 观测链路：`iperf3 -u`、`mtr` 评估丢包/抖动，再微调窗口、FEC、`interval`。
- 发现 MTU 报错（如 “fragmentation needed”）时降低 `mtu`。
- 吞吐不足先加窗口/缓冲，再视情况提高 parity 或调整 `interval`。
