# zhiliao-gateway 技术规格

> 2026-09-02 由统筹师出具，交 gateway 指挥师起草执行指令。
> **本文无真实主机名、IP、凭据**，可安全引用。真实值一律走 `.env`。

## 一、这是什么

新服务器只有 **1 个公网 IP**，而两个项目共四个主机名都要走 443。
`zhiliao-gateway` 是唯一持有 443 的进程，按 `server_name` 把流量分给两个项目。

**它没有界面、没有业务逻辑、不知道后面在跑什么。** 只做三件事：终止 TLS、看域名、转发。

## 二、拓扑

```
公网 :443
   │
   ▼
zhiliao-gateway（唯一持有 443，TLS 在此终止）
   │ 内部 HTTP，不再加密
   ├─ ${MAIN_SERVER_NAME}   ─┐
   ├─ ${LAB_SERVER_NAME}    ─┴─► 127.0.0.1:${HUB_BACKEND_PORT}   → 知了hub 现有 nginx
   ├─ ${AGENT_SERVER_NAME}  ─┐
   └─ ${ADMIN_SERVER_NAME}  ─┴─► 127.0.0.1:${TIAN_BACKEND_PORT}  → 知天现有 reverse-proxy
```

**两个项目现有的 nginx 一个都不拆**，只是从"对外前端"降级为"后端"，绑到回环端口。
⚠️ **2026-09-02 更正：原写「零仓库改动」，对知了hub 不成立。** 由知了hub 指挥师查出、统筹师核实：

| 项目 | HTTP 端口的行为 | 能否零仓库改动 |
|---|---|---|
| **知了hub** | `deploy/nginx.conf:65-68` 的 `listen 80` **只有 `return 301 https://`，不供内容**；内容全在两个 `listen 443 ssl` 块里 | ❌ **必须改仓库**。gateway 转明文到 :80 会造成**无限重定向环**；转 :443 又违反 §3「只终止一次」且后端仍需证书 |
| **知天** | `compose-nginx.conf.template:45` 的 `listen 8080` **有 `proxy_pass`，供内容**，只是被 `if ($force_https != off) { return 301 }` 守卫包着；而该守卫由 `set $force_https ${ZHITIAN_FORCE_HTTPS};` 驱动 | ✅ **可零仓库改动**，`.env` 设 `ZHITIAN_FORCE_HTTPS=off` 即可 |

⚠️⚠️ **但知天这条带一个危险耦合，必须写死在部署清单里。** 该模板第 60-63 行的注释写明：守卫刻意写成 `!= off` 而非 `= on`，是为了让笔误倒向生产行为——因为在**旧拓扑**下把它设成 `off`，等于「把企业管理后台重新摆回公网 IP 的明文 HTTP 根路径」。

**在 gateway 拓扑下它之所以安全，唯一依据是 8080 端口只绑回环。**
→ `ZHITIAN_FORCE_HTTPS=off` 与 `SERVER_PUBLIC_IP=127.0.0.1` **是一对，缺一不可**；
→ 任何人日后把绑定改回 `0.0.0.0` 而没同时把开关改回 `on`，**管理后台立刻以明文 HTTP 暴露在公网,且不报任何错**。
→ 这两个变量必须在 `.env` 里相邻放置并加注释说明其耦合关系。

另注：`location = /api/ready` 刻意不带该守卫（反代自身健康检查打的就是它），改动时勿动。

知天侧的端口降级仍是纯 `.env` 工作，归统筹师；**知了hub 侧的 nginx 改造进 P0-a，归执行者。**

## 三、TLS

| | |
|---|---|
| 证书 | Let's Encrypt 通配符，覆盖主域 + `*.` 全部子域 |
| 路径 | `${TLS_CERT_PATH}` / `${TLS_KEY_PATH}`（指向 `/etc/letsencrypt/live/<域>/`，只读挂载） |
| 续期 | `certbot.timer` 已配好并验证；**续期后 gateway 需要 reload** —— 请设计一个 renew hook |
| 后端 | **不再需要证书**，两个项目的 `TLS_CERT_PATH` 可留空 |

⚠️ **只终止一次**。内部走明文可接受，因为流量不出这台机器。

## 四、⚠️ 真实客户端 IP 的传递契约 —— 本规格最要紧的一节

gateway 的加入使链路多了一跳。**这直接决定知了hub 那 22 条 `set_real_ip_from` 怎么改。**

**gateway 侧必须做的：**

```nginx
proxy_set_header X-Real-IP        $remote_addr;
proxy_set_header X-Forwarded-For  $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header Host             $host;
```

**后端侧对应要做的（归各项目指挥师）：**

```
知了hub   把 22 条 CF 网段替换为 gateway 的可信来源（不是删空）
          real_ip_header 由 CF-Connecting-IP 改为 X-Forwarded-For
          TRUST_PROXY_HOPS 需按新拓扑重新推导，不得盲目改数字
知天      set_real_ip_from 原为 0 条；需确认其应用层限流是否按 IP 分桶，
          若是，同样要处理这一跳
```

