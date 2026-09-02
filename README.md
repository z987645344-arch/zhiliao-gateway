# zhiliao-gateway

`zhiliao-gateway` 是单公网 IP 场景下唯一监听 80/443 的前端网关。它只负责终止 TLS、按 `server_name` 路由和转发，不包含界面、业务逻辑或主动后端健康探测。

## 拓扑与安全边界

四个主机名通过环境变量配置，其中两项转发到 `127.0.0.1:${HUB_BACKEND_PORT}`，另外两项转发到 `127.0.0.1:${TIAN_BACKEND_PORT}`。仓库只保存 `CHANGE_ME_*` 占位符；真实主机名、IP、证书路径和凭据不得进入 Git。

gateway 使用 `network_mode: host`，理由是：

1. host 模式下 `$remote_addr` 应直接来自客户端连接，不经过普通 Docker 桥接 NAT；上线前仍必须从真实外部客户端观察日志验证，不能把设计判断写成实测结论。
2. 容器可以直接访问宿主机的 `127.0.0.1:<后端端口>`，无需 `host.docker.internal` 或跨项目共享 Docker 网络。

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

- 四个 HTTPS `server` 块严格按主机名路由；不匹配的 Host 或直连 IP 请求由 `default_server` 直接拒绝。
- 80 端口只为四个已配置主机名执行 HTTPS 跳转，后端不再重复跳转。
- 宽松限速为每 IP 30 请求/秒、`burst=60 nodelay`；收紧限速为每 IP 2 请求/秒、`burst=10 nodelay`；并发上限为每 IP 24。
- 知了hub 的 `/admin` 与 `/api` 使用收紧档；知天的收紧路径暂不配置，模板中保留醒目 TODO，等待其指挥师确认。不能猜测 `/chat/stream` 等 SSE 路径。
- 所有代理位置关闭响应缓冲，并把读取超时设为 300 秒，为 30–90 秒的 SSE 留出余量。
- 429、502、503、504 共用 Nginx 镜像内的极简静态错误页；页面不显示后端名称或拓扑。
- `/gateway-health` 由 gateway 直接返回 200，不访问任何后端；容器 healthcheck 按要求执行 `nginx -t`。gateway 不主动探测后端。
- 访问日志包含客户端地址、入站 XFF、Host、状态码、请求耗时、上游地址和上游状态。

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

## 故障域

**gateway 挂掉意味着两个站点同时不可访问。** 这是只有一个公网 IP 的必然代价。采用独立 gateway 的收益是两个项目的常规部署不再触碰公网 80/443，使“哪些操作可能引发共同故障”更少，并不代表共同故障域变小。

## 上线验收

上线时必须在真实服务器补齐以下验证，本地静态审阅不能替代：

- 真实证书下四个主机名无需跳过 TLS 校验即可访问，并分别命中预期上游。
- 从真实外部客户端请求后，gateway 日志中的 `$remote_addr` 是该客户端地址；继续核对两级后端最终采用的 IP 分桶键。
- 无匹配 Host 的请求被默认服务器拒绝，超阈值请求返回 429 而不是 503。
- SSE 连接持续至少 90 秒不被 gateway 中断。
- Certbot 续期后 reload 成功；另行制造 reload 失败，确认失败状态可被察觉且不掩盖证书已经续期的事实。
