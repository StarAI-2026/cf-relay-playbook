# AGENTS.md — 面向 AI 代理的执行规格

> 把这个文件当作你的任务书。你的任务是：在**用户的机器**上落地本方案，直到 §8 的验收门全部通过。
> 本文件是权威；`README.md` 是给人看的背景与实测数据，两者冲突时以本文件为准。

---

## 0. 先理解这是什么东西

**这不是一个程序。** 没有 `install`、没有依赖、没有可执行文件、没有 `npm run`。

它是一份「配置改动 + 排障知识」的规格。你要做的是改配置文件、跑命令、验证。
用户拿到仓库后不该"运行"任何东西，而是照着你的输出操作。

**不要**执行 `pip install` / `npm install` / `docker compose up` 之类。本仓库没有任何代码。

### 占位符对照

| 占位符 | 含义 | 谁来定 |
|---|---|---|
| `<VPS_IP>` | 被墙那台机器的公网 IP | 问用户 |
| `<RELAY_HOST>` | 中转用的完整主机名，如 `vps.example.com` | 问用户 |
| `<ZONE>` | 该主机名所属的根域名，如 `example.com` | 问用户 |
| `<UUID>` | 节点 UUID | 你生成（`uuidgen`） |
| `<WS_PATH>` | WebSocket 路径，如 `/a1b2c3d4` | 你生成（8 位随机） |
| `<CF_TOKEN>` | Cloudflare API Token | 问用户 |
| `<ZONE_ID>` | 由 Token 查出来，不要问用户 | 你查（§4-P2） |
| `<CLIENT_DIR>` | 客户端真实数据目录 | 你探测（§4-P4） |
| `<ORIGINAL_PORT>` | 原节点入站端口（验收要确认它还在） | 问用户或 `ss -lntp` 看 |
| `<CERT_DIR>` | 证书目录：sing-box 用 `/etc/sing-box/cf`，xray 用 `/usr/local/etc/xray/cf` | 按服务端栈定 |

---

## 1. 开工前：一次性问全，不要挤牙膏

把下面这份**整段**发给用户，让他一次答完。答不上来的项，给他一条命令或一个截图位置。

```
我要按方案把你这台被墙的机器救回来。下面 11 项一次答完我就开工，中间不再打扰你。
不知道的直接写"不知道"，我会自己查。

【机器】
1. VPS 公网 IP：____
2. 你现在还能不能连上这台机器？
   (a) 我还有别的梯子能用 → 本地代理端口是：____（在软件设置里看，常见 7890）
   (b) 完全没有梯子，只能打开商家网页控制台（VNC）：能打开 / 打不开
3. SSH 端口、用户名、密码或密钥路径：____
   （私聊内容；事后建议改密码。若你只会用面板，这题跳过）

【现状】能把这段贴进 SSH 或面板终端最好，不能就跳过：
cat /etc/os-release | head -3
ls /etc/sing-box/conf /usr/local/etc/xray 2>/dev/null
systemctl is-active sing-box xray 2>/dev/null
ss -lntp | grep -E ':(443|8443|2053|21565) '

【域名与 Cloudflare】
4. 中转用的域名（如 vps.example.com）：____
5. 它所属的根域名（如 example.com）：____
6. 这个域名确实在你现在登录的 Cloudflare 账号里吗：是 / 不确定
7. Cloudflare 怎么办：
   (a) 我给你 API Token（推荐，我全自动做完）
   (b) 不给 Token，你告诉我点哪里，我自己点

【客户端】
8. 你的电脑是 Windows 还是 macOS？
9. 用的什么客户端：Clash Verge / Clash Party / Mihomo Party / sing-box 其他？
10. 是否便携版（解压即用、装在非默认目录）：是 / 否

【授权】
11. 允许我加一条 Windows 防火墙规则吗（堵浏览器的一个真实 IP 泄露口，需要管理员权限）？
    原来的节点入站要保留吗（强烈建议保留，以后解封能直接切回）：保留 / 不保留
```

**不要**把密码、Token、UUID 写进任何会公开的文件或提交信息。

---

## 2. 术语与边界

- **中转**：`<RELAY_HOST>` → Cloudflare（橙色云）→ VPS 的 443（VLESS + WS + TLS）。
- **出口 IP 仍是用户的 VPS**，不是 Cloudflare 的。这是选本方案而非 Cloudflare Worker 的**唯一**理由。
- **Cloudflare 免费版只转发 TCP**，UDP 走不了 → 节点必须 `udp: false`。
- 本方案**只新增入口，不改原有服务**。原有入站原样保留，解封后一键切回。

