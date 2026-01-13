---
comments: true
---

# 边缘节点 SOCKS5 汇聚与公网暴露

本节给出一个完整方案：在多个边缘节点内网没有现成 SOCKS5 的情况下，用 GOST 建立 SOCKS5，并通过 Relay + RTCP 反向映射到公网汇聚机，再按需对公网开放或仅本机可用。

## 适用场景

- 多个边缘节点分布在不同内网，外网无法直连。
- 需要统一从公网汇聚机访问各边缘节点的 SOCKS5。
- 可能需要给 Kasm 等浏览器/应用提供透明或显式代理入口。

## 核心概念

- `relay`：GOST 自有协议，兼具代理和转发能力，支持 TCP/UDP 和认证；协议本身不加密，可配合 `wss/tls/quic` 等数据通道。
- `rtcp`：远程 TCP 端口转发，把“监听端口”放到转发链末端节点上，实现远程端口映射。
- 约束：当 `rtcp` 使用转发链时，链末端必须是开启 `bind=true` 的 SOCKS5 或 Relay 服务。

## 总体拓扑

```
公网用户/应用
    |
HUB_PUBLIC_IP:21080  <--- 映射端口
    |
  (Relay + RTCP 通道)
    |
边缘节点内网 SOCKS5 :1080
```

## 步骤一：汇聚机启动 Relay（BIND）

建议使用加密数据通道：

```bash
gost -L relay+wss://:8443?bind=true
```

不需要加密时可用：

```bash
gost -L relay://:8420?bind=true
```

## 步骤二：边缘节点提供 SOCKS5

如果内网机器没有 SOCKS5，可直接用 GOST 起一个：

```bash
gost -L socks5://user:pass@:1080
```

## 步骤三：边缘节点反向映射到公网汇聚机

将边缘节点 SOCKS5 映射到汇聚机公网端口（示例映射到 21080）：

```bash
gost -L rtcp://:21080/127.0.0.1:1080 -F relay+wss://HUB_PUBLIC_IP:8443
```

多个边缘节点时，使用不同的公网端口：

```bash
gost -L rtcp://:21081/127.0.0.1:1080 -F relay+wss://HUB_PUBLIC_IP:8443
gost -L rtcp://:21082/127.0.0.1:1080 -F relay+wss://HUB_PUBLIC_IP:8443
```

## 访问方式

### 对公网暴露

公网用户直接使用：

```bash
socks5://HUB_PUBLIC_IP:21080
```

### 仅汇聚机本机可用

若只希望汇聚机本机访问，将监听地址限制为 `127.0.0.1`：

```bash
gost -L rtcp://127.0.0.1:21080/127.0.0.1:1080 -F relay+wss://HUB_PUBLIC_IP:8443
```

本机使用示例：

```bash
curl --socks5-hostname 127.0.0.1:21080 https://example.com
```

## Kasm 浏览器访问

Kasm 浏览器只要能访问到 SOCKS5 地址即可使用。若无法直接设置代理参数，可选以下“透明化”方案：

### 方案 A：TUN 透明代理（推荐）

```bash
gost -L tun://:0?net=192.168.123.1/24&mtu=1420 -F socks5://user:pass@HUB_PUBLIC_IP:21080
```

将 Kasm 流量路由到该 TUN 网段即可。

### 方案 B：nftables 重定向（RED/TProxy）

```bash
gost -L red://:12345 -F socks5://user:pass@HUB_PUBLIC_IP:21080
```

再用 nftables 将 Kasm 出站流量重定向到 `:12345`。

## 安全建议

- 公网暴露务必开启 SOCKS5 认证。
- Relay 建议使用 `wss/tls/quic` 加密通道。
- 配合防火墙或白名单限制访问来源。
