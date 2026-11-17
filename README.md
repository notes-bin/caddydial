# caddydial

Caddy 反向代理控制面：通过 Admin API 管理代理条目，提供 CLI 与可选内嵌 Web。  
配置以 SQLite 为源，同步到 Caddy；业务流量仅由 Caddy 处理。

设计文档见 [`docs/superpowers/specs/2026-11-17-caddydial-design.md`](docs/superpowers/specs/2026-11-17-caddydial-design.md)。

## 要求

- Go 1.27+
- 可访问的 Caddy Admin API

## 状态

仓库已初始化；实现按设计文档推进。
