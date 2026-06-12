# 更新日志 (Changelog)

本文件记录 **AimiliVPN 多出口增强版（二开）** 相对上游原项目 [baoweise-bot/aimili-vpngate](https://github.com/baoweise-bot/aimili-vpngate) 的改动。
格式参考 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/)。

---

## [Fork 1.0.0] - 2026-06-12

首个二次开发版本，在保留上游全部能力的基础上新增多出口能力并做了性能/可靠性优化。

### 新增 (Added)
- **多出口住宅 IP（Multi-Exit）**：单机同时维持 N 条相互隔离的隧道，每条 = 独立 `tunN` + 独立策略路由表（`200+i`）+ 独立本地代理端口（`17928+i`），连接不同住宅节点。
  - 后台「管理员 → 多出口住宅 IP」面板：实时设置出口数量、国家过滤（如 `JP,KR`）、仅住宅 IP 开关，并展示每个槽位的状态 / IP / 国家 / 端口。
  - **per-slot 自动漂移**：某槽位节点掉线时自动从健康住宅节点池补齐。
  - **一键导出 3x-ui / Xray 出站配置**（`outbounds` + `routing.rules` 模板）。
  - 新增 API：`GET /api/exit_slots`、`POST /api/update_exit_slots`、`GET /api/exit_slots/3xui`。
  - 槽位状态持久化到 `slots.json`。
- 新增可调环境变量：`MAX_EXIT_SLOTS`、`MULTI_EXIT_SLOTS`、`SLOT_PORT_BASE`、`SLOT_DEV_BASE`、`SLOT_TABLE_BASE`、`EXIT_SLOTS_CHECK_INTERVAL`、`OPENVPN_TEST_CONCURRENCY`、`TCP_PRESCREEN_CONCURRENCY`、`OPENVPN_TUN_DNS`、`LOCAL_PROXY_DNS_TTL`、`LOCAL_PROXY_RELAY_TIMEOUT` 等。
- `.gitattributes`：统一 LF 行尾，避免 Windows 编辑给 bash/python 脚本引入 CRLF。

### 优化 (Changed)
- **分层并发测速**：`test_multiple_nodes` 先用高并发 TCP 连通性粗筛淘汰明确不可达的 TCP 协议节点，再对存活节点做完整 OpenVPN 精验；UDP/未知节点不参与粗筛以避免误杀。OpenVPN 精验并发数由写死的 5 改为可配置（默认 8）。
- **隧道内 DNS**：`resolve_dns_over_tun0` 加入 TTL 缓存与多上游 DNS 竞速（默认 `8.8.8.8,1.1.1.1`），消除每连接重复解析与单一 DNS 不可达时的干等超时。
- **本地代理转发**：`relay` 重写为非阻塞双向泵（带背压与半关闭传播），修复原「只 select 可读 + 阻塞 sendall」可能导致的半双工背压卡死，空闲超时可配置。
- `proxy_server` 出站设备参数化（`tun0` → `device`），代理网关支持 `stop_event` 优雅停止；策略路由 `setup/cleanup_policy_routing` 参数化（接口 + 路由表号）。以上改动默认值保持与原行为一致，**向后兼容**。

### 修复 (Fixed)
- 多出口供给器并发重入：`supervise_exit_slots_once` 加非阻塞互斥锁，避免周期线程与 API 触发线程同时对同一槽位重复拨号、累加 `ip rule`。
- 进程重启遗留孤儿隧道：启动时 `kill_slot_openvpn_processes` 回收带 `AIMILI_SLOT` 标记的旧槽位 OpenVPN 进程并清理残留路由表，避免新供给器无法在 `tunN` 上重新拨号。
- 主连接清理不再误杀多出口隧道：`kill_existing_openvpn_processes` 跳过带 `AIMILI_SLOT` 标记的进程；槽位拨号 `report_status=False`，不污染主连接的 UI 状态显示。

### 文档 (Docs)
- README 重写为本二开项目的独立文档（中英双语），突出多出口与 3x-ui 集成，含核心特性表、环境变量表、架构示意图与 3x-ui 配置示例。
- 安装脚本 `install.sh` 默认仓库地址切换为本二开仓库 `Guli-Joy/aimili-vpngate`（保留命令行参数覆盖能力）。
