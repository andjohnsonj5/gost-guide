# GOST 弱网 SOCKS5 指南

本文档说明如何在弱网（如 20% 丢包、~25ms RTT）场景下，通过 GOST 使用 UDP 传输通道来承载 SOCKS5 代理，并给出推荐配置与调优思路。

## 可选传输通道

- **KCP**：基于 UDP，带前向纠错和拥塞/重传调优，最适合高丢包、抖动场景。参考 `docs/en/docs/reference/listeners/kcp.md` 与 `docs/en/docs/reference/dialers/kcp.md`。
- **QUIC**：基于 UDP 的标准协议，握手快、具备拥塞控制，但在重度丢包下通常不如 KCP 激进。参考 `docs/en/docs/reference/listeners/quic.md`。

## 基本拓扑

- 服务器：在 UDP 端口上监听 KCP/QUIC 通道，handler 选择 `socks` (SOCKS5)。
- 客户端：本地监听 SOCKS5 端口（可开启 `udp=true` 以转发 UDP 流量），通过 dialer 把流量发到服务器的 KCP/QUIC 监听。

## KCP 推荐配置（丢包 20%、RTT ~25ms）

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

- `mode`: `fast2` 平衡延迟/带宽；需要更激进可试 `fast3`，更保守用 `fast`。
- `nodelay=1, interval=10, resend=2, nc=1`: 减少等待、提升恢复速度。
- `sndwnd/rcvwnd`: 发送/接收窗口，512 起步，吞吐不足可升至 1024（占用内存随之增加）。
- `datashard/parityshard`: FEC 配置。20% 丢包建议 `10/4`，更差链路可提高 parity（如 5），但会增加带宽开销。
- `mtu`: 常用 1350，若路径 MTU 较小需调低以避免分片。
- `sockbuf/smuxbuf`: 内核与多路复用缓冲，弱网下放大有助于平滑吞吐。
- `crypt/key`: 传输加密与密钥，自行设置强随机 key；可选 `aes`, `chacha20` 等。
- `udp=true`（SOCKS5 handler 元数据）: 开启 SOCKS5 的 UDP 转发。

## QUIC 快速示例（弱网可作为备选）

服务器：
```bash
gost -L socks5+quic://:8443
```

客户端：
```bash
gost -L socks5://:1080?udp=true -F socks5+quic://server.example.com:8443
```

如需更稳，可调低 `handshakeTimeout`、`maxIdleTimeout`，并配合 TLS 证书部署。

## 验证与排查

- 本地连通性：`curl -x socks5h://127.0.0.1:1080 https://ipinfo.io` 验证出网。
- 观察延迟/丢包：配合 `iperf3 -u` 或 `mtr` 评估链路，再微调 KCP 的窗口、FEC 与 `interval`。
- 若出现 MTU 问题（大量 `fragmentation needed`），降低 `mtu`。
- 吞吐瓶颈时先增大窗口/缓冲，再视情况提高 parity 或调整 `interval`。
