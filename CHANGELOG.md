# zhiliao-gateway 改动记录

> 每轮完成改动后在此追加记录，新条目放在最前。

## 2026-09-02 补齐证书续期与 VPC 源 IP 承重前提

- README 明确通配符证书必须使用 DNS-01，因此不需要 ACME HTTP 挑战路径，gateway 独占 80/443 与当前续期方式不冲突；若未来退回 HTTP-01，必须先增加挑战路由。
- 上线验收的第一项固定为从开发机发出真实外部请求，比对 gateway 日志行首 `$remote_addr` 与真实出口 IP；VPC NAT 若改写源 IP，gateway 与两个后端的所有按 IP 分桶都会静默失效。
- 本轮只补充两条部署承重说明，未改 Nginx 模板、Compose 或环境变量模板，也未执行服务器验证或部署。

## 2026-09-02 校准全局上传上限与扫描噪音限流边界

- 在 gateway 的 HTTP 层补上 `client_max_body_size 100m`，使最外层上限不小于现有后端最大值；它只避免 gateway 在 1MiB 默认值处提前返回 413，不替代后端自己的上传大小策略。
- 已知主机名改为宽松档 100r/s、burst 200、每 IP 并发 200；知了hub `/admin` 与 `/api` 使用 10r/s、burst 20；未知 Host/default server 单独使用 1r/s、burst 2、并发 8，并在超过阈值时返回 429。首版数字没有真实流量依据，上线后必须按 429 日志复核。
- 知天收紧路径仍不擅自填写，TODO 同时列出 `POST /chat/stream` 与 `GET /tasks/{task_id}/stream` 两条 SSE；通用 300 秒读取超时保持不变。
- README 明确 gateway 只挡扫描噪音、不重新实现业务限流，以及任何后端提高上传上限时都必须同步复核 gateway 上限。
- **Docker真实验证**：`docker compose config --quiet` 通过；官方 `nginx:stable-alpine` entrypoint 完成 envsubst 后，真实 `nginx -t` 成功，运行容器保持 healthy。容器内最终配置显示100m上限、三档rate/burst、两档并发数及两条SSE注释均为预期值。
- **行为验证**：四个占位Host分别命中预期的hub/tian测试后端；2MiB真实POST穿过gateway并返回200，未再被默认1MiB拦截；同一客户端并发80次时，general为80次200，sensitive为21次200与59次429；未知Host连续请求先返回403，超限后返回429，未出现503；`/gateway-health`直接返回200。
- **客户端地址边界**：独立Docker桥接客户端的实际容器地址与gateway传出的X-Real-IP/XFF完全一致，证明Docker Linux网络内host模式未把它改写成gateway地址。Docker Desktop没有把host网络容器的443暴露给Windows宿主，因此本轮仍未获得公网或Windows外部客户端证据；真实证书、真实域名/后端、SSE 90秒及Certbot reload继续留待服务器验收。

## 2026-09-02 建立唯一前端 gateway 初始骨架

- 新增基于 `nginx:stable-alpine` 与 host 网络模式的单服务 Compose 配置，按四个环境变量主机名把 HTTPS 请求路由到两组回环后端。
- 新增宽松/收紧两档请求限速、每 IP 并发限制、SSE 长连接设置、统一错误页、独立 gateway 健康端点、拓扑诊断日志与未知 Host 默认拒绝。
- README 记录真实客户端 IP 的逐跳传递契约、后端回环绑定的安全承重性质、知了hub 恢复探针边界、共同故障域和证书续期 reload 约定。
- 静态核对确认 6 个反代位置均具备 `proxy_buffering off`、300 秒读取超时、每 IP 并发限制和四行转发头；4 个 HTTPS 主机块均有直接健康端点与统一错误页；模板需要的 6 个变量与 Compose 注入项逐项一致，`.env`/密钥/证书忽略规则生效，配置花括号配对且 `git diff --check` 通过。
- 本机没有可调用的 Docker、Docker Compose 或 OpenSSL，因此无法执行 `docker compose config --quiet`、官方 Nginx 容器 `nginx -t`、容器启动、四主机路由、default server、429 阈值及 host 模式 `$remote_addr` 实测。本轮未连接服务器、未部署；真实证书、真实域名、真实后端、SSE 90 秒和证书续期 reload 同样留待具备 Docker 的环境或服务器验收，不能把当前静态审阅视为这些行为已经通过。
