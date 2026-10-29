# VPS 被墙后，用 Cloudflare CDN 把它救回来

> 一次真实的排障记录：自建 VPS 的 IP 被整体阻断（能 ping 通，但所有 TCP 端口不通），
> 不换机器、不换 IP，用 Cloudflare 的 CDN 做中转，把服务救回来，并顺手堵掉了一串真实 IP 泄露。
>
> 本文所有 IP / 域名 / UUID / 订阅链接均已替换为占位符，可直接套用到你自己的机器。

---

## 0. 两种用法

**给人看**：从 §1 往下顺着读，照 §4-§5 手动配。适合想弄明白原理的人。

**交给 AI 代理**：把这个仓库地址直接发给 AI（Claude Code / Codex / Cursor / 任何能读网页的代理），让它先读 [`AGENTS.md`](AGENTS.md)。那份是写给机器执行的任务书：一次性信息收集表、预检项、每步的 verify 命令、失败分支、7 条验收门、回滚步骤。参数用 [`inputs.example.yaml`](inputs.example.yaml) 的模板填。

> **本仓库不是程序**，没有安装和运行步骤。拿到地址后不需要 `npm install`，AI 要做的是改配置 + 跑验证。

---

## 1. 适用场景

- 自建节点所在 IP 被墙，但**机器还在、还能通过商家面板操作**（比如 VNC / 网页控制台）
- 手里有**一个托管在 Cloudflare 的域名**，且该域名可以加代理记录（橙色云）
- 客户端是 **Clash / mihomo 系**（Clash Verge、Clash Party、Mihomo Party 等）
- 不想换 IP、不想换商家、不想重新买

如果你只是想要一个能用的出口、并不在乎"出口 IP 是不是自己的机器"，用 Cloudflare Worker 方案更省事；
但如果你需要有**固定的、自己控制的出口 IP**（很多站点的风控会看 IP 的类型和归属），CDN 中转是唯一能把出口维持在你自己机器上的做法。

---

## 2. 结论先说

| 问题 | 结果 |
|---|---|
| 被墙的 IP 还能用吗 | 直连不能。TCP 全端口超时，ICMP 仍然通 |
| 中转后能通吗 | 能。稳定可用 |
| 出口 IP 是谁 | **仍然是你自己那台机器**，不是 Cloudflare 的 IP |
| 延迟代价 | 首页首字节 ~1 秒级，可接受 |
| 需要改 VPS 上的原服务吗 | 不需要，新增一个入口即可，原入口原样保留 |
| 客服说"会解封"可信吗 | 不要等。方案要能撑到解封那天，并且解封后能一键切回 |

---

## 3. 架构

```
客户端 ──TLS(443)──> Cloudflare 边缘节点 ──内部骨干──> 你的 VPS:443
                     ▲                                  ▲
                     │                                  │
              域名解析到这里（橙色云代理）          只接受来自 CF 的 TLS+WS 流量
```

关键点：

1. **域名必须开橙色云**（Proxied）。灰云 = 直接解析到你的 IP = 一样被墙。
2. **CF 只能转发 TCP**（这里用的是 WebSocket over TLS）。**UDP 走不了**，这一点后面会咬人。
3. 回源端口用 443，证书用 Cloudflare 的 **Origin Certificate**，SSL 模式设 **Full (strict)**。

---

## 4. 服务端：加一个入口，不动原有服务

以 sing-box 的模块化配置为例（`sing-box -c config.json -C conf/`），**新增一个文件**，原入口文件保持不动：

`/etc/sing-box/conf/CF-WS-443.json`

```json
{
  "inbounds": [
    {
      "tag": "CF-WS-443",
      "type": "vless",
      "listen": "::",
      "listen_port": 443,
      "users": [{ "uuid": "<UUID>" }],
      "tls": {
        "enabled": true,
        "server_name": "<RELAY_HOST>",
        "certificate_path": "/etc/sing-box/cf/origin.crt",
        "key_path": "/etc/sing-box/cf/origin.key"
      },
      "transport": { "type": "ws", "path": "<WS_PATH>" },
      "sniffing": { "enabled": true, "destOverride": ["http", "tls", "quic"] }
    }
  ]
}
```

> **为什么"不动原有服务"很重要**：你原来的入站（比如 Reality）是解封后最好用的东西。
> 新增一个文件、备份一下 `conf/` 目录，将来解封你只需要切客户端，不用重新配置服务端。

Cloudflare 侧：

