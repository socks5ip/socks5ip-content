---
title: "代码里挂代理四种写法：四个客户端的认证怎么传（2026）"
description: "curl 能通、Python 里就 407，八成是认证没传对。四种客户端只有 httpx 与 Playwright 原生支持带账号密码的代理；requests 要装 requests[socks] 且协议头写 socks5h；Chromium 不支持认证型 SOCKS5。附验收四点与代理池骨架。"
date: 2026-11-11
tags: [代理IP, requests, httpx, Playwright, 代理池, 账号安全管理]
canonical: https://socks5ip.com.cn/jiaochengzhongxin/dailiip-daima-zenmeyong/
---

# 代码里挂代理四种写法：requests、httpx、Selenium、Playwright（2026）

四种客户端里，只有两种原生支持带账号密码的代理，另外两种必须绕。同一个代理，`curl -x` 能通、换到 Python 里就 407，说明线路没问题，是客户端没把用户名密码带过去。

| 客户端 | 认证怎么传 | 关键限制 |
|---|---|---|
| requests | 写进 URL，协议头用 socks5h:// | 必须装 requests[socks]，否则抛 Missing dependencies |
| httpx | 0.28 起用 proxy=（单数） | 传 proxies= 会报参数不存在 |
| Selenium | 启动参数只认无认证代理 | 带账号密码要本地转发或改 HTTP |
| Playwright | new_context(proxy={...}) | Chromium 不认 SOCKS5 认证，同样要绕 |

依据：四种客户端行为逐项实测，核对日期 2026-11-11；版本差异以你本机实际版本为准。

## 一、先判故障归属，别在代码里猜

第一步，绕开代码，直接用 curl 走同一条代理：

```
curl -x socks5h://用户名:密码@IP:端口 https://ipinfo.io/json -m 15
```

返回的 ip 是你买的那条代理地址，说明代理本身、认证、网络三段全好，问题在代码里。

| 报错 | 真实含义 | 该查哪 |
|---|---|---|
| 407 Proxy Authentication Required | 认证没带上或格式错 | 密码是否转义、协议头对不对 |
| Connection refused / timed out | 端口不通或 IP 已过期 | 后台看有效期、本地防火墙 |
| 返回 200 但 IP 没变 | 代理根本没被使用 | 变量名拼错、库没装全、环境变量覆盖 |

第三步才是改代码。很多人跳过前两步直接换库、换写法，越换越乱。

还有个容易忽略的点：系统代理环境变量会覆盖代码里的设置。测试时先把 HTTP_PROXY、HTTPS_PROXY、ALL_PROXY 及其小写版本从 os.environ 里清掉，否则会出现传了 A 代理、流量从 B 出口走，而日志里看不出异常。

## 二、requests：协议头写成 socks5h 才不漏 DNS

requests 本身不支持 SOCKS，必须装带额外依赖的版本 `pip install "requests[socks]"`。只装 requests 的话，跑到发请求那一步会抛 Missing dependencies for SOCKS support，很多人误以为是代理配错了。

认证信息和地址拼成一条 URL 塞进 proxies 字典，http 与 https 两个键都要给。写成 `socks5h://用户名:密码@IP:端口`——socks5 是本机先解析域名再把 IP 发给代理，socks5h 是把域名整条交给代理。前者会把你访问了哪些域名留在本地 DNS 请求里。

密码含 @、:、/ 必须用 urllib.parse.quote 转义，否则 URL 解析错位，表现为 407 或连到错误端口。

验收标准：返回的 ip 等于后台显示的代理地址；再把 socks5h 改成 socks5 跑一次，两次出口应当相同，不同就说明原路没走代理。

## 三、httpx：参数名改过一次，照抄老教程会报错

httpx 从 0.28 起把 proxies 改成 proxy（单数）且只接受一个值。老教程的字典写法在新版本上会抛 TypeError: unexpected keyword argument 'proxies'。先 `python -c "import httpx; print(httpx.__version__)"` 确认版本再选写法。

它在 DNS 上和 requests 不一样：没有 socks5h 这个头，socks5:// 就走代理解析，从 requests 迁过来不用改协议头。

超时要分层设，别只给一个数字。connect 是建 TCP 到代理的时间，read 是等目标站响应的时间。代理线路慢的时候连接很快、读取很慢，只设总超时会把「线路慢」误判成「代理挂了」。

验收标准：并发 5 个请求时出口应当始终同一条；出现多个地址说明用的是轮换型套餐，属于正常，但要按 IP 分组处理结果。

## 四、Selenium：浏览器内核不支持带认证的 SOCKS5

Chrome 的 --proxy-server 不处理认证。带账号密码的代理直接填进去，浏览器会弹一个代码填不了的登录框，然后卡在那里。

无认证代理（IP 白名单模式）一行就够：启动参数写 `--proxy-server=socks5://IP:端口`，无头模式记得带 `--headless=new`。

有认证的三条路，按推荐度排：本地转发最稳（用 gost 把「带认证的远程代理」转成「本地无认证代理」，浏览器只看到 127.0.0.1）；其次改用 HTTP 代理，多数平台同一条线路同时开 SOCKS5 与 HTTP；最后是装处理 onAuthRequired 的扩展，可行但要维护扩展文件，无头模式下限制更多。