---

## 3. 客户端是无 GUI 代理时的降级原则

很多 AI 代理只能改文件、跑命令，不能操作用户的浏览器。遇到下面这些必须交给用户，**明确说"这一步需要你操作"**，不要假装自己做完了：

| 事项 | 谁做 |
|---|---|
| Cloudflare 面板登录、Origin 证书创建（无 API Token 时） | 用户 |
| 商家 VNC 控制台粘贴脚本 | 用户 |
| Windows UAC 授权 | 用户点"是" |
| 浏览器里验证 claude.ai 能打开 | 用户 |
| 检测站看 WebRTC/信任分 | 用户，或你用无头浏览器自己看 |

---

## 4. 预检（动手前做完；任何一项不过就停下来报告，不要硬上）

### P1 — 确认真的被墙，而不是服务挂了

| 现象 | 结论 |
|---|---|
| ICMP 通，但 22/80/443 等**所有** TCP 端口超时 | IP 级阻断 → 本方案适用 |
| ICMP 通，个别端口不通 | 防火墙/服务问题 → **不要**上中转，先修服务 |
| ICMP 也不通 | 机器宕机或路由问题 → 先找商家 |

判定要用**境外**出口反查（在线 tcping 类工具；注意某些站点的 `/ping/` 走 ICMP 会误报全绿，要用 TCP 检测）。
境外能连 22、境内连不上 = 确认被墙。

### P2 — 确认域名在当前操作的 Cloudflare 账号里

```bash
export CF_TOKEN='<CF_TOKEN>'
export ZONE='<ZONE>'
curl -s -H "Authorization: Bearer $CF_TOKEN" \
  "https://api.cloudflare.com/client/v4/zones?name=$ZONE" \
  | jq -r '.result[] | "\(.id) \(.name) \(.status)"'
export ZONE_ID=$(curl -s -H "Authorization: Bearer $CF_TOKEN" \
  "https://api.cloudflare.com/client/v4/zones?name=$ZONE" | jq -r '.result[0].id')
echo "ZONE_ID=$ZONE_ID"
```

返回空 = 域名 NS 虽指向 Cloudflare，但**不在这个账号**。停下来让用户换域名或换账号。

### P3 — 识别服务端软件

```bash
ls /etc/sing-box/conf 2>/dev/null && echo "→ sing-box 模块化"
ls /usr/local/etc/xray 2>/dev/null && echo "→ xray"
systemctl is-active sing-box xray 2>/dev/null
```

**先确认再动手。** 两者配置格式完全不同，见 §6。

### P4 — 找客户端真实数据目录

便携版的配置常在安装目录下的 `data/`，而 `%APPDATA%` 里可能只是残留：

```bash
ls -la "<安装目录>/data/"
ls -la "$APPDATA/" | grep -i -E 'clash|mihomo|verge'
```

判据：哪个目录里有 `profiles/`、`work/config.yaml` 且**最近有修改时间**。改错目录会让你以为配置没生效。

### P5 — 端口占用与端口选择

```bash
ss -lntp | grep -E ':(443|8443|2053) '
```

**Cloudflare 可回源的 HTTPS 端口只有这几个**：`443`、`2053`、`2083`、`2087`、`2096`、`8443`。
不要用别的端口，CF 不会转发。

决策：

| 现状 | 做法 |
|---|---|
| 443 空着 | 新入口用 **443** |
| 443 已被原入站占用（常见） | 新入口改用 **8443** 或 **2053**，**不要**动原入站 |
| 443 被 nginx/caddy 占用 | 保持占用，新入口用 8443，或让用户在 CF 侧用不同主机名分流 |

### P6 — 这个 zone 下还有别的主机名吗

`ssl` 是 **zone 级**设置，改了会连带影响到同一根域名下**所有**橙色云主机名。
如果别的站点源站用的是自签证书，把它们一起打成 526。

```bash
curl -s -H "Authorization: Bearer $CF_TOKEN"   "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/dns_records?proxied=true&per_page=100"   | jq -r '.result[] | "\(.type) \(.name) -> \(.content)"'
```

