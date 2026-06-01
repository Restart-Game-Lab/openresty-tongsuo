# OpenResty + Tongsuo (铜锁) Docker 镜像

> 🔐 **国密全栈 OpenResty**: SM2/SM3/SM4 + TLCP/NTLS + 国际 HTTPS 100% 兼容

基于 [1Panel OpenResty Dockerfile](https://github.com/1Panel-dev/appstore)，将 OpenSSL 3.5.6 替换为[铜锁 (Tongsuo) 8.5.0-pre1](https://github.com/Tongsuo-Project/Tongsuo)，实现国密算法全栈支持。

> 版本命名对齐 1Panel appstore：`{openresty_version}-{panel_revision}-{ubuntu_codename}-tongsuo`
> 例：`1.29.2.5-0-noble-tongsuo`

## 特性

| 特性 | 状态 |
|------|------|
| OpenResty 1.29.2.5 | ✅ |
| Tongsuo 8.5.0-pre1 (基于 OpenSSL 3.5.4) | ✅ |
| SM2/SM3/SM4 国密算法 | ✅ |
| TLCP (GB/T 38636) 双证书 | ✅ |
| NTLS 国密传输 | ✅ |
| RFC 8998 TLS 1.3 单证书 | ✅ |
| 国际 ECC/RSA + AES-GCM | ✅ |
| PCRE2 JIT | ✅ |
| LuaJIT + LuaRocks | ✅ |
| HTTP/2, HTTP/3, WebSocket | ✅ |

## 快速开始

### 从 GitHub Container Registry 拉取

```bash
docker pull ghcr.io/preca-hoshino/openresty-tongsuo:latest
```

### 运行

```bash
docker run -d -p 80:80 -p 443:443 \
  --name openresty-tongsuo \
  ghcr.io/preca-hoshino/openresty-tongsuo:latest
```

### 本地构建

```bash
docker build -t openresty-tongsuo ./build
```

自定义版本:

```bash
docker build \
  --build-arg RESTY_OPENSSL_VERSION=8.5.0-pre1 \
  --build-arg RESTY_VERSION=1.29.2.5 \
  --build-arg RESTY_J=$(nproc) \
  -t openresty-tongsuo \
  ./build
```

## 验证

```bash
# 1. 进入容器
docker exec -it openresty-tongsuo bash

# 2. 查看铜锁版本
/usr/local/openresty/openssl3/bin/openssl version
# → "Tongsuo 8.5.0-pre1 (OpenSSL 3.5.4)"

# 3. 验证国密算法
/usr/local/openresty/openssl3/bin/openssl ecparam -list_curves | grep SM2
echo "test" | /usr/local/openresty/openssl3/bin/openssl dgst -sm3

# 4. 查看 OpenResty 编译信息
/usr/local/openresty/nginx/sbin/nginx -V 2>&1
```

## Nginx 国密配置示例

```nginx
server {
    listen 443 ssl;

    # 国际证书 (标准 HTTPS)
    ssl_certificate     /etc/ssl/certs/ecc/server.crt;
    ssl_certificate_key /etc/ssl/certs/ecc/server.key;

    # 国密双证书 (铜锁特有)
    ssl_sign_certificate     /etc/ssl/certs/sm2/server-sign.crt;
    ssl_sign_certificate_key /etc/ssl/certs/sm2/server-sign.key;
    ssl_enc_certificate      /etc/ssl/certs/sm2/server-enc.crt;
    ssl_enc_certificate_key  /etc/ssl/certs/sm2/server-enc.key;

    # 混合加密套件
    ssl_ciphers 'ECC-SM2-WITH-SM4-SM3:ECDHE-SM2-WITH-SM4-SM3:\
                 ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256';
    ssl_protocols TLSv1.2 TLSv1.3;
}
```

## CI/CD

本仓库使用 GitHub Actions 自动构建和推送镜像到 GitHub Container Registry，采用 **4 阶段流水线**：

| 阶段 | Job 名称 | 说明 |
|------|----------|------|
| 1 | `📋 准备 — 元数据` | 生成镜像标签、labels、版本号 |
| 2 | `🔨 构建 — 编译镜像` | 编译 Docker 镜像（缓存到 GitHub Actions Cache） |
| 3 | `📤 推送 — ghcr.io` | 登录 ghcr.io（自动重试），从缓存秒级重建并推送 |
| 4 | `✅ 验证 — 拉取测试` | 从 ghcr.io 拉取镜像，验证 SM2/SM3/SM4/OpenResty |

**触发条件：**

- **自动触发**: 推送到 `main`/`master` 分支（仅 `build/` 和 workflow 变更时）
- **手动触发**: Actions → "🐳 Build & Push OpenResty + Tongsuo" → Run workflow
- **定时构建**: 每周一 6:00 UTC（保持基础镜像更新）
- **PR 检查**: PR 只运行准备+构建（验证 Dockerfile 可编译，不推送）

**ghcr.io 登录重试：** 推送和验证阶段内置 5 次自动重试+指数退避，解决偶发性 `Client.Timeout` 问题。

## 项目结构

```
.
├── .github/workflows/
│   └── docker-build.yml       # CI/CD 工作流
├── build/
│   ├── Dockerfile             # 镜像构建文件
│   ├── nginx.conf             # OpenResty 主配置
│   ├── nginx.vh.default.conf  # 默认虚拟主机
│   └── tmp/
│       ├── pre.sh             # 构建前脚本
│       └── default.sh         # 构建后脚本
├── tongsuo-migration-guide.md # 迁移指南
└── README.md
```

## 与传统 OpenSSL 版的区别

| 组件 | 原版 | 铜锁版 |
|------|------|--------|
| 密码库 | OpenSSL 3.5.6 | Tongsuo 8.5.0-pre1 |
| 国密支持 | ❌ | ✅ SM2/SM3/SM4 |
| async session lookup 补丁 | 需要 patch | 内置 |
| Camellia/SEED/RC5/MD2 | 可选 | 已删除 |
| FIPS | OpenSSL FIPS 140 | GM/T 0028 国密局认证 |

## 许可证

本项目基于 [1Panel OpenResty Dockerfile](https://github.com/1Panel-dev/appstore)（GPL-3.0）改造，采用 [GNU General Public License v3.0](LICENSE)。

本项目打包的上游组件各自遵循原始许可协议：

| 组件 | 许可证 |
|------|--------|
| [OpenResty](https://github.com/openresty/openresty) | BSD-2-Clause |
| [Tongsuo (铜锁)](https://github.com/Tongsuo-Project/Tongsuo) | Apache-2.0 |
| [nginx](https://nginx.org) | BSD-2-Clause |
| [PCRE2](https://github.com/PCRE2Project/pcre2) | BSD-3-Clause |
| [LuaJIT](https://luajit.org) | MIT |
| [1Panel appstore Dockerfile](https://github.com/1Panel-dev/appstore) | GPL-3.0 |

## 参考

- [铜锁 GitHub](https://github.com/Tongsuo-Project/Tongsuo)
- [铜锁文档](https://www.tongsuo.net/docs)
- [RFC 8998 - TLS 1.3 + 国密](https://datatracker.ietf.org/doc/html/rfc8998)
- [GB/T 38636-2020 - TLCP 协议](https://std.samr.gov.cn/gb/search/gbDetailed?id=71D5A5E5F5C5B5C5E5F5C5B5C5E5F5C5)
- [1Panel OpenResty Dockerfile](https://github.com/1Panel-dev/appstore/tree/dev/apps/openresty)
