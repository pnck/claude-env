# claude-env

在 Docker 容器中运行 Claude Code，复用宿主机上现成的 SOCKS5 代理做透明代理，自动接管容器内全部出站流量。

## 架构

```
nftables (fwmark 1 / SO_MARK 0x438)
  → hev-socks5-tproxy :1088   （透明代理，接管 TCP+UDP）
  → 宿主机 SOCKS5              （默认 = 与容器同链路的宿主机 IP:1080）

dnsmasq :53 （前端）
  → addn-hosts（遥测屏蔽）
  → 普通域名：dnsproxy 127.0.0.1:5353 → DoH (8.8.8.8/1.1.1.1)  ← 经隧道，防 DNS 泄漏
  → NO_PROXY_DOMAINS：dnsproxy 127.0.0.1:5354（dnsdir UID）→ 明文 DNS 8.8.8.8/1.1.1.1  ← 直连
      + nftset 把 A 记录写入 noproxy_ips

nftables output:
  meta skuid 1001 return         （dnsdir 进程出站直连，NO_PROXY_DOMAINS 查询从真实出口发出）
  ip daddr @noproxy_ips return   （域名/IP 命中直连集合：不打标、不进隧道）

claude 容器  network_mode: service:proxy   （共享 proxy 网络栈，无需应用侧配置）
```

容器内不运行任何隧道客户端，compose file 中只需填入 SOCKS5 地址/端口。

## 使用

**1. 触发 Action**：Actions → Build Images & Generate Compose → Run workflow

Action 会构建 `claude-proxy` 和 `claude-code` 多架构镜像（amd64 + arm64）推送到 GHCR，并生成 `docker-compose.yml` 作为 Artifact / Release 资产供下载。

**2. 准备宿主机 SOCKS5**

确保宿主机上有一个 SOCKS5 代理在运行，且**监听在容器可达的地址**——不能只听 `127.0.0.1`。常见做法：

- 把监听地址设为 `0.0.0.0`，或宿主机 LAN IP；
- 客户端里开启 **Allow LAN / 允许局域网连接**。

> 注：proxy 容器在自己的 network namespace 里，宿主机的 `127.0.0.1` 对它不可达。容器通过宿主机在 Docker 网桥/局域网上的地址去连这个 SOCKS5。

**3. 填入配置并启动**

下载 `docker-compose.yml`，按需编辑 `proxy` 服务的 `environment`：

```yaml
environment:
  # 留空 = 自动探测“与本容器同链路的宿主机 IP”(默认网关)，最省事
  - SOCKS5_SERVER=
  - SOCKS5_PORT=1080
  # 可选认证（无认证留空即可）
  - SOCKS5_USERNAME=
  - SOCKS5_PASSWORD=
  # UDP 转发模式 udp|tcp；若上游支持 UDP 则填 `udp`（不影响 Claude 的连接，它仅使用纯 TCP/HTTPS）
  - SOCKS5_UDP=tcp
```

`SOCKS5_SERVER` 三种填法：

| 填法 | 适用场景 |
|------|----------|
| 留空（推荐） | 自动取默认网关，即与容器同链路的宿主机地址 |
| `192.168.x.x` / 网关 IP | 想显式指定宿主机地址 |
| `host.docker.internal` | 显式指向宿主机（compose 已内置 `extra_hosts: host-gateway`） |

需要暴露开发服务器端口时，在 `proxy` 服务的 `ports` 块中取消注释对应行（端口映射须在 `proxy` 服务下才生效，`claude` 容器共享其网络栈）。

然后启动：

```bash
docker compose up -d
docker attach claude-code   # 直连 PTY
# 退出但保持运行：Ctrl+P Ctrl+Q
# 彻底停止：docker compose down
```

启动后可 `docker logs claude-proxy` 检查，看到 `upstream reachable OK` 即代表已连上宿主机 SOCKS5。

## 隐私加固：默认禁用遥测

`claude` 容器默认设置：

```yaml
environment:
  - DISABLE_TELEMETRY=1
  - CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1
```

后者是官方"一键"开关，除遥测外一并关闭错误上报、反馈调查、自动更新检查——容器本就基于固定镜像版本每次重建，不依赖运行期自动更新。

`proxy` 容器额外在 DNS 层拦截已知的遥测/统计域名（如 `statsig.anthropic.com`，解析到 `0.0.0.0` 使连接失败），即便应用侧开关被绕过或后续版本新增遥测调用也能兜底。如需拦截更多域名，在 `proxy` 服务的 `EXTRA_BLOCKED_HOSTS` 环境变量中追加（逗号分隔）；如需临时放开遥测，清空对应环境变量即可。

如需单独放开内置屏蔽中的某一类域名（例如需要联调遥测，或自行接入 Sentry 上报），在 `proxy` 服务的 `environment` 中设置：

```yaml
environment:
  # 放开 Anthropic/Statsig 遥测域名屏蔽（statsig.anthropic.com 等），默认 0=仍屏蔽
  - ALLOW_ANTHROPIC_TELEMETRY=0
  # 放开 Sentry 域名屏蔽（sentry.io / o1.ingest.sentry.io），默认 0=仍屏蔽
  - ALLOW_SENTRY=0
```

设为 `1` 即放开对应一类域名的屏蔽；两者相互独立，可分别开启。