| 结果 | 做法 |
|---|---|
| 只有 `<RELAY_HOST>` 这一条 | 放心用 `strict` |
| 还有别的主机名 | **先问用户**那些站的源站证书是否有效；不确定就改用 `full`（非 strict），或换一个专用子域名/专用 zone |
| 用户不清楚 | 用 `full`，并在报告里写明这是一处安全折衷 |

---

## 5. 服务器访问：两条路，先判定

IP 被墙时用户本机通常**无法直接 SSH**。

### 分支 A — 用户还有别的可用代理（优先）

境外出口能连上 VPS，就可以走 HTTP CONNECT 隧道 SSH：

```python
import socket, paramiko
sock = socket.create_connection(("127.0.0.1", <PROXY_PORT>))
sock.sendall(f"CONNECT {VPS_IP}:22 HTTP/1.1\r\nHost: {VPS_IP}:22\r\n\r\n".encode())
resp = sock.recv(4096)                     # 必须读到 200，否则说明代理连不上
t = paramiko.Transport(sock)
t.connect(username="root", password="<PASSWORD>")
```

优点：你能自己执行、自己验证。

### 分支 B — 没有任何代理，只能商家 VNC

**不要尝试 SSH。** 改为生成一份**一次性脚本**让用户整段粘贴进 VNC：

- 幂等（重复执行不报错）
- 自带回滚（先 `cp -a <配置目录> <配置目录>.bak-$(date +%F)`）
- 末尾自带验证并打印结果
- 生成后告诉用户：粘贴 → 回车 → 把输出整段贴回来

---

## 6. 实施步骤

> 每步后面都跟 **verify**。verify 不过就停下，不要往下走。

### STEP 1 — Cloudflare DNS 记录（先只建记录，不急着改 SSL）

```bash
curl -s -X POST -H "Authorization: Bearer $CF_TOKEN" -H "Content-Type: application/json" \
  "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/dns_records" \
  -d '{"type":"A","name":"<RELAY_HOST>","content":"<VPS_IP>","proxied":true,"ttl":1}' | jq '.success'
```

**必须 `proxied: true`（橙色云）。** 灰云 = 直接暴露 IP = 一样被墙。

**verify**：`dig +short <RELAY_HOST>` 返回 Cloudflare 的段（`104.x` / `172.67.x` 等），**不是** `<VPS_IP>`。

### STEP 2 — Origin Certificate（在改 SSL 模式之前）

API 方式（Token 需带 Origin CA 权限）：

```bash
curl -s -X POST -H "Authorization: Bearer $CF_TOKEN" -H "Content-Type: application/json" \
  "https://api.cloudflare.com/client/v4/certificates" \
  -d '{"hostnames":["<RELAY_HOST>"],"requested_validity":5475,"request_type":"origin-rsa","csr":""}'
```

没有 Origin CA 权限时，让用户点面板：**SSL/TLS → Origin Server → Create Certificate**，主机名填 `<RELAY_HOST>`，有效期 15 年。

落到服务器：

按服务端栈选目录（**xray 和 sing-box 的证书目录必须和 STEP 4 里 `certificate_path` 指向的一致，否则服务起不来**）：

```bash
# sing-box
CERT_DIR=/etc/sing-box/cf
# xray（改用这一行，注释掉上面那行）
# CERT_DIR=/usr/local/etc/xray/cf

install -d -m 700 "$CERT_DIR"
install -m 600 origin.crt origin.key "$CERT_DIR"/
# 服务若以专用用户运行（如 xray:xray），必须改属主，否则读不到证书
chown -R root:root "$CERT_DIR"
echo "$CERT_DIR"    # 记下它，STEP 4 要填进去
```

**verify**：`openssl x509 -in /etc/sing-box/cf/origin.crt -noout -subject -dates` 能看到主机名和 15 年有效期。

### STEP 3 — SSL 模式设 Full (strict)

**必须在 STEP 2 之后做**，否则回源握手失败，站点会 526。

⚠️ **这是 zone 级设置，会影响同一根域名下所有橙色云主机名。** 先按 §4-P6 确认，再决定用 `strict` 还是 `full`。

```bash
curl -s -X PATCH -H "Authorization: Bearer $CF_TOKEN" -H "Content-Type: application/json" \
  "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/settings/ssl" \
  -d '{"value":"strict"}' | jq '.result.value'
```

### STEP 4 — 服务端新增入站