| 项 | 值 |
|---|---|
| DNS 记录 | `A` 记录，指向 VPS IP，**橙色云** |
| SSL/TLS 模式 | Full (strict) |
| 证书 | Origin Certificate（15 年），装到 VPS 上 |
| 端口 | 443（CF 支持的 HTTPS 端口，不要用非标准端口） |

---

## 5. 客户端：mihomo 节点

```yaml
proxies:
  - name: cf-relay
    type: vless
    server: <RELAY_HOST>     # 用域名，不要硬写优选 IP，理由见 §8.1
    port: 443
    uuid: <UUID>
    udp: false               # CF 中转不承载 UDP，这里必须 false
    tls: true
    servername: <RELAY_HOST>
    network: ws
    ws-opts:
      path: <WS_PATH>
      headers:
        Host: <RELAY_HOST>
```

配套的 DNS（**fake-ip 是必须的**）：

```yaml
dns:
  enable: true
  ipv6: false
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.1/16
  respect-rules: false
  # 中转域名本身不能走 fake-ip，否则连不上自己的服务器
  fake-ip-filter:
    - "*.lan"
    - "*.local"
    - "+.<YOUR_DOMAIN>"
    - "time.*.com"
    - "ntp.*.com"
    - "+.msftconnecttest.com"
    - "+.msftncsi.com"
  default-nameserver: [223.5.5.5, 119.29.29.29]
  proxy-server-nameserver: [223.5.5.5, 119.29.29.29]
  # 用加密 DNS，理由见 §8.4
  nameserver:
    - https://dns.alidns.com/dns-query
    - https://doh.pub/dns-query
  fallback:
    - https://dns.google/dns-query
    - https://cloudflare-dns.com/dns-query
  fallback-filter:
    geoip: true
    geoip-code: CN
```

分流规则骨架：

```yaml
rules:
  - DOMAIN,<RELAY_HOST>,DIRECT        # 中转域名自己必须直连，否则死循环
  # 明确要走代理的国外域名（按需补充）
  - DOMAIN-SUFFIX,claude.ai,Proxy
  - DOMAIN-SUFFIX,anthropic.com,Proxy
  - DOMAIN-SUFFIX,openai.com,Proxy
  - DOMAIN-SUFFIX,google.com,Proxy
  - DOMAIN-SUFFIX,github.com,Proxy
  # 国内走直连
  - GEOSITE,cn,DIRECT
  - DOMAIN-SUFFIX,cn,DIRECT
  - GEOIP,LAN,DIRECT
  - GEOIP,CN,DIRECT
  - MATCH,Proxy
```

> **必须确认 geoip.dat / geosite.dat 真的存在**。
> 我们遇到过一次"所有国内流量都走代理"，查了半天规则，结果是数据文件缺失，
> GEOSITE/GEOIP 规则全部失效，一路落到最后的 `MATCH,Proxy`。检查一下数据目录里
> `geosite.dat`、`geoip.dat`、`Country.mmdb` 三个文件在不在。

---

## 6. 两个开关别混淆

这是整件事里最容易搞错的地方。它们管的是**两件不同的事**：

- **开关一：怎么把流量抓进来** —— 系统代理 / 虚拟网卡（TUN）
- **开关二：抓到之后往哪送** —— 规则 / 全局

| 抓法 | 送法 | 国内网站 | 国外网站 | 会不会漏真实 IP |
|---|---|---|---|---|
| 系统代理 | 规则 | 快 | 走代理 | 会漏（不认系统代理的软件） |
| **虚拟网卡** | **规则** | **快** | **走代理** | **不漏** ← 推荐 |
| 系统代理 | 全局 | 慢 | 走代理 | 还是会漏 |
| 虚拟网卡 | 全局 | 慢到打不开 | 走代理 | 不漏 |

**"开虚拟网卡"不等于"全部走代理"。** 虚拟网卡只决定"所有软件都被接管"，
国内还是直连还是绕美国，取决于规则/全局这个开关。

实测代价（20 Mbps 美国线路，中国大陆家宽）：

| 站点 | 虚拟网卡 + 规则 | 虚拟网卡 + 全局 |
|---|---|---|
| 百度 | 179 ms | 1763 ms |
| 腾讯 | 162 ms | 失败 |
| B 站 | — | 3029 ms |
| 网易云音乐 | — | 失败 |

**结论：默认就用「虚拟网卡 + 规则」。** 全局只在"必须让国内站也看不到真实 IP"时才开，而且要接受上面的代价。

---

## 7. 怎么验证真的通了

别只看客户端显示"已连接"，要验证三件事：

