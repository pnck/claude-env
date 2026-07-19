# claude-env

在 Docker 容器中运行 Claude Code，复用宿主机上现成的 SOCKS5 代理做透明代理，自动接管容器内全部出站流量。

## 架构

```
nftables (fwmark 1 / SO_MARK 0x438)
  → hev-socks5-tproxy :1088   （透明代理，接管 TCP+UDP）
  → 宿主机 SOCKS5              （默认 = 与容器同链路的宿主机 IP:1080）

dnsproxy :53 → DoH (8.8.8.8 / 1.1.1.1)   ← 经上述隧道转发，防 DNS 泄漏

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

## 技术栈

| 组件 | 版本 |
|------|------|
| [hev-socks5-tproxy](https://github.com/heiher/hev-socks5-tproxy) | 2.11.0 |
| [dnsproxy](https://github.com/AdguardTeam/dnsproxy) | 0.81.4 |
| node | 24-slim |

---

## 工作原理

**透明代理与防回环**
nftables 在 `output` 链给所有非本地、非 DNS 的 TCP/UDP 流量打 `fwmark 1`，经 `ip rule → table 100 → local default dev lo` 重新注入到 `prerouting`，由 `hev-socks5-tproxy` 用 TPROXY 接管并转发给上游 SOCKS5。

hev 自己到上游 SOCKS5 的连接带 `SO_MARK 0x438`，在 nft 里用 `meta mark 0x438 return` 放行——无论上游是私有 IP 还是公网 IP 都不会被重新隧道化，从而避免回环。这套按 mark 放行的机制也意味着容器里不区分任何 UID，`claude`（node, UID 1000）的全部出站都会被正确隧道化。

**DNS**
容器内所有 `:53` 查询被 nft（nat 链）重定向到本地 `dnsproxy`，由它通过 DoH（HTTPS:443）向 8.8.8.8 / 1.1.1.1 发起解析，而这条 HTTPS 又会被透明代理走隧道——既防 DNS 泄漏，也不依赖上游 SOCKS5 是否支持 UDP。`dnsproxy` 同时启用了内置的遥测域名 hosts 拦截（见上文「隐私加固」）。

**自动探测宿主机**
`SOCKS5_SERVER` 留空时，entrypoint 用 `ip route show default` 取默认网关。在 Docker 网桥模式下，默认网关就是宿主机在该网桥上的地址，因此「与容器同链路的宿主机 IP」无需手填。

**关于 UDP**
`SOCKS5_UDP=udp` 走标准 UDP ASSOCIATE；若上游不支持 UDP，可改成 `tcp`（UDP-over-TCP）。Claude Code 全程 TCP/HTTPS，DNS 也走 DoH（TCP），即便 UDP 不通也不影响主流程。