先备份，再新增。**端口按 §4-P5 的决策选**（下文以 `<PORT>` 表示）。

**sing-box（模块化，新增一个文件）** — `/etc/sing-box/conf/CF-WS.json`：

```json
{
  "inbounds": [{
    "tag": "CF-WS", "type": "vless", "listen": "::", "listen_port": <PORT>,
    "users": [{ "uuid": "<UUID>" }],
    "tls": { "enabled": true, "server_name": "<RELAY_HOST>",
             "certificate_path": "/etc/sing-box/cf/origin.crt",
             "key_path": "/etc/sing-box/cf/origin.key" },
    "transport": { "type": "ws", "path": "<WS_PATH>" },
    "sniffing": { "enabled": true, "destOverride": ["http", "tls", "quic"] }
  }]
}
```

```bash
cp -a /etc/sing-box/conf /etc/sing-box/conf.bak-$(date +%F)
sing-box check -c /etc/sing-box/config.json -C /etc/sing-box/conf   # 注意 -C，不带它不会校验 conf 目录
systemctl restart sing-box                                          # 多数单元没有 ExecReload，用 restart
```

**xray（单文件，插入到 `inbounds` 数组）** — `/usr/local/etc/xray/config.json`：

```json
{
  "port": <PORT>, "protocol": "vless",
  "settings": { "clients": [{ "id": "<UUID>" }], "decryption": "none" },
  "streamSettings": {
    "network": "ws", "security": "tls",
    "tlsSettings": { "certificates": [{
      "certificateFile": "/usr/local/etc/xray/cf/origin.crt",
      "keyFile": "/usr/local/etc/xray/cf/origin.key" }] },
    "wsSettings": { "path": "<WS_PATH>" }
  },
  "sniffing": { "enabled": true, "destOverride": ["http", "tls", "quic"] }
}
```

```bash
cp -a /usr/local/etc/xray/config.json /usr/local/etc/xray/config.json.bak-$(date +%F)

# 1) 先把新入站写成单元素数组文件（下面这段照抄，按需替换占位符）
cat > /tmp/new-inbound.json <<'JSON'
[{
  "port": <PORT>, "protocol": "vless",
  "settings": { "clients": [{ "id": "<UUID>" }], "decryption": "none" },
  "streamSettings": {
    "network": "ws", "security": "tls",
    "tlsSettings": { "certificates": [{
      "certificateFile": "<CERT_DIR>/origin.crt",
      "keyFile": "<CERT_DIR>/origin.key" }] },
    "wsSettings": { "path": "<WS_PATH>" }
  },
  "sniffing": { "enabled": true, "destOverride": ["http", "tls", "quic"] }
}]
JSON

# 2) 校验文件确实是合法 JSON 数组，再增量并入
jq -e 'type == "array"' /tmp/new-inbound.json >/dev/null || { echo "新入站 JSON 不合法，停"; exit 1; }
jq --slurpfile nb /tmp/new-inbound.json '.inbounds += $nb'    /usr/local/etc/xray/config.json > /tmp/xray-new.json    && mv /tmp/xray-new.json /usr/local/etc/xray/config.json

xray run -test -c /usr/local/etc/xray/config.json
systemctl restart xray
```

> `--slurpfile` 读进来就是数组，所以新入站文件本身要写成 `[...]`，否则会变成嵌套数组、xray 启动报错。

> xray 与 sing-box 的差别：sing-box 模块化可以只加文件；xray 必须改同一个 `config.json`，**务必先备份、再用 `jq` 增量改**。

**verify**：`ss -lntp | grep ':<PORT> '` 有监听；`systemctl is-active <服务名>` = active；原入站端口仍在监听。

### STEP 5 — 客户端加节点

写进**当前生效的那个 profile**（先按 §4-P4 定位）。

**mihomo / Clash 系**：

```yaml
proxies:
  - name: cf-relay
    type: vless
    server: <RELAY_HOST>        # 填域名，别填优选 IP（README §8.1 有实测）
    port: <PORT>
    uuid: <UUID>
    udp: false                  # CF 免费版不承载 UDP
    tls: true
    servername: <RELAY_HOST>
    network: ws
    ws-opts:
      path: <WS_PATH>
      headers: { Host: <RELAY_HOST> }

proxy-groups:                   # 缺这一段，规则里的 Proxy 会指向不存在的组
  - name: Proxy
    type: select
    proxies: [cf-relay, DIRECT]

rules:
  - DOMAIN,<RELAY_HOST>,DIRECT  # 必须：否则中转域名自己被代理，死循环
  # ... 其余规则见 README §5
```

