# AGENTS.md — 面向 AI 代理的执行规格

> 把这个文件当作你的任务书。你的任务是：在**用户的机器**上落地本方案，直到 §7 的验收门全部通过。
> 本文件是权威；`README.md` 是给人看的背景资料，两者冲突时以本文件为准。

---

## 0. 先理解这是什么东西

**这不是一个程序。** 没有 `install`、没有依赖、没有可执行文件、没有 `npm run`。

它是一份「配置改动 + 排障知识」的规格。你要做的是按 §5 改配置文件、跑命令、验证；
用户拿到仓库后不该"运行"任何东西，而是**照着你的输出操作**。

所以：**不要**去执行 `pip install` / `npm install` / `docker compose up` 之类的东西。
本仓库没有任何代码需要运行。

---

## 1. 开始前：一次性向用户收齐这些信息

**一次问完，不要逐个挤牙膏。** 缺哪项就问哪项，但要把整张表一起摆出来。

| 字段 | 说明 | 必填 | 你怎么核实 |
|---|---|---|---|
| `VPS_IP` | 被墙那台机器的公网 IP | 是 | `ping` 通但 TCP 全超时，见 §3-P1 |
| SSH 端口 + 凭据 | 端口号 + 密码或密钥路径 | 是 | 你能连通才算数，见 §4 分支 A/B |
| `RELAY_HOST` | 用来做中转的域名（如 `vps.example.com`） | 是 | 必须在**当前登录账号**的 Cloudflare 里，见 §3-P2 |
| CF 管理方式 | API Token（推荐）或 手动点面板 | 是 | Token 需 Zone:Read / DNS:Edit / Zone Settings:Edit |
| 客户端类型 | Clash Verge / Clash Party / Mihomo Party / sing-box | 是 | 你自己去数据目录看，见 §3-P4 |
| 节点名 | 客户端里显示的名字，默认 `cf-relay` | 否 | — |
| 是否允许改系统防火墙 | 需要管理员权限/UAC | 否 | 见 §5-STEP6，仅 Windows |
| 是否保留原入站 | **强烈建议保留** | 否 | 默认保留 |

**绝对不要**在聊天记录、提交信息、仓库文件里写入密码、UUID 私钥、订阅链接。需要落地到磁盘时，
写进用户本机的临时目录，并告知路径。

---

## 2. 术语与边界

- **中转**：域名 → Cloudflare（橙色云代理）→ VPS 的 443 端口（VLESS + WS + TLS）。
- **出口 IP 仍是用户的 VPS**，不是 Cloudflare 的 IP。这是选本方案而不是 Cloudflare Worker 的**唯一**理由；如果用户不在乎出口 IP 归属，改用 Worker 方案更省事。
- **Cloudflare 免费版只转发 TCP**。UDP 走不了，所以节点配置必须 `udp: false`。
- 本方案**只新增入口，不改原有服务**。原有入口（如 Reality）必须原样保留，解封后可一键切回。

---

## 3. 预检（动手之前做完，任何一项不过就停下来报告，不要硬上）

### P1 — 确认真的被墙，而不是服务挂了

| 现象 | 结论 |
|---|---|
| ICMP 通，但 22/80/443 等**所有** TCP 端口超时 | IP 级阻断 —— 本方案适用 |
| ICMP 通，个别端口不通 | 防火墙/服务问题 —— **不要**上中转，先修服务 |
| ICMP 也不通 | 机器宕机或路由问题 —— 先找商家 |

用**境外节点**反查（例如在线 tcping 工具，注意有些站点的 `/ping/` 用的是 ICMP，会误判成绿；要用 TCP 检测）。
境外能连上 22 端口、境内连不上 = 确认被墙。

### P2 — 确认域名在你当前操作的 Cloudflare 账号里

这一步**必须**核实，否则记录加完不生效，白白排查几小时：

```bash
curl -s -H "Authorization: Bearer $CF_TOKEN" \
  "https://api.cloudflare.com/client/v4/zones?name=<YOUR_ZONE>" | jq '.result[] | {id,name,status}'
```

返回空数组 = 这个域名的 NS 虽然指向 Cloudflare，但**不在这个账号**。此时停下来问用户换域名或换账号。

### P3 — 识别服务端软件

```bash
ls /etc/sing-box/conf 2>/dev/null && echo "→ sing-box 模块化"
ls /usr/local/etc/xray 2>/dev/null && echo "→ xray"
systemctl status sing-box xray 2>/dev/null | head -20
```

不同软件配置格式完全不同。**先确认再动手**，不要假设。

### P4 — 找到客户端真实数据目录

便携版软件的配置目录常常不是 `%APPDATA%`，那里可能只是残留：