⚠️ **删空而不替换的后果不是日志难看**：`$remote_addr` 会变成 gateway 的地址，
后端所有按 IP 分桶的限流器会退化成**所有人共用一个桶**——
知了hub 的密码、TOTP、设备认证、反馈提交四个限流器会同时失效，
**等于暴力破解防护被拆掉**。此判断由知了hub 指挥师查证得出。

### ⚠️ 后端能不能读到转发头，是另一个独立环节 —— 已实证有一处是断的

XFF 传对只是第一环。**后端框架还必须信任这一跳，否则头传了也白传。**

知天已实证存在这一断点（2026-09-02 由知天指挥师查出，统筹师核实）：

```
Dockerfile:93   uvicorn 启动命令无 --forwarded-allow-ips（全仓 0 处）
                该项默认 127.0.0.1，而反代是另一个容器
main.py:328     未认证请求分桶键 = request.client.host
生产实证        api 容器收到的对端地址 172.18.0.3 共 23702 次
                = 反代的容器 IP，全部匿名请求落进同一个桶
```

**后果**：五个端点（登录、注册×2、忘记密码、发验证码）各 `10/hour`，
**全系统共用一个桶**——任何人一小时敲满 10 次登录，所有人该小时都登不上。
一行请求即可触发的拒绝服务。

⚠️ **修复有个反直觉的陷阱**：不能写成 `--forwarded-allow-ips=*`。
那会把"所有人共用一个桶"变成"任何人都能伪造 XFF 自选桶"，**限流直接可绕过，比现状更糟**。
必须限定为可信代理地址。

⚠️ **架构 C 下是两跳**，`X-Forwarded-For` 里会有两个地址，取哪一个与框架版本行为有关。
**这一处不能靠"加了参数应该就对了"，必须实际观察分桶键的取值来验证。**

**对 gateway 的含义**：本组件的 XFF 传递方式，直接决定两个后端各自要如何配置信任。
gateway 定稿后必须把「链路上有几跳、每跳的地址是什么」明确写进 README，
供两个项目的指挥师据此推导各自的可信代理配置。

## 五、速率限制

**放在 gateway，两个项目都不必各自加。** 现状：两个项目的 nginx 层限流均为 0 条。

分两档，具体数字由指挥师定，以下为背景：

| 档 | 覆盖 | 背景 |
|---|---|---|
| 宽松 | 静态页面、公开内容 | 旧机日志每天有稳定扫描器噪音（探 `/error_log.php`、`/.env`、`/Public/css/errorCss.css` 之类）。CF 撤走后这些直接打到源站 |
| 收紧 | `/admin`、`/api` 等管理与接口路径 | 两个项目的应用层已有认证限流；gateway 这一层是防资源消耗，不是防撞库 |

⚠️ **知天的对话接口是 SSE 长连接**（`/chat/stream`，单次可开 30–90 秒）。
`limit_conn` 与超时设置必须为它留出余量，否则会在对话中途掐断。

## 六、后端连接方式

**用回环端口，不用共享 Docker 网络。**

理由：共享网络要求两个项目的 compose 声明 external 网络 → 又变成仓库改动。
回环端口只需各自 `.env` 改两个变量。

⚠️ **实施要点**：gateway 在容器里访问宿主机回环，需要 `host.docker.internal`
（配 `extra_hosts: host-gateway`）或 host 网络模式。**两种都要实测，不要假设。**

## 七、禁止出现在仓库里的内容

```
任何真实主机名   —— 尤其管理后台主机名，这是长期红线
任何真实 IP
任何证书路径的绝对形式（走 env）
```

`.env.example` 一律用 `CHANGE_ME_*` 占位。

## 八、仓库结构建议

```
docker-compose.yml
nginx/gateway.conf.template      四个 server 块 + 限速 zone + TLS
.env.example
README.md
CHANGELOG.md
```

**规模预计两百行左右**，且几乎不变——除非新增域名或调限速阈值。

## 九、需要 gateway 指挥师自己决定的

1. 限速的具体阈值与 burst
2. 502/503 时是否返回自定义错误页（默认 nginx 裸页很难看）
3. renew hook 的实现方式（certbot deploy-hook 还是别的）
4. 健康检查怎么做（gateway 自己的，以及它如何判断后端存活）
5. 日志格式（要包含真实客户端 IP，否则排查时看到的全是 gateway 自己）

## 十、验收要点

```
四个主机名分别能路由到正确后端
不加 -k 的 curl 全部通过（证书受信）
后端日志里记录的是真实客户端 IP，不是 gateway 的地址
SSE 长连接能开满 90 秒不被掐断
限速在超阈值时返回 429，正常访问不受影响
证书续期后 gateway 自动 reload
```

## 十一、故障域说明（写进 README）

**gateway 挂掉 = 两个站点同时不可访问。** 这是单公网 IP 的必然代价，
A/B/C 三种方案都躲不掉。选 C 的收益不是缩小故障域，
而是**让"什么操作会引发故障"变少**——两个项目的常规部署都碰不到 gateway。