**sing-box 客户端**：

```json
{
  "type": "vless", "tag": "cf-relay",
  "server": "<RELAY_HOST>", "server_port": <PORT>, "uuid": "<UUID>",
  "tls": { "enabled": true, "server_name": "<RELAY_HOST>" },
  "transport": { "type": "ws", "path": "<WS_PATH>", "headers": { "Host": "<RELAY_HOST>" } }
}
```

**profile 会不会被覆盖？** 看 profile 条目里 `type:` —— `remote` 的会被订阅刷新覆盖，此时要改**扩展脚本 / override**，不要直接改 profile。`local` 的可以放心手改。

**verify**：客户端对该节点做延迟测试，有数值（不是超时）。

### STEP 6 — 关掉 UDP 泄露面（仅 Windows）

原因见 README §8.2：部分内核版本上，虚拟网卡进来的 UDP **不参与规则匹配**，会从真实网卡直接出去，
WebRTC / QUIC 会把真实宽带 IP 暴露给任何网页。

需要管理员权限（会弹 UAC，让用户点"是"）：

```powershell
$ports = @('443','3478','5349','19302-19312')
$exes = @("$env:ProgramFiles\Google\Chrome\Application\chrome.exe",
          "${env:ProgramFiles(x86)}\Microsoft\Edge\Application\msedge.exe")
foreach ($exe in $exes) {
  if (-not (Test-Path $exe)) { continue }
  $tag = [IO.Path]::GetFileNameWithoutExtension($exe)
  foreach ($p in $ports) {
    New-NetFirewallRule -DisplayName "BlockWebRtcUdp-$tag-$p" -Direction Outbound `
      -Program $exe -Protocol UDP -RemotePort $p -Action Block -Profile Any
  }
}
Get-NetFirewallRule -DisplayName 'BlockWebRtcUdp-*' | Select DisplayName,Enabled,Action
```

**只堵这几个端口**，不要堵 53（DNS）和 123（时间同步）。

**verify**（期望：无响应）：

```python
import socket
s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM); s.settimeout(3)
s.sendto(b"\x00\x01\x00\x00" + b"\x00"*16, ("162.159.207.0", 3478))
try:
    print("收到响应 → 还在漏"); print(s.recvfrom(64))
except TimeoutError:
    print("无响应 → 已阻断")
```

### STEP 7 — 重启后能一键恢复

虚拟网卡（TUN）开关**不会被客户端记住**；节点选择一般会记住（存在内核的 `cache.db`）。

**mihomo 系**：先确认 `external-controller` 已开启（在客户端设置里，常见 `127.0.0.1:9090`），然后：

```python
import json, urllib.request
API = "http://127.0.0.1:9090"          # 有 secret 就加 Authorization 头
def call(method, path, body=None):
    req = urllib.request.Request(API + path, method=method,
        data=json.dumps(body).encode() if body else None,
        headers={"Content-Type": "application/json"})
    urllib.request.urlopen(req, timeout=10).read()
