---
title: "crontab 里的脚本走了本机 IP：cron 只给四个变量（2026）"
description: "同一份脚本白天手动跑走账上的代理，凌晨定时跑就换回服务器公网出口，日志一条报错都没有。原因是 cron 起的进程只拿得到 SHELL、PATH、HOME、LOGNAME 四个变量，.bashrc 里 export 的代理它看不见。附四种补法排序、systemd 两条限制与三步验证命令。"
date: 2026-11-11
tags: [代理IP, crontab, 环境变量, 排错, 账号安全管理]
canonical: https://socks5ip.com.cn/zuixinzixun/jishuzixun/dailiip-crontab-dingshirenwu-zou-bendiip/
---

# crontab 里的脚本走了本机 IP：cron 只给四个变量（2026）

以为是脚本写错了，其实是 cron 根本没把代理变量交给它——终端里 export 的那一套，定时任务一个都看不见。同一份脚本，白天手动跑走的是账上那条线，凌晨那批定时跑就换回服务器自己的公网出口，日志没有一条报错，它照常抓完、照常入库，只是每一跳都没经过买来的 IP。

| 现象 | 真实原因 | 会不会报错 | 先查哪 |
|---|---|---|---|
| 跑完了，出口是本机 IP | 代理变量没进 cron 环境 | 不报错 | crontab 顶部变量行 |
| ModuleNotFoundError | PATH 里没有 venv | 报错并退出 | 改用解释器绝对路径 |
| FileNotFoundError | 工作目录不是脚本所在目录 | 报错 | 路径全写绝对 |
| 中文日志乱码 | LANG 没设，落到 POSIX | 不报错 | 补 LANG / LC_ALL |

依据：Linux crontab(5) 与 systemd.exec(5) 手册口径核对，核对日期 2026-11-11。

## 一、终端里有的，定时任务里为什么没有

crontab 起的进程只拿到 SHELL、PATH、HOME、LOGNAME 四个变量。这不是配置失误，是既定行为：crontab(5) 写明 SHELL 固定为 /bin/sh，HOME 与 LOGNAME 从该用户的 passwd 行取，PATH 固定为 /usr/bin:/bin。

根子在 shell 的三种启动方式。登录 shell 读 /etc/profile 与 ~/.bash_profile；交互式非登录 shell 读 ~/.bashrc；非交互 shell 两个都不读，只看 BASH_ENV 有没有指向某个文件。cron 走的是第三种，而且用的还是 /bin/sh——它连 bash 都不是。

所以「我在 .bashrc 里写了 export」这句话对终端成立、对 cron 不成立。更麻烦的是它不报错：变量缺了，程序就按没有代理的方式跑，一切正常。

想亲眼看差距，在 crontab 里加一条 `* * * * * env > /tmp/cron_env.txt`，一分钟后打开文件，和终端里 env 的输出对比。终端那份几十行，cron 这份通常只有四行。

## 二、丢的四样里，只有一样会报错

| 丢的东西 | 终端里 | cron 里 | 典型表现 | 怎么补 |
|---|---|---|---|---|
| 代理变量 | 有 | 无 | 静默走本机出口 | crontab 顶部写死 |
| 解释器路径 | venv 优先 | 系统 python | 依赖缺失报错退出 | 写 /path/venv/bin/python |
| 工作目录 | 当前目录 | 家目录 | 读错文件、写错位置 | 路径全绝对化 + cd |
| 语言环境 | zh_CN.UTF-8 | POSIX | 乱码、编码异常 | LANG=C.UTF-8 |

四种里只有解释器那条会当场报错，其余三样都是静默失败。判断「定时任务有没有问题」，不能看它有没有报错，只能看出口。

工作目录那条尤其阴：脚本里写相对路径，读的是家目录下的同名文件；写的话，把数据写到别处去了——它成功执行了，比报错更难查。

## 三、父进程不同，继承到的东西就不同

| 怎么启动的 | 父进程是谁 | 环境从哪来 | 代理变量在不在 |
|---|---|---|---|
| 终端直接跑 | 你的登录 shell | .bashrc / .bash_profile | 在 |
| nohup 加 & | 当前这个 shell | 整体继承 | 在 |
| crontab | cron 守护进程 | 只剩四个变量 | 不在 |
| systemd 服务 | PID 1 | 单元文件里的 Environment | 看配置 |

进程的环境变量在 exec 那一刻从父进程复制一份。nohup 的父进程是你的 shell，shell 有代理变量，子进程就有；cron 的父进程是守护进程，它自己的环境是开机时从 init 拿到的。

要确认某个正在跑的进程实际拿了什么环境，别猜，直接读 `/proc/PID/environ`（把 \0 转成换行再看），或者用 `ps eww -p PID`。注意这份是 exec 时的快照，运行中改不了。

## 四、四种补法的可靠程度差在哪

| 做法 | 生效范围 | 改一处影响 | 主要坑 |
|---|---|---|---|
| crontab 顶部变量行 | 该用户全部任务 | 全部 | 不展开 $VAR、不能加引号 |
| 命令前带 env | 单条任务 | 仅这条 | 重复写、易漏改 |
| 脚本内 source | 该脚本 | 仅这个 | 文件缺失即静默失效 |
| systemd + Environment | 该服务 | 仅这个 | 不展开变量、需 daemon-reload |

