---
layout: post
title: "从手动登录到全自动：一次校园网（Dr.COM）自动认证、频段隔离与限速的折腾记录"
date: 2026-10-25 20:00:00 +0800
categories:
  - 网络
tags:
  - OpenWrt
  - iStoreOS
  - nftables
  - 校园网
  - Dr.COM
  - 自动化
  - 逆向
---

> **说明**：本文由 AI 生成，经 BG6HMA 校对后发布。

> 一句话总结：校园网门户每次连接都要手动登录，索性把它的认证协议拆开，在 Windows 和 iStoreOS 路由器上各做了一套自动登录；顺手把 2.4G 单独划成一个网段并限速，踩了 nftables、DHCP、DNS、nft-qos 的一连串坑。
>
> 文中账号、密码、MAC 等凭据均已打码处理。

<!-- more -->

## 0. 免责声明

- 本文是**技术记录**。校园网通常明确禁止"路由器 / 热点共享"，这样做的账号**真的会被封**（我的号就被封到次日凌晨 3:00）。
- 请先确认学校规定；有条件的话，走"一个账号登记多台设备"的正规流程，比和检测系统对抗划算得多。
- 文中所有凭据都做了打码处理，请勿把自己的密码提交到任何公开仓库或聊天记录里。

---

## 1. 起因

学校的校园网认证页是 `http://10.10.31.5/`。问题是它**不是每次连上都要求登录**——有时候能直接上网，有时候打开网页就被劫持到登录页，必须手动输账号密码。

目标很简单：**连上就自动登录，掉线自动重登**。

手上的环境：

| 设备 | 用途 |
| --- | --- |
| Windows 11 笔记本 | 日常主力机 |
| iStoreOS（Kwrt）路由器 | 打算让路由器自己认证，全屋共享 |

---

## 2. 先把登录协议拆开

### 2.1 门户只是个壳

直接访问 `http://10.10.31.5/a70.htm`，返回的 HTML 只有 1KB 多，真正的逻辑在 `a41.js` 里。顺着 JS 看下去，这套系统是 **Dr.COM（城市热点）ePortal**，认证接口是 JSONP 风格：

- 取配置：`GET /eportal/portal/page/loadConfig?program_index=`
- 登录：`POST /eportal/portal/login?<参数>`
- 注销：`POST /eportal/portal/logout?<参数>`

### 2.2 关键发现：一次性令牌 `rcn`

`loadConfig` 的返回里有一个 `rcn` 字段。它是**本次登录的会话令牌**，必须和登录请求一起提交。

这一点是整个折腾里最重要的发现——因为**不带 `rcn` 或令牌过期时，服务器返回的是**：

```json
{"result":0,"msg":"当前页面已超时,请重新刷新页面","ret_code":"1"}
```

这句话看起来像网络故障（"页面超时"），实际是**令牌失效**。我一开始就被它带偏过，以为是网络问题。

对照实验（用错误密码探测服务器真实反应）：

```text
不带 rcn        → 当前页面已超时，请重新刷新页面
带新鲜 rcn      → Radius检查账号/密码无效！
```

第二行才是"服务器真的在校验账号密码"。所以**正确流程永远是：先取新鲜令牌，再立刻提交登录**。

### 2.3 能跑通的最小命令

```bash
# ① 取一次性令牌
RCN=$(curl -s 'http://10.10.31.5:801/eportal/portal/page/loadConfig?program_index=' \
      | grep -o '"rcn":"[^"]*"' | head -1 | cut -d'"' -f4)
# => 8iXUnv9a

# ② 带令牌登录
curl -s -X POST \
  "http://10.10.31.5:801/eportal/portal/login?\
login_method=1\
&user_account=2651****03%40unicom\
&user_password=***\
&wlan_user_ip=172.27.x.x\
&wlan_user_ipv6=\
&wlan_user_mac=AABBCCDDEEFF\
&wlan_ac_ip=&wlan_ac_name=\
&jsVersion=4.1.3&terminal_type=1&lang=zh-cn\
&is_page=1&is_page_new=1234\
&rcn=$RCN"
```

成功的判定是响应里出现 `"result":1` 或 `"ret_code":2`：

```json
{"result":0,"msg":"IP: 172.27.x.x 已经在线！","ret_code":2}
```

`ret_code:2` 表示"该 IP 已经在线"，**等价于认证通过**，不是错误。

参数里有几个小坑：