call("PATCH", "/configs", {"mode": "rule"})
call("PATCH", "/configs", {"tun": {"enable": True}})
call("PUT",   "/proxies/Proxy", {"name": "cf-relay"})
print(urllib.request.urlopen(API + "/configs", timeout=10).read().decode()[:200])
```

Windows 便携版若没有外部控制器，可通过命名管道 `\\.\pipe\MihomoParty\mihomo-admin-<PID>` 发同样的 HTTP 请求（管道名带内核 PID，**每次重启都会变**，要重新枚举）。

**Clash Verge**：没有稳定可用的命令行，向用户给出手动 3 步（打开 TUN、切规则模式、选中节点），或让他把这三项写进启动脚本。**不要假装能自动做。**

**verify**：手动关掉 TUN，运行脚本，TUN 应恢复为 true 且节点延迟有数值。

---

## 7. 失败分支

| 症状 | 先查这个 |
|---|---|
| 节点延迟超时 | `<PORT>` 是否在监听；DNS 记录是否橙色云；WS 路径两边是否一致 |
| 站点返回 526 / 525 | SSL 模式设太早或证书没装好 → 回 STEP 2→3 顺序 |
| 能连但没速度 | `udp: false` 是否写对；是否误开了全局模式 |
| 所有国内站都走代理 | geo 数据（`geosite.dat`/`geoip.dat`/`Country.mmdb`）是否缺失 |
| 加完 CF 记录不生效 | 域名是否真在当前账号（§4-P2） |
| 检测站显示真实 IP | 虚拟网卡下 UDP 泄露 → STEP 6 |
| 检测站"DNS 泄露" | 明文 DNS 被路由器 53 端口劫持 → 换 DoH（README §8.4） |
| 速度比预期慢 | 先排除"刚部署完的瞬时状态"，别急着做优选 IP（README §8.1） |

---

## 8. 验收门（全过才算完成，缺一条都不算）

逐条跑，**把实测输出贴出来**，不要只说"已完成"。

| # | 门 | 命令 | 期望 |
|---|---|---|---|
| 1 | CF 在代理这个域名 | `dig +short <RELAY_HOST>` | 返回 `104.x`/`172.67.x` 等 CF 段，不是 `<VPS_IP>` |
| 2 | 回源链路通 | `curl -s https://<RELAY_HOST>/cdn-cgi/trace \| head -3` | 有 `h=<RELAY_HOST>` 字样（CF 在边缘应答，说明橙色云生效） |
| 3 | 出口是你自己的机器 | `curl -x http://127.0.0.1:<PROXY_PORT> -s https://ipinfo.io/json`（`<PROXY_PORT>` 用 §1-Q2 问到的那个） | `"ip": "<VPS_IP>"` |
| 4 | 走虚拟网卡也是同一个出口 | `curl -s https://ipinfo.io/json` | `"ip": "<VPS_IP>"` |
| 5 | 国内是直连 | `curl -o /dev/null -s -w '%{time_total}\n' https://www.baidu.com` | < 0.3 秒 |
| 6 | WebRTC 不泄露 | 用户在 `https://ip.net.coffee/webrtc` 看，或 STEP 6 的 STUN 自测 | 显示"已禁用或无泄露" / STUN 无响应 |
| 7 | 重启后能恢复 | mihomo：关 TUN → 跑 STEP 7 脚本；<br>Clash Verge：关 TUN → 按 UI 手动 3 步 | TUN 回到 true，节点延迟有值 |
| 8 | 原入站还在 | `ss -lntp \| grep '<ORIGINAL_PORT>'` | 仍在监听 |

**没有 shell 的情况**（例如你无法操作用户机器）：门 1/2/3/4/5 由你跑；门 6/7/8 让用户截图或粘贴输出，
并在报告里标明"此项由用户提供"。

`claude.ai` 这类站点对纯脚本请求返回 403 是风控盾，**不代表不通**，必须用浏览器验证。

---

## 9. 红线

1. **不要**硬写优选 IP —— README §8.1 有 6 个 IP × 5 轮的实测数据证明无效，且会随 CF 换段失效。
2. **不要**给这个节点设 `udp: true` —— CF 免费版不转发 UDP，设了只会让 QUIC 静默失败。
3. **不要**动用户域名下的其他记录、其他站点、其他服务。
4. **不要**默认开全局模式 —— 国内站会不可用（README §6 有实测），除非用户明确要求。
5. **不要**把密码、Token、UUID、订阅链接、真实 IP 写进任何会公开的文件。
6. **不要**为了"让检测页面变绿"而伪造路径（例如只把某个查 IP 的域名塞进代理）—— 那是骗测试，不是修复，README §8.3 说明了为什么那张卡改不动。
7. **不要**把做不到的事说成做完了 —— 见 §3 的降级表。

---

## 10. 回滚

每一步都是增量，回滚只需：

1. 删掉 STEP 4 新增的入站（sing-box 删文件；xray 用 `jq 'del(.inbounds[-1])'`），重启服务
2. 删掉 STEP 1 的 DNS 记录
3. 客户端删掉节点、`proxy-groups` 与新增规则
4. 删掉 `BlockWebRtcUdp-*` 防火墙规则：`Get-NetFirewallRule -DisplayName 'BlockWebRtcUdp-*' | Remove-NetFirewallRule`
5. 原有服务全程未改，不需要回滚；备份目录 `config.json.bak-<日期>` / `conf.bak-<日期>` 可留作对照
