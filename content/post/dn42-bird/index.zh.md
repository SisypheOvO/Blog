---
title: "从零开始接入 DN42 网络 - 2: DN42 的 Bird2 配置"
date: 2026-04-22
description: "记录一下 DN42 网络中 Bird2 的配置详解"
image: img/cover.png
tags:
    - 网络
    - 叠加网络
    - DN42
    - BGP
    - Bird2
categories:
    - 网络
---

## 前言

在 dn42-registry 仓库里把身份注册了之后，除去搭建 VPN 隧道外，下一步就是配置 BGP 路由守护进程了。DN42 网络中我还是习惯用 Bird2 作为 BGP 来管理路由。

Bird2 的配置文件相对来说比较复杂，包含了全局设置、协议定义、路由过滤等多个部分。而且 DN42 自己的 wiki 没有做 i18n，这个等以后熟悉社区了跟维护者沟通吧。

时隔一个月，借这篇博客详细解释、记录一下 wiki 里提供的标准配置示例各部分的意思。整体思路就是先把大的 `bird.conf` 文件搭建好。这里面会依赖外部引入的 ROA 表和对等方配置，ROA 表是起 Bird 服务所必须的；对等方配置可以依赖 `bird.conf` 中定义的模板来简化配置，后续添加新的对等方时只需要写一个小的配置文件就行了。后续流程还包括创建一个 dummy 网卡，再往后还有 BGP Community 配置、RPKI 配置和开启 BFD 之类的。

## Bird2 配置详解

