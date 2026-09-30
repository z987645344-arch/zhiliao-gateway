# zhiliao-gateway

`zhiliao-gateway` 是单公网 IP 场景下唯一监听 80/443 的前端网关。它只负责终止 TLS、按 `server_name` 路由和转发，不包含界面、业务逻辑或主动后端健康探测。

## 拓扑与安全边界

五个主机名通过环境变量配置：主域与小作坊域转发到 `127.0.0.1:${HUB_BACKEND_PORT}`，另外两个业务域转发到 `127.0.0.1:${TIAN_BACKEND_PORT}`，`WWW_SERVER_NAME` 则永久跳转到 `MAIN_SERVER_NAME` 并保留路径与查询串。仓库只保存 `CHANGE_ME_*` 占位符；真实主机名、IP、证书路径和凭据不得进入 Git。

gateway 使用 `network_mode: host`，理由是：

1. host 模式下 `$remote_addr` 应直接来自客户端连接，不经过普通 Docker 桥接 NAT；上线前仍必须从真实外部客户端观察日志验证，不能把设计判断写成实测结论。
2. 容器可以直接访问宿主机的 `127.0.0.1:<后端端口>`，无需 `host.docker.internal` 或跨项目共享 Docker 网络。

本地 Docker 验证已从独立桥接客户端访问 host 网络 gateway，并确认 gateway 传出的 X-Real-IP/XFF 与该客户端实际容器地址完全一致；这证明了 Docker Linux 网络命名空间内的行为，但**不等于公网真实客户端验收**。生产部署后仍必须从服务器外部请求并核对日志。

> ⚠️ **两个后端都必须只绑定回环端口。这个绑定是安全承重，不是性能选择。** gateway 拓扑成立的前提是后端不可从公网直达。任一后端若改回绑定 `0.0.0.0`，它会以明文 HTTP 直接暴露在公网：TLS 只在 gateway 终止一次，后端不再持有证书，也不再负责 HTTP→HTTPS 跳转。这类误配置不会报错，站点看起来仍可能完全正常，因此两个项目的部署清单都必须验证实际监听地址。

### 真实客户端 IP 的传递契约

gateway 固定传递 `X-Real-IP`、`X-Forwarded-For`、`X-Forwarded-Proto` 和 `Host`。后端必须根据下表独立推导可信代理配置，不能盲目信任所有转发头：

| 跳 | 组件           | 它看到的 $remote_addr                         | 它发出的 XFF |
| - | ------------ | ------------------------------------------ | -------- |
| 0 | 客户端          | —                                          | 可能伪造 F   |
| 1 | gateway :443 | 真实客户端 C                                    | F, C     |
| 2 | 后端 nginx     | ⚠️ **桥网关地址**（如 172.x.0.1），**不是 127.0.0.1** | F, C, C  |
| 3 | 应用层          | 由 XFF 末位取 C                                | —        |

⚠️ **第 2 跳不是 127.0.0.1。** 后端从宿主回环收到连接时，经 `docker-proxy` 后看到的是自己 Compose 网络的桥网关地址。后端 Nginx 与应用层只能信任明确的代理跳，不能使用“信任所有来源”的配置，否则访客可伪造 XFF 绕过按 IP 限流。

> ⚠️ 知了hub 的恢复守卫使用 `RESTORE_PROBE_URL` 判断后台是否已停止。该地址必须直连 hub Nginx 的回环端口，**不得指向 gateway**。gateway 是哑代理；gateway 自身故障也会导致非 200，若用它作探针，恢复守卫可能把“gateway 挂了”误判为“hub 后台已停”并错误放行。

## 路由与流量控制

- 四个业务 HTTPS `server` 块严格按主机名路由；独立的 www HTTPS 块只向主域返回 301，不反代、不提供静态文件。不匹配的 Host 或直连 IP 请求仍由 `default_server` 直接拒绝。
- 80 端口只为五个已配置主机名执行 HTTPS 跳转，后端不再重复跳转；www 随后由其 HTTPS 块规范化到主域。
- 已知主机名的宽松档为每 IP 100 请求/秒、`burst=200 nodelay`；知了hub `/admin` 与 `/api` 的收紧档为每 IP 10 请求/秒、`burst=20 nodelay`；二者每 IP 并发上限均为 200。知天的收紧路径暂不配置，模板中保留醒目 TODO，等待其指挥师确认。
- 未知 Host 与直连 IP 才使用严格档：每 IP 1 请求/秒、`burst=2 nodelay`、并发上限 8；普通未知请求被拒绝，超过阈值时返回 429。
- 上述数字是缺少真实流量基线时的首版取值，并非已验证的最终阈值。上线后必须按 429 日志复核误伤与扫描噪音，再用真实证据调整。
- gateway 的职责是挡扫描噪音，不是重新实现业务限流。两个后端应用层已有按账号、角色或认证场景划分的精细分桶；若 gateway 在同一路径叠加更严的限制，实际先触发的是 gateway，等于架空后端分桶。因此采用“已知主机名从宽、default server 从严”的边界。
- 知天已有两条 SSE：`POST /chat/stream` 用于30–90秒对话，`GET /tasks/{task_id}/stream` 用于时长随文档大小变化的入库进度。两者都不能被误归到未知的收紧路径。
- 所有代理位置关闭响应缓冲，并把读取超时设为 300 秒，为 30–90 秒的 SSE 留出余量。
- gateway 在 HTTP 层设置 `client_max_body_size 300m`（v0.2.2 起，配合站内 Unity WebGL 的 300 MB 上传上限），取当前两个后端上限的较大值，只为避免最外层提前返回 413；具体上传策略仍由各后端自身执行。gateway 的值必须始终不小于任一后端，任何项目提高上传上限时都必须回来复核这一项。
- 429、502、503、504 共用 Nginx 镜像内的极简静态错误页；页面不显示后端名称或拓扑。
- `/gateway-health` 由 gateway 直接返回 200，不访问任何后端；容器 healthcheck 按要求执行 `nginx -t`。gateway 不主动探测后端。
- 访问日志包含客户端地址、入站 XFF、Host、状态码、请求耗时、上游地址和上游状态。
- 五个已配置主机名的 HTTPS 响应（含 www 的 301）都带 `Strict-Transport-Security: max-age=31536000`；80 端口与未知 Host 的兜底块不发送。暂不加 `includeSubDomains` 与 `preload`：前者会把同域名下所有子域一并锁定为 HTTPS，后者一旦进入浏览器预加载列表很难撤回，等所有子域都确认长期 HTTPS 后再评估。

