# zhiliao-gateway 改动记录

> 每轮完成改动后在此追加记录，新条目放在最前。

## 2026-09-04 为关闭Cloudflare代理补齐www主域规范化

- `.env.example` **+2/-0**：新增 `WWW_SERVER_NAME` 的 `CHANGE_ME_*` 占位，并注明它是 `MAIN_SERVER_NAME` 的规范化跳转别名。
- `docker-compose.yml` **+1/-0**：把 `WWW_SERVER_NAME` 显式传入官方Nginx模板渲染环境；本仓库未启用 `NGINX_ENVSUBST_FILTER`，因此没有新增并不存在的白名单配置。
- `nginx/gateway.conf.template` **+12/-1**：HTTP→HTTPS主机名列表纳入www；新增独立HTTPS块，以301永久跳转到主域并由 `$request_uri` 保留路径与查询串。四个业务反代块和两个default块未改。
- `README.md` **+4/-4**：路由说明与上线验收同步为五个配置主机名，明确www只做主域规范化，四个业务主机名仍各自回源。
- `CHANGELOG.md` **+13/-0**：新增本条普通工作记录，不写版本号、不暗示已经打标或部署。
- **改动动因**：这是关闭Cloudflare代理的前置条件；此前www→主域跳转规则住在Cloudflare而非本仓库，直连gateway没有对应server块。它是“配置住在别处、本地仓库看不见”的实例，代理一关便会静默退回default 403。
- **本场验证**：`docker compose --env-file .env.example config --quiet` 通过；官方 `nginx:stable-alpine` 完成envsubst后真实 `nginx -t` 通过，渲染结果无字面 `${WWW_SERVER_NAME}`。www实测返回301，`Location=https://main.example.com/deep/path?alpha=1&beta=2`，路径与查询串完整保留。
- **既有路由回归证据**：主域请求返回502，但访问日志原文为 `upstream_addr="127.0.0.1:31001" upstream_status="502"`，决定性证明它仍命中hub反代块；未知Host实测403，日志为 `upstream_addr="-" upstream_status="-"`，证明default拒绝仍在且没有访问上游。
- **测试夹具边界**：Docker Desktop的多个host-network容器不能互见回环假后端，第一次主域探测因此得到502；未修改仓库来迎合夹具，而是按实际访问日志判定路由归属。全部测试容器与临时证书均已清理。
- **未验证边界**：本轮不连接服务器、不部署；真实通配符证书、真实域名以及关闭Cloudflare代理后的公网行为均未验证。

## Git标签 v0.1 - 2026-09-02

- **本仓库的首个标签。覆盖 3 条工作条目、4 个提交**，其中 1 个提交属本轮存档动作本身（本条存档条目），**不是 4 件工作**：
  - 2026-09-02 建立唯一前端 gateway 初始骨架（`16ab583`）
  - 2026-09-02 校准全局上传上限与扫描噪音限流边界（`58e10fb`）
  - 2026-09-02 补齐证书续期与 VPC 源 IP 承重前提（`2dbac67`）
- **为什么是 `v0.1` 而不是 `v1.0`**：功能骨架已完整，但**从未在生产运行过一秒**。`v1.0` 会暗示「已验证可用」，而本仓库最关键的几项恰恰全部未验——真实客户端 IP 还原、真实证书与域名、SSE 90 秒长连接、证书续期触发 reload。在这些验过之前，用 `v1.0` 是把「写完了」说成「能用了」。
- ⚠️ **未验清单（不得当作已通过）**：本机 Docker 实测**只覆盖到容器客户端一跳**——即请求从同一 Docker 网络内的容器发出。以下全部未验证：真实公网客户端地址的还原是否正确、真实证书与真实域名下的行为、真实后端接入、SSE 90 秒长连接是否被中断、证书续期后 reload 是否按预期触发。**本条目不宣称任何一项已通过。**
- **本仓库当前没有配置任何 CI**：`.github/` 目录不存在，GitHub Actions 无任何运行记录。因此本标签**没有 CI 结论可引用**，与主站仓库不同。是否接入 CI 属后续独立事项，本轮未做也不顺带做。
- **存档前例行核对**：全仓 4 个提交共 645 行新增；bcrypt 哈希、疑似密钥赋值、服务器绝对路径、开发机盘符路径、未跟踪文件、二进制改动均 **0 命中**；无真实主机名（`host.docker.internal` 为 Docker 提供的固定 DNS 名）。IPv4 字面量为 `127.0.0.1`、`0.0.0.0` 与 `172.18.0.3`——末者是技术规格文档中引用的 **Docker 桥接 RFC1918 地址**，用于说明知天生产环境「api 容器把反代容器地址当成客户端」这一既有缺陷，非本网关运行地址，也不含任何公网或服务器信息。

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