```bash
# 1) 出口 IP 是不是你自己那台机器
curl -x http://127.0.0.1:7890 https://ipinfo.io/json

# 2) 走虚拟网卡（不设代理）时，出口是不是同一个
curl https://ipinfo.io/json

# 3) 目标站点自己看到的 IP（最权威）
curl https://claude.ai/cdn-cgi/trace | grep -E '^ip=|^loc=|^colo='
```

第 3 条最有用：`ip=` 就是目标站看到的地址，`colo=` 是 Cloudflare 给你的接入点，
`loc=` 是它判定的地区。三个对上了，才算真的通。

---

## 8. 六个踩坑（都有实测证据）

### 8.1 Cloudflare 优选 IP 在这条链路上没用

网上很多教程教你去测一批 Cloudflare 边缘 IP，挑延迟最低的硬写进客户端的 `server`。

**我们实测：没有意义。**

6 个候选边缘 IP + 域名默认解析（172.67.x.x），每个各跑 5 轮：

| 接入方式 | 节点延迟 | 谷歌中位 | GitHub 中位 |
|---|---|---|---|
| 104.17.51.32 | 210 ms | 1116 ms | 1662 ms |
| 104.27.86.117 | 188 ms | 1095 ms | 1300 ms |
| 104.19.84.39 | 202 ms | 1169 ms | 1537 ms |
| 104.27.121.238 | 216 ms | 1105 ms | 1385 ms |
| 104.19.154.110 | 193 ms | 1045 ms | 1535 ms |
| 104.16.97.221 | 210 ms | 1015 ms | 1393 ms |
| **直接用域名** | **275 ms** | **1091 ms** | **1532 ms** |

差异全部落在正常抖动范围内。有一次测得"延迟 1391 ms、GitHub 要 10~17 秒"，
一度以为是边缘节点选得差，回头复测才发现那是**刚部署完的瞬时状态**，
稳定后所有接入方式都在同一水平。

**结论：用域名，不要硬写 IP。** 硬写 IP 还有个副作用 —— Cloudflare 换 anycast 段之后
你的节点会直接失效，而用域名最多是重连一次。

### 8.2 虚拟网卡下，UDP 完全绕过分流规则（真实 IP 泄露的元凶）

这是最值得记的一条。

用 WebRTC 检测站一测，真实宽带 IP 被扒得干干净净。但同一个目标域名的 TCP 明明走了代理：

```
[TCP] <本机>:24669 --> 160.79.104.10:443  match Match using Proxy[cf-relay]      ← 走代理
[UDP] <本机>:56441 --> 160.79.104.10:443  doesn't match any rule using DIRECT    ← 直连出去了
[UDP] <本机>:58038 --> 162.159.207.0:3478 doesn't match any rule using DIRECT    ← STUN，就是它拿到真实 IP
```

`162.159.207.0:3478` 是 STUN（WebRTC 用来探测"我的公网 IP 是多少"的协议），
`doesn't match any rule` + `using DIRECT` 意思是：这个包**根本没进规则引擎**，直接送到真实网卡。

试过下面这些，**全部无效**：

| 尝试 | 结果 |
|---|---|
| 加规则 `NETWORK,UDP,Proxy` | 仍然 `doesn't match any rule` |
| 加规则 `IP-CIDR,<目标>/32,Proxy,no-resolve` | 仍然 `doesn't match any rule` |
| `tun.stack` 从 `mixed` 换 `system` | 仍然一样 |
| `find-process-mode` 关掉 | 仍然一样 |

**根因**：这台机器上用的这个内核版本，TUN 进来的 UDP 走的是另一条通路，不参与规则匹配。
（是内核行为问题，不是配置写错了。）

**修法：在操作系统层堵。** 用 Windows 防火墙，对浏览器的出站 UDP 只堵这几个远端端口：