```bash
ls -la "<客户端安装目录>/data/"    # 便携版
ls -la "$APPDATA/<客户端名>/"      # 安装版
```

判断依据：看哪个目录里有 `profiles/`、`work/config.yaml`、`config.yaml` 且**最近有修改时间**。
改错目录会让你以为配置没生效。

### P5 — 端口占用

```bash
ss -lntp | grep ':443 '   # 443 是否已被占用
```

被占用就换端口，或让用户决定是否复用。

---

## 4. 服务器访问：两条路，先判定走哪条

IP 被墙时，用户本机通常**无法直接 SSH**。先决定访问路径：

### 分支 A — 用户手上还有别的可用代理（推荐）

境外出口能连上 VPS，就可以走 HTTP CONNECT 隧道 SSH：

```python
# paramiko 经本地代理连服务器
sock = socket.create_connection(("127.0.0.1", <PROXY_PORT>))
sock.sendall(b"CONNECT <VPS_IP>:22 HTTP/1.1\r\nHost: <VPS_IP>:22\r\n\r\n")
# 读掉 200 响应头，再把 sock 交给 paramiko.Transport
```

优点：你可以自己执行、自己验证。缺点：需要一个能用的代理端口。

### 分支 B — 没有任何代理，只能走商家面板 VNC

那就**不要**尝试 SSH。改为生成一份**一次性脚本**，让用户整段粘贴进 VNC 控制台：

- 脚本必须**幂等**（重复执行不报错、不重复写文件）
- 脚本必须**自带回滚**（先 `cp -a conf conf.bak-$(date +%F)`）
- 脚本必须**打印验证结果**（配置语法检查 + 服务状态 + 监听端口）
- 生成后告诉用户：粘贴 → 回车 → 把输出整段贴回来

---

## 5. 实施步骤

> 每一步后面都跟一条 **verify**。verify 不过就停下来，不要往下走。

### STEP 1 — Cloudflare DNS 记录

```bash
curl -s -X POST -H "Authorization: Bearer $CF_TOKEN" -H "Content-Type: application/json" \
  "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/dns_records" \
  -d '{"type":"A","name":"<RELAY_HOST>","content":"<VPS_IP>","proxied":true,"ttl":1}'
```

**必须 `proxied: true`（橙色云）。** 灰云 = 直接暴露 IP = 一样被墙。

**verify**：`dig +short <RELAY_HOST>` 返回的是 Cloudflare 的地址段（104.x / 172.67.x 等），**不是** `VPS_IP`。

### STEP 2 — SSL 模式设为 Full (strict)

```bash
curl -s -X PATCH -H "Authorization: Bearer $CF_TOKEN" -H "Content-Type: application/json" \
  "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/settings/ssl" \
  -d '{"value":"strict"}'
```

### STEP 3 — Origin Certificate

如果 Token 有 Origin CA 权限，用 API 生成；否则引导用户在面板操作：

> Cloudflare → SSL/TLS → Origin Server → Create Certificate
> 主机名填 `<RELAY_HOST>`，有效期 15 年，密钥格式 PEM

把 `origin.crt` / `origin.key` 放到 VPS 的 `/etc/sing-box/cf/`，权限 `600`。

**verify**：`openssl x509 -in origin.crt -noout -subject -dates` 能看到主机名和 15 年有效期。

### STEP 4 — 服务端新增入站（不动原有）

备份 → 新增一个 conf 文件 → 语法检查 → 重载。

```json
{
  "inbounds": [{
    "tag": "CF-WS-443", "type": "vless", "listen": "::", "listen_port": 443,
    "users": [{ "uuid": "<UUID>" }],
    "tls": { "enabled": true, "server_name": "<RELAY_HOST>",
             "certificate_path": "/etc/sing-box/cf/origin.crt",
             "key_path": "/etc/sing-box/cf/origin.key" },
    "transport": { "type": "ws", "path": "<WS_PATH>" },
    "sniffing": { "enabled": true, "destOverride": ["http", "tls", "quic"] }
  }]
}
```

- `<UUID>` 用 `uuidgen` 新生成，**不要复用原有节点的**
- `<WS_PATH>` 用随机路径，如 `/` + 8 位随机串
- 先 `sing-box check`，再 `systemctl reload sing-box`

**verify**：`ss -lntp | grep ':443 '` 能看到监听；`systemctl is-active sing-box` = active。

### STEP 5 — 客户端加节点

写进**用户当前生效的那个 profile 文件**（不是随便一个）：

