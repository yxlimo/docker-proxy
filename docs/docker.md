# Docker Hub 代理

把 `registry-1.docker.io` 反代到 Vercel，让国内客户端不再直连被阻断的上游。

## 为什么需要

实测（家宽出口，中国电信上海）：

| 域名 | 用途 | 直连结果 |
| --- | --- | --- |
| `registry-1.docker.io` | API / manifest | **100% 失败**，TLS 握手被重置 |
| `auth.docker.io` | 换令牌 | **100% 失败**，同样 |
| `hub.docker.io` / `docker.io` | 网页 / index | **100% 失败**，同样 |
| `production.cloudfront.docker.com` | **blob 实际下载** | 部分阻断，见下文 |

失败形态是 TLS 层：带 SNI 的 ClientHello 发出后被 RST，`openssl s_client` 报
`Cipher is (NONE)`。改用干净 IP 直连同样失败，所以不是单纯的 DNS 污染。

`openssl s_client` 的对照实验确认阻断按 SNI 关键字触发：同一个 IP，
换不同的 SNI 域名，结果不同。

**关键结论**：`production.cloudfront.docker.com` 只是**概率性**阻断
（30 次采样 13 次成功，约 43% 成功率），而 `registry-1.docker.io` 是
**确定性**阻断（20 次采样 0 次成功）。也就是说：

- manifest / token 必须走代理（否则 100% 失败）
- blob 有机会直连 CDN（客户端自带重试，实测 6 次 nginx 拉取全部成功）

## 端点

| 路径 | 说明 |
| --- | --- |
| `GET /v2/` | registry 探测，返回 200 + `Docker-Distribution-Api-Version` |
| `GET /v2/{name}/manifests/{ref}` | manifest，按 name+ref+Accept 缓存 |
| `GET /v2/{name}/blobs/{digest}` | blob，返回 307 跳转或流式转发 |
| `GET /health` | 健康检查 |

`name` 可以含斜杠（`library/nginx`、`bitnami/redis`），解析时从右往左切分：
最后一段是 reference，倒数第二段是资源类型，其余是仓库名。

## 客户端配置

### podman

`/etc/containers/registries.conf`：

```toml
unqualified-search-registries = ["docker.io"]

[[registry]]
prefix = "docker.io"
location = "docker.io"

[[registry.mirror]]
location = "<你的代理域名>"
insecure = false
```

### Docker daemon

`/etc/docker/daemon.json`：

```json
{
  "registry-mirrors": ["https://<你的代理域名>"]
}
```

改完重启 daemon：`systemctl restart docker`。

> `registry-mirrors` 的路径前缀支持情况：moby 从 2013 年起就允许 mirror 带路径
> 前缀（`registry: allow mirror path prefix in config`），当前实现会把
> `<mirror>/<path>` 拼上 `/v2`。本项目直接挂在域名根路径（`/v2/...`），
> 不依赖路径前缀特性，兼容性最好。

### 验证

```bash
podman pull docker.io/library/alpine:3.19
```

## 认证模型

代理在**服务端**完成认证，客户端无需 `docker login`：

1. 客户端 `GET /v2/` → 代理返回 **200**（不带 `WWW-Authenticate`）
2. 客户端后续请求**不带** `Authorization` 头
3. 代理按 `scope=repository:{name}:pull` 向 `auth.docker.io` 换令牌，
   用服务端凭据（`DOCKERHUB_USERNAME` + `DOCKERHUB_TOKEN`）做 Basic 认证
4. 代理把 `Bearer {token}` 附加到上游请求

未配置服务端凭据时按匿名拉取，令牌仍由代理换取（只是不带 Basic 头）。

**令牌缓存**：上游签发的令牌有效期 300 秒（`expires_in`），按 scope 缓存，
提前 30 秒过期。客户端请求同一仓库不会反复打 auth 服务。

**401 重试**：上游返回 401 时强制换一次令牌重试，处理令牌提前失效的情况。

### 安全边界

