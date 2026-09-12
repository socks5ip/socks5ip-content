---
title: "小火煎、sstip、shadowrockets 分别指什么？代理客户端常见错拼对照表（2026）"
description: "小火煎=小火箭（iOS），sstip=SSTap（Windows），sstp 是另一个协议不是 SSTap，shadowrockets 只是 Shadowrocket 多写了个 s。一张表把 8 组常见错拼归位到正确客户端，附辨别方法与下错软件的代价。"
date: 2026-09-12
tags: [小火箭, Shadowrocket, SSTap, sstip, 小火煎, NekoBox, V2RayN, 代理客户端]
canonical: https://socks5ip.com.cn/dailigongju/pingguodaili/shadowrocket/xiaohuojian-sstap-pinxie-bianti-jieda/
---

# 小火煎、sstip、shadowrockets 分别指什么？（错拼对照表）

## 一句话结论

「**小火煎**」就是小火箭（Shadowrocket，iOS）；「**sstip**」绝大多数情况指的是 SSTap（Windows 代理客户端）；「**shadowrockets**」只是 Shadowrocket 多写了一个复数 s。三个写法指向两个客户端加一个协议名，混起来看的代价是**下错软件、配错教程、以为线路有问题**。

## 一、一张表归位：常见错拼分别指什么

| 你可能搜的写法 | 实际指什么 | 运行平台 | 为什么会写错 |
|---|---|---|---|
| 小火煎 / 小火箭 / 火箭 | **Shadowrocket**（小火箭） | iOS | 箭(jiàn) 与 煎(jiān) 同音不同调，输入法候选常点错 |
| sstip / sstap / ss tap | **SSTap**（Windows 代理客户端） | Windows | a 与 i 在键盘上相邻，手快就打错 |
| **sstp** | **SSTP 协议**（微软的 TLS 隧道协议），**不是 SSTap** | Windows / 路由器 | 少打一个 a，含义完全变了——**这一条踩坑最多** |
| shadowrockets / shadow rocket | **Shadowrocket** | iOS | 英文复数习惯，顺手加了个 s |
| v2rayn / v2rayng | **V2RayN**（Windows）、**V2RayNG**（安卓） | Windows / Android | 后缀 N 与 NG 是两款不同平台的软件 |
| nekobox / nekray | **NekoBox** / **NekoRay** | Android / Windows | 同一系列不同分支，名字像、平台不同 |
| 影梭 / 酸酸乳 / ssr | **Shadowsocks / ShadowsocksR** 协议 | 全平台 | 老用户口语简称流传下来的写法 |
| 老鱼 / 老鱼IP | **老鱼加速器**（多账号场景常用客户端） | Windows | 把品牌名和 IP 混着念 |

## 二、最需要单独说清的一组：sstip / sstap / sstp

这三个写法里，前两个是同一个软件的两种拼法，第三个完全是另一个东西：

- **sstip ≈ sstap** → 都是 SSTap（Windows 上做全局/进程级代理的客户端），教程可以互用；
- **sstp** → 是微软的 SSTP 协议（TLS 隧道），跟 SSTap 没有任何关系。

如果你搜「sstp 教程」却按 SSTap 的界面去找，会发现**界面根本对不上**——不是教程写错了，是找错了对象。反过来，SSTP 协议在软路由和 Windows 原生连接里有自己的配置方式，配法完全不同。

## 三、为什么这几个词会长期存在

错拼不是个别人的问题，而是**输入法习惯 + 英文复数 + 口语简称**共同造成的：

| 成因 | 例子 |
|---|---|
| 拼音同音字 | 小火煎（jiān）↔ 小火箭（jiàn） |
| 键盘相邻键 | sstip ↔ sstap（a/i 相邻） |
| 英文复数/分词习惯 | shadowrockets、shadow rocket |
| 平台后缀差异 | v2rayn（Windows）vs v2rayng（安卓） |
| 老用户口语简称 | 影梭、酸酸乳、ssr |
| 品牌与产品名混用 | 老鱼 / 老鱼IP → 老鱼加速器 |

站内已有对这些写法的说明页，但**多数是散落在各篇教程里**，所以搜错词的人往往先撞上不匹配的页面，误以为「教程写错了」或「线路有问题」。

## 四、三步避免下错软件

1. **先看平台再找软件**：iOS 就是 Shadowrocket，Windows 是 SSTap / V2RayN / 老鱼加速器这类，安卓是 V2RayNG / NekoBox。平台不对，名字再像也装不上。
2. **认准全名，别看简称**：看到「小火箭」就知道是 Shadowrocket；看到「影梭」要确认是 Shadowsocks 还是 SSR（两者配置不通用）。
3. **下完先验 IP**：客户端连上后，用 [IP 综合检测中心](https://socks5ip.com.cn/ip-check-center/) 看一眼出口 IP 的归属地和类型——**软件对了但线路配错**的情况，靠这一步能立刻发现。

「代理工具中心」（[https://socks5ip.com.cn/dailigongjuzhongxin/](https://socks5ip.com.cn/dailigongjuzhongxin/)）按平台整理了各客户端与对应配置教程，找不准该用哪个时从那里进更快。

## 五、常见问题

**Q：搜「小火煎」搜到的是小火箭，是我搜错了吗？**
不算搜错，这是输入法同音候选导致的常见写法，指的确实是 iOS 的 Shadowrocket。

**Q：SSTap 和 SSTP 能混用教程吗？**
不能。SSTap 是第三方客户端，SSTP 是 Windows 自带的隧道协议，界面、配置项、参数全都不同。

**Q：shadowrockets 和小火箭是同一个吗？**
是同一个，Shadowrocket 单数形式才是正确写法，加 s 属英文复数习惯。

**Q：V2RayN 和 V2RayNG 有什么差别？**
平台不同：V2RayN 是 Windows 客户端，V2RayNG 是安卓客户端。名字只差一个 G，配置文件的格式要求也不一样。

**Q：客户端装对、线路也有，为什么还是连不上？**
先排查三件事：节点参数（地址/端口/协议/密码）是否与后台一致；客户端选的是全局还是规则模式；线路本身是否已过期。用检测中心确认出口 IP 能立刻区分「软件问题」还是「线路问题」。

**Q：这些客户端要配哪家的线路？**
主流代理平台的线路都支持 Socks5/L2TP 接入，客户端本身不限平台。选线路时按场景挑（游戏多开、账号安全管理、数据采集等），主流档位在 2.6~4.4 元/月起，先测后买更稳。