```yaml
- name: cf-relay
  type: vless
  server: <RELAY_HOST>        # 填域名，不要填优选 IP，理由见 README §8.1
  port: 443
  uuid: <UUID>
  udp: false                  # CF 免费版不承载 UDP，必须是 false
  tls: true
  servername: <RELAY_HOST>
  network: ws
  ws-opts:
    path: <WS_PATH>
    headers: { Host: <RELAY_HOST> }
```

同时在规则最前面加一条，防止中转域名自己被代理成死循环：

```yaml
- DOMAIN,<RELAY_HOST>,DIRECT
```

DNS 必须是 fake-ip，且 `<YOUR_ZONE>` 要进 `fake-ip-filter`（详见 README §5）。

**注意**：有些客户端会把 profile 重新生成后覆盖你的手改。判断方法 —— 看 profile 条目里 `type: local` 还是 `remote`；
`remote` 的会被订阅刷新覆盖，此时应改**扩展脚本 / override**，而不是直接改 profile。

**verify**：客户端里对该节点做一次延迟测试，有数值（不是超时）。

### STEP 6 — 关闭 UDP 泄露面（Windows 客户端）

原因见 README §8.2：在部分内核版本上，虚拟网卡进来的 UDP **不参与规则匹配**，
会直接从真实网卡出去，导致 WebRTC / QUIC 把你的真实宽带 IP 暴露给任何网页。

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
```

需要管理员权限。**只堵这几个端口**，不要堵 53（DNS）和 123（时间同步）。

**verify**：对 `162.159.207.0:3478` 发一个 STUN 包应当**无响应**（有响应说明还在漏）。

### STEP 7 — 做成重启后能一键恢复

虚拟网卡（TUN）开关**不会**被客户端记住（节点选择一般会记住）。

生成一个恢复脚本，做四件事并打印结果：重载配置 → 切规则模式 → 开 TUN → 选中节点。

**verify**：手动关掉 TUN，运行脚本，TUN 应重新变 true，且节点延迟有数值。

---

## 6. 失败分支

| 症状 | 先查这个 |
|---|---|
| 节点延迟超时 | 服务器 443 是否在监听；CF 记录是否橙色云；WS 路径是否两边一致 |
| 能连但没网速 | `udp: false` 是否写对；是否误开了全局模式 |
| 所有国内站都走代理 | geo 数据文件（`geosite.dat`/`geoip.dat`/`Country.mmdb`）是否缺失 |
| 加完 CF 记录不生效 | 域名是否真的在当前账号（§3-P2） |
| 检测站显示真实 IP | 虚拟网卡下的 UDP 泄露，见 STEP 6 |
| 检测站显示"DNS 泄露" | 上游是明文 DNS 被路由器 53 端口劫持，换 DoH，见 README §8.4 |
| 速度比预期慢 | 先确认不是"刚部署完的瞬时状态"；不要急着做优选 IP（README §8.1 有实测数据） |

---

## 7. 验收门（全过才算完成，缺一条都不算）

1. `curl` 目标站点的 `/cdn-cgi/trace`，`ip=` 等于 `VPS_IP`
2. 不设代理（走虚拟网卡）时，出口 IP 也是 `VPS_IP`
3. 国内站点 200 ms 内返回（证明真的直连，没绕路）
4. WebRTC 检测结果为「已禁用或无泄露」
5. 目标服务在浏览器里能正常打开（`claude.ai` 这类站点对纯脚本请求返回 403 是风控盾，不代表不通，**必须用浏览器验证**）
6. 重启客户端后，恢复脚本能让一切复原
7. 原有入站仍在（解封后能切回）

做完把上面 7 条**逐条贴出实测数值**，不要只说"已完成"。

---

## 8. 红线

1. **不要**硬写优选 IP —— 本仓 README §8.1 有实测数据证明无效，且会随 CF 换段失效。
2. **不要**给这个节点设 `udp: true` —— Cloudflare 免费版不转发 UDP，设了只会让 QUIC 静默失败。
3. **不要**动用户域名下的其他记录、其他站点、其他服务。
4. **不要**默认开全局模式 —— 国内站会变得不可用（实测数据在 README §6），除非用户明确要求。
5. **不要**把密码、UUID、订阅链接、真实 IP 写进任何会公开的文件。
6. **不要**为了"让检测页面变绿"而伪造路径（例如只把某个查 IP 的域名塞进代理）—— 那只是骗测试，不是修复。README §8.3 说明了为什么那张卡改不动。

---

## 9. 回滚

每一步都是增量，回滚只需：

1. 删掉 STEP 4 新增的那个 conf 文件，`systemctl reload sing-box`
2. 删掉 STEP 1 的 DNS 记录
3. 客户端删掉节点与那几条规则
4. 删掉 `BlockWebRtcUdp-*` 防火墙规则
5. 原有服务从头到尾没被改过，不需要回滚