## 域名/IP 级直连绕过

当 `ANTHROPIC_BASE_URL` 指向走 CDN 的自定义 LLM Endpoint 时，后端 IP 常常频繁变化，静态 CIDR 白名单难以维护。`proxy` 服务提供两种直连绕过方式：

- `NO_PROXY_DOMAINS`：逗号分隔域名列表。dnsmasq 把匹配域名的查询转发给**直连解析器**（`127.0.0.1:5354`，以专用 `dnsdir` UID 运行，nftables 对该 UID 出站直接放行），从容器真实出口发出 DNS 请求；同时通过 `nftset=` 把解析得到的 A 记录原子写入 `noproxy_ips` 集合，使后续 TCP/UDP 连接也走直连。这样 **DNS 查询本身** 和 **后续连接** 都从真实网络位置发出，GeoDNS/CDN 能感知到容器实际出口 IP，返回最近的边缘节点。域名天然匹配全部子域名，无需通配符。允许最多一个前导点/尾随点（如 `.example.com`、`example.com.`），会自动归一化去掉。
- `NO_PROXY_IPS`：逗号分隔静态 IPv4/CIDR 条目。容器启动时通过 `nft add element` 直接写入 `noproxy_ips`，不依赖 DNS。

> 注意：本容器不支持 IPv6（内核层面已禁用 `net.ipv6.conf.all.disable_ipv6=1` 等 sysctl），`NO_PROXY_IPS` 中的 IPv6 条目会被检测并忽略（stderr 警告）。

示例：

```yaml
services:
  proxy:
    environment:
      # CDN 场景：按域名动态直连（自动覆盖子域名）
      - NO_PROXY_DOMAINS=api.anthropic.example.com,cdn.example.net.
      # 静态直连 IPv4/CIDR
      - NO_PROXY_IPS=104.18.0.0/16,203.0.113.10
```

## 技术栈

| 组件 | 版本 |
|------|------|
| [hev-socks5-tproxy](https://github.com/heiher/hev-socks5-tproxy) | 2.11.0 |
| [dnsproxy](https://github.com/AdguardTeam/dnsproxy) | 0.81.4 |
| dnsmasq | 随 Debian trixie 源 |
| node | 24-slim |

---

## 工作原理

**透明代理与防回环**
nftables 在 `output` 链给所有非本地、非 DNS 的 TCP/UDP 流量打 `fwmark 1`，经 `ip rule → table 100 → local default dev lo` 重新注入到 `prerouting`，由 `hev-socks5-tproxy` 用 TPROXY 接管并转发给上游 SOCKS5。

hev 自己到上游 SOCKS5 的连接带 `SO_MARK 0x438`，在 nft 里用 `meta mark 0x438 return` 放行——无论上游是私有 IP 还是公网 IP 都不会被重新隧道化，从而避免回环。这套按 mark 放行的机制也意味着容器里不区分任何 UID，`claude`（node, UID 1000）的全部出站都会被正确隧道化。

**DNS**
容器内所有 `:53` 查询被 nft（nat 链）重定向到本地 `dnsmasq`。`dnsmasq` 作为前端负责两件事：一是通过 `addn-hosts=` 加载遥测屏蔽 hosts 文件；二是把其余查询转发到 `127.0.0.1:5353` 的 `dnsproxy`，再由后者走 DoH（HTTPS:443）访问 8.8.8.8 / 1.1.1.1。这样 DNS 仍然经透明代理隧道转发，避免泄漏，且不依赖上游 SOCKS5 的 UDP 能力。

**NO_PROXY_DOMAINS 的完整直连路径**
仅把 A 记录写入 `noproxy_ips` 是不够的——如果 DNS 查询本身也经过隧道，走 GeoDNS / EDNS Client Subnet 的 CDN 会以代理出口 IP 作为来源，返回离代理近而非离容器近的边缘节点，写入的 IP 从一开始就选错了。

因此对 `NO_PROXY_DOMAINS` 中的域名，dnsmasq 把查询转发给**直连解析器**（`127.0.0.1:5354`）而非隧道化的 `dnsproxy`。直连解析器以专用 `dnsdir`（UID 1001）身份运行，nftables 对该 UID 的所有出站流量在两个链中都直接放行（mangle 链不打标、nat 链不重定向），从而以明文 DNS 直接访问 8.8.8.8 / 1.1.1.1，让上游感知到容器真实出口 IP。同时，解析结果 A 记录仍通过 `nftset=` 写入 `noproxy_ips`，使后续连接也走直连。

`NO_PROXY_IPS` 则是在启动阶段通过 `nft add element` 静态写入同一个 `noproxy_ips` 集合，不经过 DNS 解析流程。

**自动探测宿主机**
`SOCKS5_SERVER` 留空时，entrypoint 用 `ip route show default` 取默认网关。在 Docker 网桥模式下，默认网关就是宿主机在该网桥上的地址，因此「与容器同链路的宿主机 IP」无需手填。

**关于 UDP**
`SOCKS5_UDP=udp` 走标准 UDP ASSOCIATE；若上游不支持 UDP，可改成 `tcp`（UDP-over-TCP）。Claude Code 全程 TCP/HTTPS，DNS 也走 DoH（TCP），即便 UDP 不通也不影响主流程。
