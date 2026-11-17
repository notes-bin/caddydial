# caddydial 设计文档

> 日期：2026-11-17  
> 状态：待实现  
> 模块路径：`github.com/notes-bin/caddydial`  
> Go 版本：1.27+

## 1. 目标

提供 Caddy 反向代理条目的控制面：CLI + 可选内嵌 Web。  
**SQLite 为配置源，Caddy Admin API 为生效面。** 业务流量只经过 Caddy，本应用不转发用户流量。

## 2. 范围

### 包含

- 代理条目增删改查（以库为准，同步到 Caddy）
- `@id` 精确定位；ETag / `If-Match` 降低并发冲突
- 主动 + 被动健康检查配置写入 Caddy `reverse_proxy`
- 健康展示：`GET /reverse_proxy/upstreams` + Caddy `/metrics`
- 配置导出 CSV / XLSX；导入写入 SQLite 再同步
- `//go:embed` 内嵌 Web；`caddyctl web --listen ...`
- 双容器部署：Caddy 镜像 + 本应用镜像
- Web 安全基线 B：默认本机；对外监听强制管理员密码/Token + Cookie 会话
- 审计日志

### 不包含（首版）

- 多用户 / RBAC
- DuckDB
- 由本应用自行探活并改路由（健康由 Caddy 数据面处理）
- 公网裸奔 Admin API / Web

## 3. 架构

### 3.1 进程与分层

```text
cmd/caddyctl                 # 入口
internal/model               # 领域类型（含 route JSON 包装）
internal/store               # SQLite（落盘 + WAL）
internal/caddyapi            # Admin API 客户端（超时、ETag、@id、upstreams、metrics）
internal/syncer              # 库 → Caddy 原样提交 / 对账
internal/service             # 用例（CLI 与 Web 共用）
internal/cli                 # 子命令
internal/web                 # embed 静态资源 + HTTP
```

依赖注入：手工构造 + functional options，无 DI 框架。

### 3.2 数据流（写）

1. CLI 或 Web 提交变更  
2. 校验 `route_json` 为合法 JSON，且 `@id` 与行 `id` 一致  
3. 事务写入 `proxies` + `audit_logs`  
4. `syncer` 将 `route_json` **原样** `POST`/`PATCH` 到 Admin API（不做字段映射）  
5. 成功 → `sync_status=synced`；失败 → `error` + `sync_error`，可 `sync` 重试  

禁用（`enabled=0`）：从 Caddy `DELETE /id/{id}`，库中保留记录。

### 3.3 数据流（读 / 健康）

- 列表与详情：以 SQLite 为准  
- 节点健康：实时查 Caddy upstreams + metrics，**不落库**  
- 被动：用 `fails` 与 `route_json` 内 `max_fails` 等对比  
- 主动：优先 `/metrics` 中的 upstream 健康指标  

### 3.4 CLI 与 Web 并行

- 短命令：`caddyctl <subcommand>`  
- 长驻：`caddyctl web --listen 127.0.0.1:8080`  
- 可另开终端同时使用 CLI；两者独立进程，共用同一 store 与同一 Caddy Admin  

## 4. SQLite 结构（尽量简单）

库文件示例：`caddydial.store`（**不以 `.db` 结尾**），`PRAGMA journal_mode=WAL`。

### 4.1 `proxies`

| 列 | 类型 | 说明 |
| --- | --- | --- |
| `id` | TEXT PK | 与 `route_json.@id` 一致 |
| `route_json` | TEXT NOT NULL | Caddy Route 对象原文，可直接作为 Admin API body |
| `notes` | TEXT | 仅本系统备注 |
| `enabled` | INTEGER NOT NULL DEFAULT 1 | |
| `sync_status` | TEXT NOT NULL | `synced` / `pending` / `error` |
| `sync_error` | TEXT | |
| `created_at` | TEXT NOT NULL | RFC3339 |
| `updated_at` | TEXT NOT NULL | RFC3339 |

`route_json` 示例：

```json
{
  "@id": "app_example_com",
  "match": [{"host": ["app.example.com"]}],
  "handle": [{
    "handler": "reverse_proxy",
    "upstreams": [{"dial": "10.0.1.1:8080"}],
    "health_checks": {
      "active": {"path": "/health", "interval": "30s", "timeout": "5s"},
      "passive": {"fail_duration": "30s", "max_fails": 1}
    }
  }]
}
```

### 4.2 `audit_logs`

| 列 | 类型 | 说明 |
| --- | --- | --- |
| `id` | INTEGER PK AUTOINCREMENT | |
| `proxy_id` | TEXT | |
| `action` | TEXT NOT NULL | create/update/delete/sync/enable/disable/… |
| `source` | TEXT NOT NULL | cli / web |
| `detail` | TEXT | JSON 快照（请求体、前后 route、错误等） |
| `created_at` | TEXT NOT NULL | |

### 4.3 `schema_meta`

| 列 | 类型 |
| --- | --- |
| `key` | TEXT PK |
| `value` | TEXT NOT NULL |

## 5. CLI 命令

| 命令 | 作用 |
| --- | --- |
| `list` / `get` | 查库 |
| `add` / `update` / `delete` | 写库并同步 |
| `sync` | 重推 pending/error 或全量 |
| `health` | 聚合 upstreams + metrics |
| `export` / `import` | CSV / XLSX（应用层导出，不用 DuckDB） |
| `web` | 启动内嵌 Web |

全局 flag：`--admin`、`--store`、`--server`、`--format`。

## 6. Web

- `//go:embed` 嵌入静态页与模板  
- 代理列表（含健康徽章）、编辑 `route_json`/表单、审计查看、导出  
- 健康状态短轮询或手动刷新  

### 安全基线 B

- 默认绑定 `127.0.0.1`  
- 监听非回环地址时：**未配置管理员密码/Token 则拒绝启动**  
- 登录后 HttpOnly Cookie 会话；写操作需已登录  
- 凭证来自 flag 或环境变量（如 `CADDYDIAL_WEB_TOKEN`），不明文入库  
- 文档强调：Caddy Admin 与 Web 均勿对公网裸奔  

## 7. 部署（双镜像）

```text
用户流量 → Caddy（流量面：TLS、反代、健康踢除）
                ↑ Admin API（仅 Docker 内网）
本应用镜像（控制面：Web/CLI、SQLite volume）
```

- Compose：`caddy` + `caddydial`  
- 应用 `--admin http://caddy:2019`（服务名以 compose 为准）  
- 启动后对库做 sync，保证 Caddy 重启后从 SQLite 重建路由  

## 8. 错误处理

- 所有外部调用带 context 超时  
- 同步失败不静默：状态落库，支持重试  
- CLI 非 0 退出；Web 返回明确错误  

## 9. 测试

- store：CRUD、`route_json` 原样、audit  
- caddyapi：httptest mock  
- service：同步失败 → error；disabled → DELETE  
- web：未鉴权写 401；对外 listen 无 Token 启动失败  

## 10. 技术选型摘要

| 项 | 选择 |
| --- | --- |
| 语言 | Go 1.27+ |
| CLI | 标准库 flag 子命令分发 |
| Web | net/http + embed |
| 存储 | SQLite 落盘 + WAL |
| 导出 | CSV 标准库；XLSX 轻量库（如 excelize） |
| 架构 | 轻量分层，非 Clean/Hexagonal |
| DI | 手工构造 + functional options |