crontab 顶部直接声明最省事，一行一个 name = value。两个坑：值不会被 shell 展开，写 PATH = $PATH:/opt/bin 只会得到字面量；别加引号，引号会成为值的一部分。

脚本内 source 听着干净，实际最脆：路径一挪、权限一改、换台机器没同步，任务就静默失效。

## 五、systemd 那一栏不认美元符号

systemd 不经过 shell，Environment= 里的 $HOME 会原样当字面值传进去。正确写法是分行写完整值，值里含空格必须整体加引号。

变量多了改用 EnvironmentFile=，格式比想象的严格：一行一个 KEY=value，不写 export，值不加引号，只有以 # 开头的整行算注释，行尾不能跟注释。

两个容易忽略的点：改了单元文件必须 systemctl daemon-reload 再 restart，只 restart 读的还是旧配置；服务用 User= 切到别的账号时 HOME 跟着变，依赖 ~ 的相对路径会指到另一个目录。

## 六、被 no_proxy 绕过的本地客户端

客户端跑在本机 127.0.0.1:1080，而 no_proxy 又写着 127.0.0.1，这条代理会被自己绕过。no_proxy 的语义是访问这些目标时不走代理，它匹配的是目标主机，而代理服务器本身的地址也在匹配范围内。

麻烦在于 `no_proxy=localhost,127.0.0.1` 是极常见的默认值，不少镜像和现成脚本里就带着这一行。判断方法看 curl 的详细输出：显式 `-x` 指定代理时一定生效；换成靠环境变量跑，若看到的是「Trying 目标站 IP」而不是「Trying 127.0.0.1:1080」，就是被跳过了。解法是把 no_proxy 收窄，只留真正需要直连的内网段。

## 七、同一台机器上跑着两套变量名

| 工具 | 认哪些变量 | SOCKS5 写法 | 注意 |
|---|---|---|---|
| curl | 小写 http_proxy / https_proxy / all_proxy | socks5h:// | 忽略大写 HTTP_PROXY |
| requests / urllib3 | 大小写都读，含 NO_PROXY | socks5h:// | 需装 socks 支持 |
| Chromium 系 | 不读环境变量 | 启动参数 | 需参数或策略下发 |
| git | http.proxy 配置优先 | socks5h:// | 配置与环境变量会打架 |

curl 出于安全考虑忽略大写的 HTTP_PROXY，这是当年 httpoxy 漏洞的处置结果。只写大写的话，浏览器可能生效、curl 一定不生效，很容易误判成代理坏了。

协议前缀也要留意：写 socks5:// 是本机先解析域名再交给代理，写 socks5h:// 是把域名交给代理解析。做多账号业务时后者更稳，DNS 不会从服务器这边漏出去。

## 八、三步验证的具体命令

三步，不用在终端里替它试。

第一步看环境：加 `* * * * * env > /tmp/cron_env.txt 2>&1`，一分钟后读文件，找 https_proxy 与 no_proxy 在不在、值对不对。

第二步看出口：再挂 `* * * * * curl -s --max-time 15 https://api.ipify.org > /tmp/cron_ip.txt`，把文件里的地址与服务器公网 IP 对比，相同就是没走代理。用 curl 而不是 ping——ping 走 ICMP，多数客户端只实现 TCP 转发。

第三步在业务脚本开头打日志：把 os.environ 里键名带 proxy 的项打印出来，再留一条出口地址。

如果任务挂在宝塔面板的「计划任务」里，语义和 crontab 一样，底层还是 cron、还是 /bin/sh，区别只是日志能在面板直接看。别因为是从面板点进去的，就以为它会继承登录 shell 的环境。

## 九、官方入口与相关页面

| 用途 | 页面 |
|---|---|
| 🧰 客户端与配置教程 | [代理工具中心](https://socks5ip.com.cn/dailigongjuzhongxin/) ｜ [SSTap 配置教程](https://socks5ip.com.cn/dailigongju/sstapjiaocheng/) |
| 🔍 出口与测速自测 | [IP 综合检测中心](https://socks5ip.com.cn/ip-check-center/) |
| 💰 全平台比价 | [价格中心](https://socks5ip.com.cn/jiagezhongxin/) |
| 🚀 聚合开通 | [一站式注册中心](https://linkdd.cn/socks5ip) |

## 十、常见问题

**Q：手动跑正常，加到 crontab 就不走代理，最可能是哪一步？**
九成是代理变量没进 cron 环境。先跑那条 env 任务，文件里没有 https_proxy 就是它。把变量写进 crontab 顶部，或改用命令行带 env 的写法。

**Q：crontab 里能不能直接写 export？**
不能。crontab 用的是自己的变量行语法，写 name = value 就行，不能写 $PATH 这类展开，也别加引号。

**Q：宝塔面板的定时任务和 crontab 是一回事吗？**
执行环境是一回事，底层都是 cron、都是 /bin/sh，区别只在日志能在面板里看。所以代理变量一样要显式声明。

**Q：怎么确认脚本这一次真的走了代理？**
别在终端替它测。让任务自己把出口地址写进文件，和服务器公网 IP 比：一样就是没走，不一样才算通。

**Q：systemd 里配了 Environment 还是不生效？**
先 daemon-reload 再 restart。然后用 systemctl show -p Environment 看实际值，文件里写对不等于传进去对。

**Q：跑在 Docker 里的定时任务也要单独配吗？**
要。容器里也是独立环境，宿主机的变量不会自动进去，容器内的 cron 同样只拿四个变量。
