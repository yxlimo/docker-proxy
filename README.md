## docker-proxy

用 Vercel 代理 Docker Hub 镜像仓库，让国内客户端不再直连被 TLS 层阻断的
`registry-1.docker.io`。客户端无需 `docker login`。

## 状态：无限期搁置

代码已就绪（测试通过，已推送 GitHub），但**从未部署**——没有 Vercel 项目、
没有域名、没有配置凭据。搁置期间不做任何推进。

恢复时需要：

```bash
cd vercel-projects/docker-proxy
vercel --yes --scope correct1 --prod
vercel git connect --scope correct1 --yes
```

再配环境变量 `DOCKERHUB_USERNAME` / `DOCKERHUB_TOKEN`（只读权限的 PAT 即可）、
绑定自定义域名、设 WAF 访问控制。

部署步骤见下方「部署」章节，实现细节见 `docs/`。

## 功能

| 能力 | 说明 |
| --- | --- |
| Registry v2 代理 | `/v2/...` 实现 Docker Registry v2 协议，manifest / blob / tags 全支持 |
| 服务端认证 | 代理自行换取令牌，客户端无需 `docker login`；拉取配额按账号计而非按出口 IP 计 |
| manifest 缓存 | 进程内缓存，键含 Accept；digest 引用缓存 24 小时，tag 缓存 5 分钟 |
| blob 双模式 | 默认透传 307 让客户端直连 CDN（省流量）；可切换为代理流式转发（更可靠） |
| 错误原样透传 | 上游状态码与响应体不包装，Docker 客户端据此区分 401 / 404 / 429 |
| 限速头透传 | `ratelimit-*` 响应头原样返回，客户端可据此提示剩余配额 |
| 健康检查 | `/health` 返回上游地址、是否已配置凭据、缓存条目数 |

## 部署

1. fork 本仓库，登录 [Vercel](https://vercel.com/) 选择自己的仓库新建项目。

2. 绑定自己的域名。**必须绑定**——`*.vercel.app` 在国内基本不可用
   （实测被 DNS 污染 + TLS SNI 层面 RST 双重阻断）。

3. 配置服务端凭据（Settings → Environment Variables）

   | 变量 | 说明 |
   | --- | --- |
   | `DOCKERHUB_USERNAME` | Docker Hub 用户名 |
   | `DOCKERHUB_TOKEN` | Personal access token（只读权限即可） |

   凭据只用于向 `auth.docker.io` 做 Basic 认证换令牌，不进仓库、不进代码。
   配好后客户端无需 `docker login`，且拉取配额按账号计（200 次/6 小时）
   而非按出口 IP 计（100 次/6 小时，共享 IP 的配额会被其他租户消耗）。

4. 配置访问控制。**代理不做鉴权**，公开暴露前请在 Vercel WAF 里加 IP 白名单
   ——配了 Docker Hub 凭据后，代理也能访问该账号的私有仓库。

推送到 `master` 会触发 Vercel 重新部署。

### 全部环境变量

| 变量 | 默认值 | 说明 |
| --- | --- | --- |
| `DOCKERHUB_USERNAME` | 空 | Docker Hub 用户名 |
| `DOCKERHUB_TOKEN` | 空 | Personal access token（只读权限即可） |
| `DOCKER_REGISTRY_BASE` | `https://registry-1.docker.io` | 上游 registry |
| `DOCKER_AUTH_BASE` | `https://auth.docker.io` | 上游 auth 服务 |
| `DOCKER_BLOB_MODE` | `redirect` | blob 处理方式：`redirect` 交客户端直连 CDN，`proxy` 由代理流式转发 |
| `DOCKER_TIMEOUT_MS` | `15000` | 上游请求超时 |
| `DOCKER_CACHE_MAX_SIZE` | `500` | 进程内 manifest 缓存条目上限 |
| `DOCKER_MANIFEST_TTL_DIGEST_MS` | `86400000` | digest 引用缓存时长（24 小时） |
| `DOCKER_MANIFEST_TTL_TAG_MS` | `300000` | tag 引用缓存时长（5 分钟） |

## 使用方法

把代理地址配成镜像源。

podman —— `/etc/containers/registries.conf`：

```toml
unqualified-search-registries = ["docker.io"]

[[registry]]
prefix = "docker.io"
location = "docker.io"

[[registry.mirror]]
location = "<你的代理域名>"
insecure = false
```

Docker daemon —— `/etc/docker/daemon.json`：

```json
{
  "registry-mirrors": ["https://<你的代理域名>"]
}
```

改完重启 daemon：`systemctl restart docker`。

然后照常拉取：

```bash
podman pull docker.io/library/alpine:3.19
```

健康检查：

```bash
curl "https://<你的代理域名>/health"
```

## 注意事项

- 代理只做转发，不做鉴权与限流。公开暴露前务必在 Vercel WAF 里加访问控制。
- Bot Protection 会对非浏览器客户端下发 JS 挑战，客户端解不开会拿到 403。
  用 Firewall 里的 custom rule 配 `bypass` 动作放行自己的来源。
- blob 的 307 跳转指向签名 URL，约 50 分钟过期，**该响应不能被缓存**。
- 上游 blob CDN 在国内有约 43% 的概率性 TLS 阻断，`redirect` 模式下
  由客户端重试消化；需要更稳可设 `DOCKER_BLOB_MODE=proxy`。
- 免费额度每月 100GB 流量，`proxy` 模式下 blob 全部走代理。

## 本地开发

```bash
pnpm install
pnpm test
```

测试会在本地起一个假上游，覆盖路径解析、Accept 协商、manifest 缓存、
令牌缓存与 401 重试、blob 两种模式、限速头透传等场景。
