# caddydial

Caddy 反向代理控制面：通过 Admin API 管理代理条目，提供 CLI 与可选内嵌 Web。  
配置以 SQLite 为源，同步到 Caddy；业务流量仅由 Caddy 处理。

## 要求

- Go 1.27+
- 可访问的 Caddy Admin API

## 状态

仓库已初始化；实现按本地设计说明推进（`docs/superpowers/` 仅供本地/AI 使用，不纳入版本库）。