| 参数 | 说明 |
| --- | --- |
| `wlan_user_ip` | 点分十进制（`172.27.x.x`）。而 `online_list` 接口要的是**整数**形式的 IP |
| `wlan_user_mac` | **大写、无冒号**（`aa:bb:cc:dd:ee:ff` → `AABBCCDDEEFF`） |
| `is_page` / `is_page_new` | 模拟"从页面发起的请求"，建议照抄 |
| `rcn` | 每次都要重新取，别缓存 |

---

## 3. 账号后缀：别相信门户页面上写的选项

门户页面的 JS 里有一段运营商配置：

```js
carrier='{"yys":{"data":[
  {"id":"1","name":"校园用户","suffix":""},
  {"id":"2","name":"校园电信","suffix":"@dx"},
  {"id":"3","name":"校园联通","suffix":"@lt"},
  {"id":"4","name":"校园其他","suffix":""}
]}}'
```

看起来"校园联通"就是 `@lt`。但我实测发现**这张表是过时的**。用正确密码逐个试：

| 提交的账号串 | 服务器返回 | 结论 |
| --- | --- | --- |
| `2651****03@unicom` | `登录策略检查完成` | ✅ 账号+后缀+密码全匹配 |
| `2651****03@dx` | `AC认证失败` | 后缀不适用 |
| `2651****03@lt` | `AC认证失败` | 页面上写的"联通"是错的 |
| `2651****03@cmcc` | `AC认证失败` | 后缀不适用 |
| `2651****03`（不带后缀） | `AC认证失败` | 必须带后缀 |
| `2651****03@unicom` + 错密码 | `Radius检查账号/密码无效` | 密码不匹配 |

**结论：后缀是账号属性，不是可以随便选的参数。**只能通过"能不能走到策略检查"这一步来反推。

> 教训：页面上的配置是"可能过时"的 UI 元数据，真正以服务器校验结果为准。

---

## 4. 一段插曲：门户显示的在线账号不是我的

调试过程中出现过一件诡异的事：门户主页（`http://10.10.31.5/`，源码里的 `uid=`）显示当前在线账号是 `2651****01@unicom`，而我的学号是 `...03`，并且**我这台机器的 IP 和 MAC 都对得上**。

当时的排查：

1. 用 `...01` + 我的密码 → `密码无效`（说明密码不属于这个号）
2. 用 `...03` + 我的密码 → 走到策略检查（说明密码属于 `...03`）
3. 换 cookie、伪造 `X-Forwarded-For`、换 MAC 参数 → 门户始终返回同一个 `uid`

也就是说：**门户认为在线的号不是我的号**，但那台机器的 IP/MAC 确实是这台电脑的。

后来账号解封后，门户重新显示 `...03`，这个异常自己消失了。它的成因我至今没有证据能解释（可能是封禁机制留下的会话残留，也可能是 AC 侧的记录问题）。

**这段经历真正的价值是教训**：

- 我一开始看到"实名上网管控提醒 + 封禁至次日 3:00"，就直接下了"你的账号被封"的结论——**过度推断**。那条提示只是"某一次请求"的返回，不等于账号的全局状态。
- 单一 API 响应不能当结论；要区分"接口对这次请求说了什么"和"账号现在是什么状态"。
- 拿着用户的密码去反复试登录，还会触发门户的限流（后面就遇到连续 `当前页面已超时`），把环境搞得更差。

---

## 5. Windows 端：让电脑自己登录

### 5.1 调度方式

最早用的是"隐藏窗口常驻进程"，后来发现**父进程退出会把子进程带走**，于是改成 **Windows 计划任务**：

- 登录时触发（延迟 15 秒，等网络就绪）
- 每 N 分钟跑一次 `-Action once`（跑完即退，不常驻）
- 进程崩了由任务计划程序重跑

这个"跑一次就退出"的模式比常驻进程可靠得多：**任务边界清晰，不会积累僵尸进程**。

### 5.2 黑框闪窗：`-WindowStyle Hidden` 是不够的

计划任务每 2 分钟呼叫一次 `powershell.exe`，即使加了 `-WindowStyle Hidden`，**还是会闪一下黑框**。

Windows 上真正无窗口的办法是**用 VBS 的 WScript.Shell 以窗口模式 0 启动**：

```vbs
' run-hidden.vbs
Set sh = CreateObject("WScript.Shell")
sh.Run "powershell.exe -NoProfile -ExecutionPolicy Bypass -File ""C:\path\campusnet.ps1"" -Action once", 0, False
```