## 容器权限

- 容器以 `cap_drop: ALL` 丢弃全部 Linux capability，只加回四项：`NET_BIND_SERVICE`（在 host 网络下绑定 80/443）、`SETUID` 与 `SETGID`（master 进程切换到 `nginx` 用户运行 worker）、`CHOWN`（启动时把 `/var/cache/nginx` 下的临时目录交给 worker）。配合 `no-new-privileges`，worker 以非 root 的 `nginx` 用户处理请求。
- 只加回 `NET_BIND_SERVICE` 是常见误配：官方镜像会在启动时报 `chown("/var/cache/nginx/client_temp", 101) failed (1: Operation not permitted)` 并退出。

## 配置与启动

1. 复制 `.env.example` 为 `.env`，把所有 `CHANGE_ME_*` 替换为部署现场值。不得提交 `.env`。
2. 确认两个后端端口只监听 `127.0.0.1`，并由各自项目完成可信代理配置。
3. 在不会泄露展开变量的前提下检查 Compose：

   ```sh
   docker compose config --quiet
   ```

4. 启动并确认配置与容器状态：

   ```sh
   docker compose up -d
   docker compose exec -T gateway nginx -t
   docker compose ps
   ```

模板挂载为 `/etc/nginx/templates/default.conf.template`，由官方 Nginx 镜像启动脚本执行 `envsubst`。本项目不设置 `NGINX_ENVSUBST_FILTER`，避免新增变量后忘记加入过滤白名单而把 `${VAR}` 原样留给 Nginx。

## 证书续期与 reload

Certbot 应通过 deploy hook 在成功续期后触发 gateway reload，示例中的仓库路径仍是占位符：

```sh
certbot renew --deploy-hook 'docker compose --project-directory CHANGE_ME_GATEWAY_REPO exec -T gateway nginx -s reload'
```

不得在 hook 后添加 `|| true` 或吞掉退出码。deploy hook 发生在证书已成功续期之后：reload 失败不会撤销已经续好的证书，但属于独立故障，必须由 Certbot/systemd 的失败状态或告警发现并修复；修复前 gateway 仍可能继续提供旧证书。

续期走 DNS-01——这是通配符证书的必然要求，因此不需要
`/.well-known/acme-challenge/`，gateway 独占 80/443 与续期不冲突。
⚠️ 若将来改用非通配证书、或新增不在通配范围内的域名，验证会退回 HTTP-01，
必须先在 gateway 增加挑战路径，否则续期静默失败、直到证书过期站点全挂才被发现。

## 故障域

**gateway 挂掉意味着两个站点同时不可访问。** 这是只有一个公网 IP 的必然代价。采用独立 gateway 的收益是两个项目的常规部署不再触碰公网 80/443，使“哪些操作可能引发共同故障”更少，并不代表共同故障域变小。

## 上线验收

上线时必须在真实服务器补齐以下验证，本地静态审阅不能替代：

⚠️ 上机后的第一项验证，必须是「gateway 访问日志里的 `$remote_addr`
是不是访问者的真实出口 IP」。公网流量经 VPC NAT 进入本机，
若该 NAT 不保留源 IP，`$remote_addr` 会变成一个固定地址，于是：
所有限速分桶塌成一个、传给两个后端的 XFF 末位全一样、
两个项目的应用层按 IP 分桶同时失效——**而且一切看起来完全正常，不报任何错**。
验法：从开发机访问一次，比对日志行首与开发机的真实出口 IP。
**这一条不过，本文件里所有限速阈值都没有意义。**

- 真实证书下五个主机名无需跳过 TLS 校验即可访问；四个业务主机名分别命中预期上游，www 返回指向主域且保留路径与查询串的 301。
- 从真实外部客户端请求后，gateway 日志中的 `$remote_addr` 是该客户端地址；继续核对两级后端最终采用的 IP 分桶键。
- 无匹配 Host 的请求被默认服务器拒绝，超阈值请求返回 429 而不是 503。
- SSE 连接持续至少 90 秒不被 gateway 中断。
- Certbot 续期后 reload 成功；另行制造 reload 失败，确认失败状态可被察觉且不掩盖证书已经续期的事实。

## License

当前仓库未附带开源许可证，默认保留全部权利；公开复用前请先联系项目作者。
