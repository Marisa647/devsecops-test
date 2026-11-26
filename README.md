# DevSecOps Test Task

Розгортання захищеного веб-сервера з WAF та централізованим збором логів.

## Що зроблено

- **ubu1** — веб-сервер (Nginx + ModSecurity + OWASP CRS) у Docker-контейнері
- **ubu2** — моніторинг (Grafana + Loki) + WireGuard VPN
- Firewall за принципом default deny
- SSH тільки по ключах, без root і паролів
- SSH до ubu1 доступний лише через WireGuard
- Збір логів nginx/ModSecurity у Loki + відображення в Grafana
- Алерт у Telegram при >5 блокуваннях з однієї IP за короткий проміжок часу

Основні проблеми та їх вирішення описані в [docs/ISSUES.md]
