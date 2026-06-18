# OpenResty + Tongsuo

基于 [1Panel OpenResty Dockerfile](https://github.com/1Panel-dev/appstore)，将 OpenSSL 替换为[铜锁 (Tongsuo)](https://github.com/Tongsuo-Project/Tongsuo)，提供国密 SM2/SM3/SM4、TLCP 双证书、NTLS 传输支持，同时保持国际 HTTPS 完全兼容。

版本标签格式: `{openresty_version}-{panel_revision}-{ubuntu_codename}-tongsuo`

## 概览

| 组件 | 版本 |
|------|------|
| OpenResty | 1.31.1.1 |
| Tongsuo | 8.5.0-pre1 (API-compatible with OpenSSL 3.5.4) |
| PCRE2 | 10.47 (JIT) |
| LuaJIT + LuaRocks | bundled |
| HTTP/2, HTTP/3, WebSocket | supported |

**国密能力**: SM2/SM3/SM4 算法、TLCP (GB/T 38636) 双证书、NTLS 国密传输、RFC 8998 TLS 1.3 单证书。

## 快速开始

```bash
# 拉取
docker pull ghcr.io/preca-hoshino/openresty-tongsuo:latest

# 运行
docker run -d -p 80:80 -p 443:443 ghcr.io/preca-hoshino/openresty-tongsuo:latest

# 本地构建
docker build -t openresty-tongsuo ./build
```

构建参数可通过 `--build-arg` 覆盖，参见 [Dockerfile](build/Dockerfile) 中的 `ARG` 声明。

## 验证

```bash
docker exec -it openresty-tongsuo bash

# 铜锁版本
/usr/local/openresty/openssl3/bin/openssl version
# Tongsuo 8.5.0-pre1 (OpenSSL 3.5.4)

# 国密算法
/usr/local/openresty/openssl3/bin/openssl ecparam -list_curves | grep SM2
echo "test" | /usr/local/openresty/openssl3/bin/openssl dgst -sm3

# OpenResty 编译信息
/usr/local/openresty/nginx/sbin/nginx -V 2>&1
```

## Nginx 国密配置

```nginx
server {
    listen 443 ssl;

    # 国际证书
    ssl_certificate     /etc/ssl/certs/ecc/server.crt;
    ssl_certificate_key /etc/ssl/certs/ecc/server.key;

    # 国密双证书 (铜锁扩展指令)
    ssl_sign_certificate     /etc/ssl/certs/sm2/server-sign.crt;
    ssl_sign_certificate_key /etc/ssl/certs/sm2/server-sign.key;
    ssl_enc_certificate      /etc/ssl/certs/sm2/server-enc.crt;
    ssl_enc_certificate_key  /etc/ssl/certs/sm2/server-enc.key;

    ssl_ciphers 'ECC-SM2-WITH-SM4-SM3:ECDHE-SM2-WITH-SM4-SM3:\
                 ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256';
    ssl_protocols TLSv1.2 TLSv1.3;
}
```

迁移细节见 [tongsuo-migration-guide.md](tongsuo-migration-guide.md)。

## CI/CD

GitHub Actions 四阶段流水线: 元数据 -> 构建(缓存) -> 推送 ghcr.io -> 拉取验证。

- **自动**: 推送 `main`/`master` 且 `build/` 或 workflow 有变更
- **手动**: Actions -> Run workflow
- **定时**: 每周一 06:00 UTC
- **PR**: 仅构建验证，不推送

推送阶段内置指数退避重试以应对 ghcr.io 偶发超时。

## 项目结构

```
build/
  Dockerfile
  nginx.conf
  nginx.vh.default.conf
  tmp/{pre,default}.sh
.github/workflows/docker-build.yml
tongsuo-migration-guide.md
```

## 许可证

[GPL-3.0](LICENSE) -- 基于 [1Panel OpenResty Dockerfile](https://github.com/1Panel-dev/appstore) 改造。

上游组件许可:

| 组件 | 许可证 |
|------|--------|
| [OpenResty](https://github.com/openresty/openresty) | BSD-2-Clause |
| [Tongsuo](https://github.com/Tongsuo-Project/Tongsuo) | Apache-2.0 |
| [nginx](https://nginx.org) | BSD-2-Clause |
| [PCRE2](https://github.com/PCRE2Project/pcre2) | BSD-3-Clause |
| [LuaJIT](https://luajit.org) | MIT |
| [1Panel Dockerfile](https://github.com/1Panel-dev/appstore) | GPL-3.0 |

## 参考

- [Tongsuo](https://github.com/Tongsuo-Project/Tongsuo) | [文档](https://www.tongsuo.net/docs)
- [RFC 8998 - TLS 1.3 + SM](https://datatracker.ietf.org/doc/html/rfc8998)
- [GB/T 38636-2020 - TLCP](https://std.samr.gov.cn/gb/search/gbDetailed?id=71D5A5E5F5C5B5C5E5F5C5B5C5E5F5C5)