计划任务的动作改成：

```text
wscript.exe  "C:\path\run-hidden.vbs"
```

这样连闪都不闪。

### 5.3 核心逻辑（简化）

```powershell
$ip  = (Get-NetIPAddress -AddressFamily IPv4 | Where-Object { $_.IPAddress -notlike '127.*' }).IPAddress
$mac = (Get-NetAdapter | Where-Object Status -eq 'Up').MacAddress   # 大写下划线，需再处理成无冒号

# 每次登录前重新取令牌 —— 这是整个工具的灵魂
$cfg = Invoke-WebRequest 'http://10.10.31.5:801/eportal/portal/page/loadConfig?program_index='
$rcn = ([regex]'"rcn":"([^"]+)"').Match($cfg.Content).Groups[1].Value

$q = "login_method=1&user_account=$acct&user_password=$pwd" +
     "&wlan_user_ip=$ip&wlan_user_mac=$mac&jsVersion=4.1.3" +
     "&terminal_type=1&lang=zh-cn&is_page=1&rcn=$rcn"

$r = Invoke-WebRequest "http://10.10.31.5:801/eportal/portal/login?$q" -Method POST
$ok = $r.Content -match '"result":1|"ret_code":2'
```

### 5.4 编码坑

脚本里有中文注释时，**保存成 UTF-8 无 BOM** 会让 Windows PowerShell 5.1 按 GBK 解析，直接语法错误。加 BOM 就好。而 `.cmd` 启动器反过来——**纯 ASCII + CRLF** 最稳（BOM 和 LF 都会让 `cmd.exe` 解析出错）。

---

## 6. 路由器端：iStoreOS 手动装插件

### 6.1 先摸清环境

```sh
cat /etc/openwrt_release
# DISTRIB_ID='Kwrt' / DISTRIB_RELEASE='24.10-SNAPSHOT'
# DISTRIB_TARGET='mediatek/filogic'（MT7981/7986 这类）

opkg print-architecture   # aarch64
uname -m                  # aarch64
```

实际包用的架构名是 **`aarch64_cortex-a53`**。

### 6.2 软件源不通 → 手动下 ipk

路由器的源用不了，改从清华镜像手动下载（国内访问官方源经常超时）：

```sh
M=https://mirrors.tuna.tsinghua.edu.cn/openwrt/releases/24.10.0/packages/aarch64_cortex-a53

# 确认包名（版本号必须完全对上）
curl -s "$M/packages/" | grep -o 'href="[^"]*nft-qos[^"]*"'
curl -s "$M/luci/"     | grep -o 'href="[^"]*nft-qos[^"]*"'

curl -sO "$M/packages/nft-qos_1.0.6-r4_all.ipk"
curl -sO "$M/luci/luci-app-nft-qos_26.283.50386~24db9bf_all.ipk"
curl -sO "$M/luci/luci-i18n-nft-qos-zh-cn_26.283.50386~24db9bf_all.ipk"
```

`_all` 结尾的包与架构无关，这点很关键——跨版本安装的成功率更高。

### 6.3 安装时遇到的两个问题

**① opkg 锁被占用**

```text
opkg_conf_load: 有任务在执行中, 请稍后再试.: /var/lock/opkg.lock
```

不是锁文件残留，而是 **iStore 应用商店正在后台装东西**（`ps | grep opkg` 能看到）。等它装完就好，**别急着删锁**。

**② 缺内核模块依赖**

```text
cannot find dependency kmod-nft-netdev for nft-qos
cannot find dependency kmod-nft-bridge for nft-qos
```

这两个是内核模块，跨版本装不上是正常的。好在 `nft-qos` 本体是脚本集合，缺它们也能跑（用不到网桥层过滤时无影响）。

---

## 7. 限速失效的三次踩坑（本文最有价值的部分）

装上 `nft-qos` 后配了"按网段限速"，规则看起来生成了，**但完全没效果**。前后踩了三个坑，一个比一个隐蔽。

### 坑 1：一个 IPv6 占位符，让整张表加载失败

`nft-qos` 的配置文件里带了示例值：

```text
option dynamic_cidr6 'AAAA:BBBB::1/64'
```

这串东西被原样写进了生成的规则文件：

```nft
ip saddr AAAA:BBBB::1/64 limit rate over 78 kbytes/second drop
```

