# 文档索引

| 文档 | 内容 |
| --- | --- |
| [部署与运维](deployment.md) | 部署步骤、环境变量、函数区域选择、WAF IP 白名单 |
| [代理实现](docker.md) | 协议细节、认证模型、blob 两种模式、限速与缓存策略 |

## 快速索引

**部署到 Vercel** → [deployment.md](deployment.md#部署)

**客户端怎么配** → [docker.md](docker.md#客户端配置)

**为什么客户端不用 docker login** → [docker.md](docker.md#认证模型)

**理解 Docker 限速** → [docker.md](docker.md#限速与配额)

**blob 走直连还是走代理** → [docker.md](docker.md#blob-的两种处理方式)

**搞清楚为什么是 iad1 区域** → [deployment.md](deployment.md#函数区域)

**接入 WAF 白名单** → [deployment.md](deployment.md#waf配置-ip-白名单)
