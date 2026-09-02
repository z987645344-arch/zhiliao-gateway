# zhiliao-gateway 改动记录

> 每轮完成改动后在此追加记录，新条目放在最前。

## 2026-09-02 建立唯一前端 gateway 初始骨架

- 新增基于 `nginx:stable-alpine` 与 host 网络模式的单服务 Compose 配置，按四个环境变量主机名把 HTTPS 请求路由到两组回环后端。
- 新增宽松/收紧两档请求限速、每 IP 并发限制、SSE 长连接设置、统一错误页、独立 gateway 健康端点、拓扑诊断日志与未知 Host 默认拒绝。
- README 记录真实客户端 IP 的逐跳传递契约、后端回环绑定的安全承重性质、知了hub 恢复探针边界、共同故障域和证书续期 reload 约定。
- 静态核对确认 6 个反代位置均具备 `proxy_buffering off`、300 秒读取超时、每 IP 并发限制和四行转发头；4 个 HTTPS 主机块均有直接健康端点与统一错误页；模板需要的 6 个变量与 Compose 注入项逐项一致，`.env`/密钥/证书忽略规则生效，配置花括号配对且 `git diff --check` 通过。
- 本机没有可调用的 Docker、Docker Compose 或 OpenSSL，因此无法执行 `docker compose config --quiet`、官方 Nginx 容器 `nginx -t`、容器启动、四主机路由、default server、429 阈值及 host 模式 `$remote_addr` 实测。本轮未连接服务器、未部署；真实证书、真实域名、真实后端、SSE 90 秒和证书续期 reload 同样留待具备 Docker 的环境或服务器验收，不能把当前静态审阅视为这些行为已经通过。