`nft` 把 `AAAA` 当成**主机名**去解析，于是：

```text
Error: Could not resolve hostname: Name has no usable address
加载退出码=1
```

**因为 `nft -f` 是整文件原子加载，这一行报错 → 整张表都没进内核 → 限速完全不存在。**

而 UI 上看起来一切正常：配置项都在、规则文件也有内容。只有 `nft list tables` 里少了那张表。

修复：删掉 `dynamic_cidr6`（只做 IPv4 限速就不需要它）。

### 坑 2：流量分载（flow offloading）绕过 netfilter

修好语法后规则终于进内核了，但速度还是没变。原因是：

```text
firewall.@defaults[0].flow_offloading='1'       # 软件分载
firewall.@defaults[0].flow_offloading_hw='1'    # 硬件分载
```

**分载会让数据包走"快速路径"绕过 netfilter**，nft 的规则形同虚设。这是这类限速最常见的失效原因，尤其是在 MediaTek 这种带硬件加速的平台上。

```sh
uci set firewall.@defaults[0].flow_offloading='0'
uci set firewall.@defaults[0].flow_offloading_hw='0'
uci commit firewall
```

### 坑 3：dynamic 模式的 monitor 会不断重建表

又过了几天，限速"自己好了又自己没了"。日志里有大量：

```text
nft-qos-monitor: ACTION=update, IPADDR=192.168.24.56, HOSTNAME=...
nft-qos-dynamic: ACTION=update, ...
```

我的判断（没有源码级证据）：dynamic 模式会为每台在线设备持续 `update`，**每次重建整张表**；一旦某次重建被打断或失败，表就变成空的——而配置文件和 UI 上完全看不出来。

### 最终方案：放弃 app，写静态 nft 规则

既然 `nft-qos` 是"动态生成 + 反复重建"，那就换成**声明式**的固定规则，放进 `/etc/nftables.d/`（fw4 每次启动都会包含它）：

```nft
# /etc/nftables.d/20-limit-24g.nft
chain limit24g_upload {
    type filter hook postrouting priority 0; policy accept;
    ip saddr 192.168.24.0/24 limit rate over 125 kbytes/second counter drop
}
chain limit24g_download {
    type filter hook prerouting priority 0; policy accept;
    ip daddr 192.168.24.0/24 limit rate over 250 kbytes/second counter drop
}
```

几点说明：

- **`/etc/nftables.d/*.nft` 里的内容是"链"**，会被内联进 `table inet fw4`（对比一下你系统里已有的 `12-mangle-ttl-128.nft` 就明白了）。
- `limit rate over X` 是**丢包式限速**（超过就丢），不是平滑整形。优点是配置简单、不依赖 tc；缺点是 TCP 会重传、延迟抖动，实际吞吐会略低于标称值。
- 加 `counter` 是为了能验证：`nft list table inet fw4 | grep "limit rate over"` 能看到丢弃计数在涨，就是真的在工作。
- 单位换算：**Mbps × 125 = kbytes/second**（2 Mbps → 250 kB/s）。

顺手写了个小工具方便调速度：

```sh
# /usr/bin/qos24g  用法: qos24g <下行Mbps> <上行Mbps>
D=$1; U=$2
printf "chain limit24g_upload {\n ... limit rate over %d ..." $((U*125)) $((D*125)) \
  > /etc/nftables.d/20-limit-24g.nft
fw4 reload
```

> 小坑：OpenWrt 默认 `PATH` 里**没有 `/usr/local/bin`**，脚本放那儿会 `command not found`，需要放到 `/usr/bin` 或建软链。

### 关于 `nft-qos` 的单位换算

这也是个坑。实测（`dynamic_bw_down` → 生成的 `limit rate over`）：

| 界面填的值 | 生成的限速 | 实际 |
| --- | --- | --- |
| `10` | `156 kbytes/second` | ≈1.2 Mbps |
| `80` | `1250 kbytes/second` | 10 Mbps |

倍数约 15.6，**相当于把 Mbps 多除了一次 8**。所以这个界面不能按"Mbps"直觉填——填 10 只会得到 1.2 Mbps。想限 10 Mbps 得填 80。

（也遇到过填 8 得到 142 kB/s 的情况，倍数又不一样，说明换算不稳定。这也坚定了我改用静态规则的决心。）

---

## 8. 2.4G 单独限速、5G 不限：把频段拆成两个网段