_官方文章：[Bird2](https://dn42.dev/howto/Bird2)_

---

### 全局变量定义

```ini
define OWNAS = <OWNAS>;
define OWNIP = <OWNIP>;
define OWNIPv6 = <OWNIPv6>;
define OWNNET = <OWNNET>;
define OWNNETv6 = <OWNNETv6>;
define OWNNETSET = [<OWNNET>+];
define OWNNETSETv6 = [<OWNNETv6>+];
```

这一段用 `define` 定义了一些 ASN、IP 段相关的全局变量，方便后面多次复用，写过代码的话应该都懂。`OWNNETSET` 和 `OWNNETSETv6` 是前缀集合（prefix set），加了 `+` 表示匹配该前缀及其所有更长的子前缀。

之所以要同时定义 `OWNNET` 和 `OWNNETSET` 两个变量，是因为 Bird 在某些语法场景下（比如 `route ... reject`）需要单个前缀，而在过滤器中做匹配（`net ~`）时需要集合，Bird 的变量解析机制不允许在集合字面量中直接内嵌变量。

---

### 基础设置

```ini
router id OWNIP;

protocol device {
    scan time 10;
}
```

`router id` 用这台主机的 IPv4 地址全局变量作为路由器的全局标识符，BGP 要求每个路由器有唯一 ID。

`protocol device` 让 Bird 每 10 秒扫描一次系统网络接口的状态变化（如接口 up/down、地址增删）。

---

### 工具函数

```ini
function is_self_net() -> bool {
  return net ~ OWNNETSET;
}
function is_self_net_v6() -> bool {
  return net ~ OWNNETSETv6;
}
```

意思基本显然且平凡，即判断当前路由的前缀是否属于本 AS 自己的网段，下文中的过滤器 `filter` 中会用来避免从外部邻居那里接受自己的前缀，以防止路由环路。

```ini
function is_valid_network() -> bool {
  return net ~ [
    172.20.0.0/14{21,29},
    172.20.0.0/24{28,32},
    ...
    10.0.0.0/8{15,24}
    172.20.0.0/14{21,29}, # dn42
    172.20.0.0/24{28,32}, # dn42 Anycast
    172.21.0.0/24{28,32}, # dn42 Anycast
    172.22.0.0/24{28,32}, # dn42 Anycast
    172.23.0.0/24{28,32}, # dn42 Anycast
    172.31.0.0/16+,       # ChaosVPN
    10.100.0.0/14+,       # ChaosVPN
    10.127.0.0/16+,       # neonetwork
    10.0.0.0/8{15,24}     # Freifunk.net
  ];
}

...

function is_valid_network_v6() -> bool {
    return net ~ [
    fd00::/8{44,64} # ULA address space as per RFC 4193
  ];
}
```

一个白名单过滤器，定义了 dn42 生态中所有合法的 IPv4 地址范围。`{21,29}` 的语法表示前缀长度必须在 21 到 29 之间。不在此范围内的路由一律拒绝，==防止对等方泄漏公网路由或其他无关路由进来==。各段分别对应注释中的不同网络。除了官方分配的地址段之外，还有一些社区网络（如 ChaosVPN、neonetwork、Freifunk.net）也在 dn42 中使用了自己的地址段。

IPv6 版本类似，只接受 `fd00::/8{44,64}`，即 RFC 4193 定义的 ULA（唯一本地地址）空间。

---

### ROA 表（路由源授权）

```ini
roa4 table dn42_roa;
roa6 table dn42_roa_v6;

protocol static {
    roa4 { table dn42_roa; };
    include "/etc/bird/roa_dn42.conf";
};

protocol static {
    roa6 { table dn42_roa_v6; };
    include "/etc/bird/roa_dn42_v6.conf";
};
```

声明了两张 ROA 表（IPv4 和 IPv6），并通过 `include` 从外部文件加载 ROA 数据。

ROA 数据的本质是"哪个 ASN 被授权通告哪个前缀"的映射关系。下文中的 BGP 过滤器会用 `roa_check()` 查询这张表来验证路由的合法性。

---

### Kernel 协议（Bird <-> 系统内核）

```ini
protocol kernel {
    scan time 20;
    ipv4 {
        import none;
        export filter {
            if source = RTS_STATIC then reject;
            krt_prefsrc = OWNIP;
            accept;
        };
    };
}
```

控制 Bird 与 Linux 内核路由表之间的同步：

- **import none**：不从内核路由表导入任何路由到 Bird,从而限制 Bird 仅通过 BGP 学习路由）。
- **export filter**：将 Bird 学到的路由写入内核，但排除静态路由（`RTS_STATIC`），因为静态路由只是占位用的（见下文），不需要写入内核。
- **krt_prefsrc = OWNIP**：设置导出到内核的路由的首选源地址，确保主机（即该路由器）向 dn42 发出的数据包使用自己的 dn42 IP 作为源地址，而不是随机选一个。

IPv6 同理。

---

### 静态路由（前缀通告的来源）

```ini
protocol static {
    route OWNNET reject;

    ipv4 {
        import all;
        export none;
    };
}
```

声明一条 `reject` 类型的静态路由，目的不是为了真正路由流量，而是让 Bird 内部有一条自己网段的路由记录。

BGP 的 export filter 中检查 `source ~ [RTS_STATIC, RTS_BGP]`，这条静态路由满足 `RTS_STATIC` 条件，从而被通告给邻居——也就是告诉整个 dn42："我拥有这个前缀，请把目的地是这个前缀的流量发给我。" `reject` 类型意味着如果真有流量到达但没有更具体的路由可匹配，就丢弃（黑洞路由），这是正常行为。

ipv6 同理。

---

### BGP 模板 (Core part)

```ini
template bgp dnpeers {
    local as OWNAS;
    path metric on;

    ipv4 {
        <import filter part>
        <export filter part>
    };

    ipv6 {
        <import filter part>
        <export filter part>
    };
}
```

`template` 定义了一个可复用的 BGP 配置模板，所有 dn42 对等方都继承它，避免重复配置。`path metric on` 表示 AS 路径中每一跳的代价为 1，Bird 会优选 AS path 最短的路由。

以下是 import/export filter 的具体内容，ipv4 和 ipv6 的过滤器内容相同，分别放在对应的 `ipv4` 和 `ipv6` 块中。

#### Import Filter（入站过滤）

```ini
import filter {
    if is_valid_network() && !is_self_net() then {
        if (roa_check(dn42_roa, net, bgp_path.last) != ROA_VALID) then {
            # Reject when unknown or invalid according to ROA
            print "[dn42] ROA check failed for ", net, " ASN ", bgp_path.last;
            reject;
        } else accept;
    } else reject;
};
```

安全检查，由外到内：

1. 前缀必须在 dn42 合法范围内，且不接受自己的前缀从别人那里传回来，以防环路。
2. `roa_check()` 验证通告这个前缀的最终 ASN（`bgp_path.last`）是否被 ROA 表授权。不是 `ROA_VALID` 的一律拒绝，并打印日志。

全部通过则 accept。

#### Export Filter（出站过滤）

```ini
export filter {
    if is_valid_network() && source ~ [RTS_STATIC, RTS_BGP] then accept;
    else reject;
};
```

只通告属于 dn42 合法范围的、且来源是静态路由（自己的前缀）或 BGP 学来的路由。避免将内核路由、直连路由等无关路由泄漏出去。

#### Import Limit

```ini
import limit 9000 action block;
```

如果从单个对等方收到超过 9000 条路由，就阻断该会话。这是一种安全机制，防止某个对等方因配置错误向本端泄漏海量路由（比如全表泄漏），保护主机的内存以及稳定性。

---

### 引入对等方配置

```ini
include "/etc/bird/peers/*";
```

从 `/etc/bird/peers/` 目录加载所有对等方的配置文件。一般来说每次创建一个 peer 时，只需要在该目录下创建一个与该 peer 对应的配置文件即可。每个配置文件中定义一个 `protocol bgp ... from dnpeers` 实例，继承上面提到的模板，基本上只需要填入邻居的 IP 和 ASN 就行了。

---

### 整体数据流

整个配置的数据流思路：

- 路由器（即这台主机）通过静态路由声明自己拥有的前缀，BGP 模板将其通告给所有对等方；
- 从对等方接收路由时经过白名单、防环、ROA 的各个验证，合格的路由写入内核路由表（附带正确的源地址），从而使路由器能够正确地转发 dn42 流量。
