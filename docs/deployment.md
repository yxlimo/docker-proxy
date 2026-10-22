# 部署与运维

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

### 生成 Docker Hub token

Docker Hub → Settings → Personal access tokens → Generate new token，
Scopes 只勾 **Read**。token 只显示一次。

## 函数区域

`vercel.json` 里 `regions` 设为 `iad1`（美东）。

这个选择与「离用户近」的直觉相反，原因是上游的 CloudFront 边缘落点：

| region | 解析到的 CloudFront 节点 | 到上游延迟 | 冷 miss 中位延迟 |
| --- | --- | --- | --- |
| `iad1` | `IAD61-P9`（阿什本） | 6–12ms | **0.525s** |
| `hkg1` | `TLV55-P3`（特拉维夫） | ~231ms | 0.721s |

30 样本实测：iad1 全部样本 < 0.7s，hkg1 有 5 个样本 > 0.7s。
PoP 落点由 BGP 决定（国内一律 sin1），改 region 不影响客户端到 PoP 的路径，
只影响函数到上游 CloudFront 的边缘选择。

## WAF 配置：IP 白名单

Vercel 的 Bot Protection 会给非浏览器客户端下发 JS 挑战，Docker / podman
解不开挑战会直接拿到 403。**不要开 Bot Protection**，
改用 custom rule 配 `bypass` 动作放行自己的来源。

在项目 → Firewall 里加一条 custom rule：

- **Action**：`bypass`
- **Condition**：`ip_address` 等于你的出口 IP

家宽是动态 IP，IP 变了规则就失效。可以按 `AS Number`（运营商）匹配，
换 IP 不影响；也可以用脚本自动同步 IP。

注意 Vercel 的两类 bypass 作用范围不同：

| 类型 | 豁免范围 | 生效 |
| --- | --- | --- |
| system bypass | 仅系统级缓解措施（DDoS） | 立即 |
| conditional bypass（custom rule） | Bot Protection + 自己的 challenge 规则 | 立即（custom rule 需 publish） |

Bot Protection 属于后者，要用 custom rule 配 `bypass`，system bypass 不生效。

## 客户端配置

podman 见 [docker.md](docker.md#客户端配置)，Docker daemon 同页有对应说明。

## 运维要点

- **签名 URL 约 50 分钟过期**：blob 的 307 响应不能缓存，
  代码里已设 `Cache-Control: no-store`。
- **上游 blob CDN 有约 43% 的概率性 TLS 阻断**：`redirect` 模式下由客户端
  重试消化（实测 6/6 拉取成功）。若体感不佳，切 `DOCKER_BLOB_MODE=proxy`
  换成全走代理，代价是占用 Vercel 流量。
- **manifest 缓存是省配额的关键**：只有 manifest GET 消耗限速配额，
  命中进程内缓存就不回源。
- **进程内缓存每实例一份**：冷启动即失效，但冷启动只回源一次，不影响正确性。