需求：**只限制 2.4G 频段的所有设备，5G 不受影响。**

`nft-qos` 是按 IP/MAC 限速的，**看不到设备连的是哪个频段**。所以只能从网络结构下手：让 2.4G 变成一个独立的二层网络。

### 改造前

```text
2.4G SSID  ┐
5G   SSID  ├── br-lan ── 10.0.0.0/24
有线口      ┘
```

### 改造后

```text
2.4G SSID  ── phy0-ap0 ── br-24g ── 192.168.24.0/24 ── 独立防火墙区域（限速）
5G   SSID  ── phy1-ap0 ─┐
有线口      ── lan3/lan4 ┴─ br-lan ── 10.0.0.0/24（不限速）
```

核心几步（UCI）：

```sh
# 1) 新建 2.4G 网桥
uci add network device
uci set network.@device[-1].name='br-24g'
uci set network.@device[-1].type='bridge'
uci set network.@device[-1].ports='phy0-ap0'

# 2) 新建接口（⚠️ 必须写 device，否则网桥不会创建）
uci add network interface
uci set network.@interface[-1].device='br-24g'
uci set network.@interface[-1].proto='static'
uci set network.@interface[-1].ipaddr='192.168.24.1'
uci set network.@interface[-1].netmask='255.255.255.0'

# 3) 从 br-lan 的 ports 里摘掉 phy0-ap0（⚠️ 忘了这步会两边冲突）
# 4) 新建 DHCP 段，interface='24g'
# 5) 新建防火墙区域 wifi24g + forwarding 到 wan
# 6) 无线绑定：uci set wireless.default_radio0.network='24g'
```

### 踩的两个坑

**坑 A：接口没写 `device`**

只加了 `network` 段、没写 `device='br-24g'`，结果 `br-24g` 根本没被创建，2.4G 网卡"悬空"——设备连上又断开，日志里反复出现：

```text
hostapd: phy0-ap0: AP-STA-DISCONNECTED ...
```

**坑 B：同一个网卡被两个网桥声明**

`phy0-ap0` 还留在 `br-lan` 的 `ports` 里，同时又被 `br-24g` 声明。内核只能认一个，结果 `br-24g` 建不起来。必须从 `br-lan` 里显式删掉。

**坑 C：匿名段的名字不是你想的那个**

用 `uci add network interface` 建的段，名字是自动生成的（比如 `cfg096d96`），而不是你以为的 `24g`。所以：

```sh
uci set dhcp.@dhcp[-1].interface='24g'   # 绑不上！
```

结果 `br-24g` 有 IP，但 **DHCP 不下发地址**，2.4G 设备拿不到 IP。

修复：把接口段重命名成 `24g`，让网络接口名、DHCP 的 `interface`、防火墙区域的 `network` 三者一致：

```sh
uci rename network.cfg096d96='24g'
uci commit network
```

### 验证方法

```sh
ip -4 addr show br-24g                     # 应有 192.168.24.1/24
ls /sys/class/net/br-24g/brif/             # 应只有 phy0-ap0
ls /sys/class/net/br-lan/brif/             # 应只剩 lan3 lan4 phy1-ap0
cat /tmp/dhcp.leases                       # 2.4G 设备应拿到 192.168.24.x
```

---

## 9. DNS：`allservers` 的隐藏陷阱

校园网下发的 DNS 是 `59.74.x.x`，这种运营商 DNS 在某些环境下会被污染。于是想换成公共 DNS。

但**只加 `server` 是不够的**。先看原来的配置：

```text
dhcp.@dnsmasq[0].allservers='1'
```

`allservers` 对应 dnsmasq 的 `--all-servers`：**同时向所有上游发查询，谁先回用谁**。

问题在于：校园网 DNS 在局域网内，**响应通常最快**，所以被污染的答案反而会抢先被采用——你新加的干净 DNS 根本轮不到。

正确做法是**彻底踢掉 WAN 下发的上游**：

```sh
uci set dhcp.@dnsmasq[0].noresolv='1'      # 不使用 /etc/resolv.conf 里的上游
uci set dhcp.@dnsmasq[0].allservers='0'    # 关掉"谁快用谁"
uci -q delete dhcp.@dnsmasq[0].server
uci add_list dhcp.@dnsmasq[0].server='223.5.5.5'
uci add_list dhcp.@dnsmasq[0].server='119.29.29.29'
uci commit dhcp
/etc/init.d/dnsmasq restart
```

