---
title: "从零开始接入 DN42 网络 - 3: Dummy 网卡创建与保持、ROA 配置"
date: 2026-04-23
description: "如题。"
image: img/cover.png
tags:
    - 网络
    - 叠加网络
    - DN42
    - BGP
    - Bird2
    - ROA
categories:
    - 网络
---

## 前言

在前两篇文章里我们已经完成了 DN42 网络的注册和 Bird2 的基础配置，接下来首先必须创建一个 dummy 网卡来承载 BGP 协议的 IP 地址，之后还要配置 ROA 来让 Bird 能够正确地验证路由的合法性。

## Dummy 网卡创建与保持

DN42 网络中 BGP 协议的 IP 地址是不会绑定在物理网卡上的，而是绑定在一个 dummy 网卡上，它作为虚拟的网络接口来承载这些 IP 地址，而 Bird 会监听这个接口来进行 BGP 通信。

创建 dummy 接口：

```bash
ip link add dn42-dummy type dummy
ip link set dev dn42-dummy up
```

为其分配此路由器的 IP 地址（对 ipv4 和 ipv6 都要执行一次）：

```bash
ip addr add dev dn42-dummy <路由器IP>/<子网>
```

需要注意这里的 IP 指的是 DN42 网络中选定的 IP 地址，而不是主机的真实公网 IP。

然而这些命令是临时生效的，重启后会丢失。如果需要维持这个 dummy 网卡的配置在重启后不变，可以把这些命令写入系统的网络配置中，例如我的 Ubuntu 24.04 默认使用 Netplan 管理网络，可以创建一个新的 Netplan 配置文件 `/etc/netplan/60-dn42-dummy.yaml` 来定义这个 dummy 网卡：

```yaml
network:
  version: 2
  dummy-devices:
    dn42-dummy:
      addresses:
        - 172.23.15.34/27
        - fd24:ac8e:9e04::2/48
```

保存后执行

```bash
chmod 600 /etc/netplan/60-dn42-dummy.yaml
netplan apply
```

来设置安全的权限并应用配置。如果不设置 600 权限，可能会在 apply 时遇到以下 warning：

```plaintext
** (generate:924409): WARNING **: 03:49:04.343: Permissions for /etc/netplan/60-dn42-dummy.yaml are too open. Netplan configuration should NOT be accessible by others.
```

## ROA 配置

ROA 表一般是由社区维护的一个公共资源，包含了 DN42 网络中所有合法的路由前缀和它们对应的 ASN 信息。

可以直接从注册中心生成，也可以用下面这些预构建的 BIRD ROA 表，我用的是 burble 源里列出的最后两个表：

由 [dn42regsrv](https://git.burble.com/burble.dn42/dn42regsrv) 生成的 ROA 文件可从 burble.dn42 获取：

| URL | IPv4/IPv6 | 说明 |
| --- | --- | --- |
| <https://dn42.burble.com/roa/dn42_roa_46.json> | 两者皆有 | 用于 RPKI 的 JSON 格式 |
| <https://dn42.burble.com/roa/dn42_roa_bird1_46.conf> | 两者皆有 | Bird1 格式 |
| <https://dn42.burble.com/roa/dn42_roa_bird1_4.conf> | 仅 IPv4 | Bird1 格式 |
| <https://dn42.burble.com/roa/dn42_roa_bird1_6.conf> | 仅 IPv6 | Bird1 格式 |
| <https://dn42.burble.com/roa/dn42_roa_bird2_46.conf> | 两者皆有 | Bird2 格式 |
| <https://dn42.burble.com/roa/dn42_roa_bird2_4.conf> | 仅 IPv4 | Bird2 格式 |
| <https://dn42.burble.com/roa/dn42_roa_bird2_6.conf> | 仅 IPv6 | Bird2 格式 |

由 [roa_wizard](https://github.com/Kioubit/dn42_registry_wizard) 生成的 ROA 文件可从 kioubit.dn42 获取：

| URL | IPv4/IPv6 | 说明 |
| --- | --- | --- |
| <https://kioubit-roa.dn42.dev/?type=v4> | 仅 IPv4 | Bird2 格式 |
| <https://kioubit-roa.dn42.dev/?type=v6> | 仅 IPv6 | Bird2 格式 |
| <https://kioubit-roa.dn42.dev/?type=json> | 两者皆有 | 用于 RPKI 的 JSON 格式 |

为了即时生效，并且能定时更新到最新的 ROA 表，我们分两步走：

1. 先简单地手动下载最新的 ROA 表到本地，并放到 Bird 的配置目录下，以便直接跑通 Bird 服务。

    ```bash
    wget -4 -O /tmp/roa_dn42.conf https://dn42.burble.com/roa/dn42_roa_bird2_4.conf && mv -f /tmp/roa_dn42.conf /etc/bird/roa_dn42.conf
    wget -4 -O /tmp/roa_dn42_v6.conf https://dn42.burble.com/roa/dn42_roa_bird2_6.conf && mv -f /tmp/roa_dn42_v6.conf /etc/bird/roa_dn42_v6.conf
    ```

2. 使用 cron 或者 systemd timer 定时执行一个脚本来定期更新 ROA 表，这里我还是用 cron。

```bash
0 * * * * wget -4 -O /tmp/roa_dn42.conf https://dn42.burble.com/roa/dn42_roa_bird2_4.conf && mv -f /tmp/roa_dn42.conf /etc/bird/roa_dn42.conf && wget -4 -O /tmp/roa_dn42_v6.conf https://dn42.burble.com/roa/dn42_roa_bird2_6.conf && mv -f /tmp/roa_dn42_v6.conf /etc/bird/roa_dn42_v6.conf && birdc c
```

这个 cron job 每小时执行一次，下载最新的 ROA 表并替换掉旧的文件，最后执行 `birdc c` 来让 Bird 重新加载配置，使新的 ROA 表生效。

现在 ROA 配置好了，Bird 就能够正确地运行了，下一步就可以直接开始添加 Peer 了。运行一下 `birdc c` 来跑通 Bird 服务，输出应该类似于：

```plaintext
BIRD 2.18.1 ready.
Reading configuration from /etc/bird/bird.conf
Reconfigured
```