```powershell
# 只堵 WebRTC/QUIC 相关端口，不碰 53（DNS）和 123（时间同步）
$ports = @('443','3478','5349','19302-19312')
foreach ($exe in @("$env:ProgramFiles\Google\Chrome\Application\chrome.exe",
                   "${env:ProgramFiles(x86)}\Microsoft\Edge\Application\msedge.exe")) {
  if (-not (Test-Path $exe)) { continue }
  $tag = [IO.Path]::GetFileNameWithoutExtension($exe)
  foreach ($p in $ports) {
    New-NetFirewallRule -DisplayName "BlockWebRtcUdp-$tag-$p" -Direction Outbound `
      -Program $exe -Protocol UDP -RemotePort $p -Action Block -Profile Any
  }
}
```

检测站随即从"可能泄露"变成"WebRTC 已禁用或无泄露"。

副作用：浏览器的 QUIC（HTTP/3）会退回 TCP，首屏可能慢一点点，功能不受影响。

### 8.3 "中国出口 IP"那张卡不是泄露，也改不动

很多 IP 检测站会显示两栏：一栏是"你的国外出口 IP"，另一栏是"你的中国出口 IP"。
看到后者显示你真实的国内 IP，很容易误判成"漏了"。

实际抓连接可以看到，那一栏是网页主动去问**国内的查 IP 服务**得到的：

```
2026.ip138.com    GeoSite(cn)   DIRECT
my.ip.cn          GeoSite(cn)   DIRECT
```

按分流规则，国内站直连，所以国内服务器当然看到你的真实 IP。
**这不是绕过，是你平时上淘宝、微信时同样会发生的事。**

试过把它们改成走代理：

| 动作 | 结果 |
|---|---|
| `ip138.com` 加进代理 | 该站确实看到了美国 IP，但页面上那栏没变（它还有别的数据源） |
| `ip.cn` 加进代理 | 对方服务器直接断开连接 —— 它**拒绝国外 IP 访问** |

**结论：只要保留"国内直连"，这一栏就必然是真实国内 IP。** 唯一能让它变美国 IP 的办法是
全局模式（所有国内流量也绕美国），代价见 §6 的表格。为了一个测试读数付这个代价，不值得。

### 8.4 明文 DNS 被路由器在 53 端口截胡

检测站有时会报"DNS 可能泄露"，显示的解析器是本地运营商。查下来是这样：

内核用**明文 UDP** 向 `223.5.5.5:53` 查询，而家用路由器/光猫普遍会在 53 端口做透明劫持，
把查询转给自己的上游（运营商 DNS）。于是权威服务器看到的就是运营商解析器。

**修法：把国内 nameserver 换成加密 DNS（DoH）**，走 443 端口，劫持不到：

```yaml
nameserver:
  - https://dns.alidns.com/dns-query
  - https://doh.pub/dns-query
default-nameserver: [223.5.5.5, 119.29.29.29]   # 仅用于引导解析上面两个域名
```

只换国内 nameserver 的话，权威服务器看到的仍然是**国内**解析器（阿里/腾讯），
所以检测站有时仍会判"中国 DNS"—— 这是加密了但没出国，属于预期行为。
要让解析器也显示国外，得让 `fallback` 的 DoH 查询真正走代理（`respect-rules: true`），
代价是国内域名解析会绕远，国内 CDN 命中率下降。按需取舍。

### 8.5 虚拟网卡设置不会自己记住

客户端重启内核后，`tun.enable` 会回到 false。节点选择一般会记住（存在内核的 `cache.db` 里），
但虚拟网卡不会。

**修法**：写一个一键恢复脚本，重启后双击即可 —— 重载配置、切规则模式、开 TUN、选中节点，并打印当前状态。

### 8.6 两个小坑

- **`.bat` 文件里不能写中文注释。** cmd 按 GBK 解析批处理文件，UTF-8 的中文会把整行命令吞掉，
  报一堆莫名其妙的"不是内部或外部命令"。批处理里只用 ASCII。
- **Cloudflare 上加错了账号的域名不会生效。** 域名 NS 指向 Cloudflare 不代表它在你当前登录的
  这个账号里。如果加完记录、等了半天解析还没变，先去域名的 Overview 页确认它确实在你账号下。

---

## 9. 自检清单

上线后逐条过一遍：

- [ ] `curl https://<目标站>/cdn-cgi/trace` 里的 `ip=` 是自己的 VPS IP
- [ ] 不设代理（走 TUN）时，出口 IP 也是同一个
- [ ] IPv6 不可达（否则会从这条路径漏出真实地址）
- [ ] 国内站 200 ms 以内（说明真的在直连，没绕美国）
- [ ] WebRTC 检测显示"已禁用或无泄露"
- [ ] DNS 解析结果是 198.18.x.x 这类假地址（说明 DNS 被客户端接管了，没走运营商明文）
- [ ] 重启客户端后，有脚本能一键恢复 TUN + 规则模式 + 节点选择

---

## 10. 说明

- 本文面向**自有服务器的运维与排障**，所有配置均针对你自己拥有或获授权管理的机器。
- 不同地区对网络接入设备与转发服务有各自的监管要求，公开发布此类教程可能带来合规风险，
  请自行评估是否适合公开传播。
- 文中提到的产品与商标归各自所有者，本文与 Cloudflare、Anthropic 等公司无隶属或背书关系。
- 数据均来自单机单线路实测，会随运营商、时段、线路质量变化，仅供参考。