验证：

```sh
cat /tmp/resolv.conf          # 应该只剩 nameserver 127.0.0.1
nslookup www.baidu.com 127.0.0.1
```

顺带一提：实测 `223.6.6.6` 在这个网络里**超时不可用**，而 `223.5.5.5`、`119.29.29.29` 正常。所以公共 DNS 也得先逐个测通再写进配置——**否则 `noresolv=1` 之后上游全挂，会直接断网**。

---

## 10. TTL 统一，以及"防检测"的边界

路由器上原本就有一条现成的 TTL 规则：

```nft
# /etc/nftables.d/12-mangle-ttl-128.nft
chain custom_ttl_set {
    type filter hook postrouting priority 300; policy accept;
    oifname "wan" ip ttl set 128
    oifname "wan" ip6 hoplimit set 128
}
```

这里有个容易误解的点：**hook 链不需要 `jump`**。只要写了 `type filter hook postrouting`，它就会自动挂到那一层，和 `jump` 到普通链是两种机制。

统一 TTL 能抹掉"Windows 128 / 手机 64"的差异——因为路由器转发本来就会减 1，把值重写成一个固定值之后，外部看到的就是一致的。

但必须诚实说明它能做什么、不能做什么：

| 检测手段 | TTL 规则能解决吗 |
| --- | --- |
| IP TTL 差异 | ✅ 能 |
| MSS / TCP 选项差异 | ✅ 大致能（配合 MSS clamp） |
| **TLS 指纹（JA3/JA4）** | ❌ 不能，属于应用层 |
| **并发连接数 / 流量形态** | ❌ 不能 |
| **认证侧审计（一个账号的会话）** | ❌ 不能 |

所以：**能降低被判定为"共享"的概率，但做不到隐身。** 而且更朴素的检测方式是——网管在交换机上看到某个宿舍口插了台路由器。这个任何规则都防不住。

---

## 11. 定时重启：别为了"清内存"而清

想加个"定时重启清理内存"。先看数据：

```text
MemTotal:      490 MB
MemAvailable:  332 MB     ← 可用
used:          107 MB     ← 实际使用（还含 66MB 文件缓存）
uptime:        1 天 6 小时
load average:  0.00
```

**内存非常宽裕，不存在"需要清理"的压力。** Linux 的 buff/cache 会在需要时自动回收，不是泄漏。

所以做成了**条件重启**（只在真紧张时重启），而不是无条件每天重启：

```sh
# /etc/crontabs/root
30 4 * * * AVAIL=$(awk '/MemAvailable/{print $2}' /proc/meminfo); \
  if [ "$AVAIL" -lt 61440 ]; then logger -t auto-reboot "low mem ${AVAIL}kB"; reboot; fi
```

### 一个容易埋雷的细节

在 Windows 上用单引号生成 shell 命令写进 crontab 时，反斜杠会被**原样写入文件**：

```text
... awk "/MemAvailable/{print \$2}" ...
```

这里 `\$2` 在双引号里恰好等价于 `$2`，所以**这次是能跑的**；但这类"转义污染"很容易在别的地方变成静默失败（任务存在、永远不触发、也不报错）。

**更稳的做法**：用 here-doc 写文件，写完立刻 `grep` 核对内容（确认没有多余的反斜杠/转义），再单独跑一遍判断逻辑验证。

---

## 12. 游戏打不开 ≠ DNS 污染

一个很典型的误判：

> 现象：视频能刷，但原神卡在登录界面。

一开始怀疑 DNS 被污染。但这个假设很快站不住：

1. **DNS 污染会同时打死视频和游戏**（两者都要解析域名），而视频是正常的。
2. 用干净 DNS 和校园网 DNS 解析同一个域名，结果一致（而且我选的测试域名本身是 NXDOMAIN，测试方法就是错的）。
3. **对照实验**：电脑上开代理 → 游戏能进；关代理 → 卡登录。

结论：**是校园网对游戏服务器/协议的封锁，不是 DNS 污染。** 代理能解，因为它是完整隧道。

这件事的教训是方法论的：**先做对照实验（开/关代理、换网络），再猜原因。** 单看"网页能开、某应用不行"，能列出十种可能，逐个验证会浪费大量时间。

顺带把 MTU、NAT/UPnP、passwall 都排查了一遍：

