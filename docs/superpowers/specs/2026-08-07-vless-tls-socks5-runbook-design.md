# 脱敏 VLESS + TLS + SOCKS5 运维 Runbook 设计

## 目标

新增一篇面向具备 Linux、SSH、Cloudflare 与代理客户端基础的工程师的公开博客文章。文章应让读者以自己的域名、VPS 与 SOCKS5 服务凭据复现部署、验证和故障定位流程，同时不暴露任何生产身份信息。

## 发布边界

- 站点源码分支为 main；GitHub Actions 将构建产物部署到 gh-pages。
- 正文创建于 \_posts/2026-08-07-vless-tls-socks5-runbook.md，不直接编辑 gh-pages 的生成 HTML。
- 所有域名、IP、端口、令牌、用户名、密码和面板路径使用角括号占位符。

## 文章结构

1. 版本矩阵与安全边界。
2. 目标架构与流量方向。
3. Cloudflare DNS、TLS 入站和订阅的部署条件。
4. Clash Verge Rev 的订阅导入、多订阅隔离和配置文件脚本。
5. 服务端、客户端与 SOCKS5 的分层验证命令及结果判定。
6. REALITY 与 TLS 选择、错误映射、住宅代理入口轮换和交接清单。

## 验收标准

- 文章带有 Jekyll post front matter，能被本站构建识别。
- 文中不含本次部署的真实域名、IP、令牌或凭据。
- 每个关键部署环节至少有一个可执行验证命令与预期解释。
- 明确指出 main 是源码发布入口，gh-pages 是自动生成的发布分支。