配置了服务端凭据后，代理**能访问该账号的私有仓库**——等于把它们开放给
所有能访问代理的人。代理务必配合访问控制使用（见 [部署文档](deployment.md#waf配置-ip-白名单)）。

## 限速与配额

Docker Hub 的拉取限速**只统计 manifest 的 GET 请求**（blob 不计）：

| 身份 | 配额 | 归属维度 | 实测 `ratelimit-limit` |
| --- | --- | --- | --- |
| 匿名 | 100 / 6h | **出口 IP** | `100;w=3600` |
| Personal 账号 | 200 / 6h | **账号** | `200;w=3600` |

实测确认：不同出口 IP 的 `ratelimit-remaining` 独立计数；
使用账号凭据后 `docker-ratelimit-source` 变为用户名。

**这正是配服务端凭据的主要理由**：Vercel 出口 IP 是共享的，匿名配额
会被其他租户消耗；账号配额只算自己的。

实测「只有 manifest GET 计数」：

```
5 次 blob 307 之后  remaining = 100   ← blob 不消耗
1 次 manifest GET  remaining = 99    ← manifest 消耗
HEAD 请求            remaining = 不变  ← HEAD 不计数
```

所以 manifest 缓存是省配额的关键：命中缓存就不回源，不消耗配额。

## blob 的两种处理方式

### redirect（默认）

上游对 blob 返回 307 + `Location`（指向 `production.cloudfront.docker.com`
的签名 URL）。代理原样透传这个跳转，客户端自己直连 CDN 下载。

- **优点**：不消耗 Vercel 流量；实测 43% 的 TLS 阻断率由客户端重试消化
- **代价**：客户端到 CDN 的连接有约 43% 概率首次失败
- 签名 URL 有效期约 50 分钟，且每次请求签名不同 → **该响应不能缓存**
  （`Cache-Control: no-store`）

### proxy

代理自己跟随跳转并流式转发内容。

- **优点**：客户端只与代理通信，避开国内对 CDN 的阻断
- **代价**：占用 Vercel 流量（Hobby 每月 100GB）；blob 常有几十 MB，
  需要用流式响应绕开 4.5MB 的响应体上限

通过环境变量切换：

| 变量 | 默认值 | 说明 |
| --- | --- | --- |
| `DOCKER_BLOB_MODE` | `redirect` | `redirect` 或 `proxy` |

## 缓存策略

manifest 缓存在进程内，键为 `kind | name | reference | Accept`：

- **必须带 Accept**：同一个 URL 在不同 Accept 下返回不同内容
  （OCI index / docker manifest list / 单架构 manifest），且 Docker Hub
  按客户端给出的**优先级顺序**协商类型，所以 Accept 归一化时只去空白、
  **不重排**。
- digest 引用（`sha256:...`）内容寻址，永久有效 → 缓存 24 小时
- tag 可能被重新指向 → 缓存 5 分钟

| 变量 | 默认值 | 说明 |
| --- | --- | --- |
| `DOCKER_MANIFEST_TTL_DIGEST_MS` | `86400000` | digest 引用的缓存时长 |
| `DOCKER_MANIFEST_TTL_TAG_MS` | `300000` | tag 引用的缓存时长 |
| `DOCKER_CACHE_MAX_SIZE` | `500` | 进程内缓存条目上限 |

blob 响应不缓存。

## 环境变量

| 变量 | 默认值 | 说明 |
| --- | --- | --- |
| `DOCKERHUB_USERNAME` | 空 | Docker Hub 用户名，配了凭据才生效 |
| `DOCKERHUB_TOKEN` | 空 | Personal access token（只读权限即可） |
| `DOCKER_REGISTRY_BASE` | `https://registry-1.docker.io` | 上游 registry |
| `DOCKER_AUTH_BASE` | `https://auth.docker.io` | 上游 auth 服务 |
| `DOCKER_BLOB_MODE` | `redirect` | blob 处理方式 |
| `DOCKER_TIMEOUT_MS` | `15000` | 上游请求超时 |
| `DOCKER_CACHE_MAX_SIZE` | `500` | 进程内缓存条目上限 |
| `DOCKER_MANIFEST_TTL_DIGEST_MS` | `86400000` | digest 缓存时长 |
| `DOCKER_MANIFEST_TTL_TAG_MS` | `300000` | tag 缓存时长 |

## 实现要点

**`/v2/` 返回 200 而非 401**：客户端看到 401 会去找 `WWW-Authenticate` 里的
realm 换令牌，而 realm 指向被阻断的 `auth.docker.io`。返回 200 让客户端
不再自行认证，令牌改由代理在服务端获取。

**尾斜杠**：`trailingSlash: false` 会把 `/v2/` 308 重定向到 `/v2`。实测
podman 能正确跟随该重定向（请求日志显示 `GET /v2/` → `GET /v2`），
不影响协议流程。

**`maxRedirects: 0`**：blob 的 307 必须自己处理，不能让 axios 跟随。

**不缓存 307 响应**：签名 URL 约 50 分钟后失效，且每次签名不同。

**流式转发不用 `pipe`**：`pipe` 要求目标实现完整的 EventEmitter 接口，
而 Vercel 的 `res` 与测试 mock 的接口并不一致，改为手动 `write` / `end`。