验收标准：打开 IP 查询页读到的地址是代理地址，且没有弹认证框——页面停在空白且进程不退出，基本就是被认证框卡住了。

## 五、Playwright：context 级代理最能隔离，同样卡在认证

Playwright 的代理挂在 context 上而不是浏览器上，同一个实例可以开多个 context、每个用不同代理，天然适配多账号隔离。认证直接传字典，不用拼 URL。

关键限制要说清：Chromium 内核不支持 SOCKS5 代理的认证。server 写成 socks5:// 并同时带用户名密码，认证不会生效，连接会被拒。这不是 Playwright 的缺陷，是内核能力边界，Selenium 那边卡在同一个地方。

还有个容易误判的地方：context 之间隔离的是 Cookie 与存储，不隔离浏览器指纹。同一个 browser 起的多个 context，UA、Canvas、WebGL 仍高度一致。开三个 context 不等于三个独立环境，真要隔离得给每个 context 单独设 UA、时区、语言、视口。

验收标准：每个 context 各自访问一次 IP 查询接口，出口两两不同；再查一次 WebRTC，确认暴露的地址不是本地真实地址。

## 六、出口验收：四个点必须一起验

| 验什么 | 怎么验 | 不合格意味着 |
|---|---|---|
| 出口地址一致 | 访问 ipinfo.io/json 看 ip 字段 | 没走代理 |
| 地理位置与 ASN 一致 | 查 country、isp、as 三个字段 | 标称住宅、实际云机房 |
| DNS 未泄漏 | 跑 dnsleaktest 看 DNS 归属地 | 用了 socks5 本地解析 |
| 浏览器特征自洽 | 时区、语言、WebRTC 是否跟代理地区一致 | 典型自动化特征 |

第二项是识破「标称住宅、实际机房」的唯一办法：买美国住宅静态，却看到 ASN 挂着 Amazon、Google、DigitalOcean，就是买错了。一个美国地址配着 Asia/Shanghai 时区和中文 Accept-Language，是最容易被识别的组合。

四项全过才算可用，任一项不过都别急着上量——上量后排查成本是现在的十倍。

## 七、代理池：先分清哪类失败该换 IP

| 失败类型 | 例子 | 换 IP 有用吗 |
|---|---|---|
| 可重试 | 超时、连接被拒、407、目标站 403/429 | 有用，大概率被标记了 |
| 不可重试 | DNS 解析失败、目标站 404、代码抛 TypeError | 没用，换一百条也一样失败 |

骨架不需要复杂框架：列表 + 健康检查 + 失败冷却 + 轮换，四十行够用。轮换粒度按业务定：同一账号的连续操作要绑同一条地址，不同账号之间才轮换。把「每请求换一次」当默认值是常见误区，它会让会话看起来像被一群人轮流使用。

健康检查别只 ping 端口，端口通不代表能用，很多失效 IP 依然接受 TCP 连接。应该发一个真实请求并校验返回的地址是否等于预期值。

验收标准：人为把池里一条代理的密码改错，再跑 10 次请求，应当看到它被标记、冷却，其余请求不受影响，总成功率仍接近 100%。

## 八、官方入口与相关页面

| 用途 | 页面 |
|---|---|
| 🧰 客户端下载与配置教程 | [代理工具中心](https://socks5ip.com.cn/dailigongjuzhongxin/) ｜ [SSTap 配置教程](https://socks5ip.com.cn/dailigongju/sstapjiaocheng/) |
| 🔍 出口与住宅属性自测 | [IP 综合检测中心](https://socks5ip.com.cn/ip-check-center/) |
| 💰 各平台价格表与比价 | [价格中心](https://socks5ip.com.cn/jiagezhongxin/) |
| 🚀 聚合开通 | [一站式注册中心](https://linkdd.cn/socks5ip) |

## 九、常见问题

**Q：curl 能通，Python 就是不通，为什么？**
三个常见原因：没装 requests[socks]；协议头写成 socks5:// 而密码含特殊字符没转义；系统 HTTP_PROXY 覆盖了你传的参数。按这三条依次排除。

**Q：socks5 和 socks5h 到底差在哪，必须用 h 吗？**
差在 DNS 由谁解析。socks5 让本机先解析域名，访问记录留在本地；socks5h 把域名交给代理解析。做多账号业务必须用 h。

**Q：Selenium 能挂带账号密码的 SOCKS5 吗？**
不能。Chromium 的启动参数不处理认证，会弹一个代码填不了的登录框。要么用 IP 白名单的无认证代理，要么本地起转发，要么改 HTTP 代理。

**Q：一条代理能开多少并发？**
没有统一数字，取决于套餐的并发上限与目标站容忍度。从 5 个并发起测，出现 429 就到顶了。一上来开 50 个是把自己送进黑名单最快的方式。

**Q：IP 换了，网站还是认出我是同一个人，为什么？**
IP 只是识别维度之一。Cookie、指纹、时区、字体、Canvas 都不随 IP 变。只换 IP 不换环境，等于换了外套没换脸。

**Q：买之前怎么验？**
优先选支持免费测试的平台，按第六章四项逐个过，别只测能不能连上。测试节点和你实际买的那条线不一定是同一批机器，正式开通后再验一次。