- WAN MTU 1500，没有 PPPoE 头开销；
- `miniupnpd` 在跑，但配置是 `internal_iface='lan'`，**新加的 2.4G 网段不在服务范围**（如果要联机，需要把它加进去）；
- passwall 装了但 `enabled=0`，没有残留规则。

---

## 13. 最后的架构与清单

```text
                 ┌──────────────┐
校园网 10.10.31.5 │  ePortal 认证 │
                 └──────┬───────┘
                        │  HTTP（rcn + login）
                 ┌──────┴───────┐
                 │   iStoreOS   │
                 │  campusnet   │← procd 服务，30s 巡检，掉线重登
                 │  nftables    │← TTL 统一 / 2.4G 限速
                 │  dnsmasq     │← 干净上游 DNS
                 └──┬────────┬──┘
                    │        │
        br-24g      │        │      br-lan
    192.168.24.0/24 │        │    10.0.0.0/24
    2 Mbps 限速     │        │    不限速
       2.4G 设备     │        │    5G / 有线
```

最终清单：

| 功能 | 实现 | 持久化 |
| --- | --- | --- |
| 自动登录 | shell 脚本 + procd 服务 | `/etc/init.d/campusnet` |
| TTL 统一 | nftables.d 链 | `/etc/nftables.d/12-mangle-ttl-128.nft` |
| 2.4G 隔离 | 独立网桥/网段/防火墙区域 | UCI 配置 |
| 2.4G 限速 | 静态 nft 规则 | `/etc/nftables.d/20-limit-24g.nft` |
| 调速度 | `qos24g <下行> <上行>` | `/usr/bin/qos24g` |
| 干净 DNS | dnsmasq `noresolv` + 显式 server | UCI 配置 |
| 条件重启 | cron | `/etc/crontabs/root` |

---

## 14. 踩坑清单（速查）

| # | 坑 | 现象 | 修复 |
| --- | --- | --- | --- |
| 1 | 缺少 `rcn` 令牌 | `当前页面已超时` | 每次登录前重新取 |
| 2 | 门户的运营商表过时 | `AC认证失败` | 实测反推真实后缀 |
| 3 | `nft` 里的 IPv6 占位符 | 整表加载失败，静默无限速 | 删除 `dynamic_cidr6` |
| 4 | flow offloading | 规则在但不生效 | 关闭软/硬分载 |
| 5 | `nft-qos` dynamic monitor | 规则"自己消失" | 改用静态规则 |
| 6 | `nft-qos` 单位换算 | 填 10 只有 1.2 Mbps | 用 nft 直接写 kB/s |
| 7 | 网桥接口漏写 `device` | `br-24g` 不创建 | 补 `device` |
| 8 | 同一网卡归属两个网桥 | 网桥起不来 | 从 `br-lan` 移除 |
| 9 | 匿名段名与引用名不一致 | 2.4G 拿不到 IP | `uci rename` 统一命名 |
| 10 | `allservers=1` | 污染 DNS 抢先 | `noresolv=1` + 显式 server |
| 11 | `-WindowStyle Hidden` | 计划任务闪黑框 | VBS `Run(...,0,False)` |
| 12 | Windows PS 5.1 读无 BOM UTF-8 | 中文注释导致语法错误 | 加 BOM |
| 13 | crontab 转义污染 | 任务永不触发且不报错 | here-doc + 写后核对 |
| 14 | `PATH` 不含 `/usr/local/bin` | `command not found` | 放 `/usr/bin` 或建软链 |

---

## 15. 写在最后

这次折腾里，真正花时间的不是"写脚本"，而是**判断每一个现象背后是什么**：

- "页面已超时"不是网络问题，是令牌问题；
- "规则没生效"不是配置写错，而是整张表因为一行 IPv6 占位符没加载；
- "网页能开、游戏不行"不是 DNS 污染，是协议层封锁；
- "规则用了几天就没了"不是玄学，是动态重建机制的副作用。

以及一个更重要的教训：**别把一次接口的返回当成全局结论。** 我曾因为一条"热点共享封禁"的提示，直接下结论说账号被封，而事实是那次请求确实被拦、但账号状态和这个号段的关系远比一条 JSON 复杂。

最后再强调一次：校园网禁止共享是**明文规定**，本文只是记录"技术上它是怎么工作的"。真要长期稳定用网，去问学校能不能登记多设备、或者宿舍有没有独立的有线口——那比每天和检测系统斗智斗勇划算得多。

---

*如果这篇文章帮你少踩一个坑，那就值了。*
