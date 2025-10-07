# PrivChat - 基于 Matrix Synapse 的私有聊天服务器

**二创作者**: shijian

## 1. 项目概述

PrivChat 是一个基于 `Matrix <https://matrix.org>`__ 开放标准实现的私有聊天服务器。它利用 `Element Synapse` 作为核心后端，为您提供一个安全、去中心化且可互操作的实时通信平台。您可以完全掌控自己的聊天数据，并选择与其他 Matrix 服务器进行联邦通信。

**主要特性**:
*   **私有托管**: 完全控制您的聊天服务器和数据。
*   **去中心化**: 支持与其他 Matrix 服务器联邦，实现跨服务器通信。
*   **端到端加密**: 确保您的对话内容安全。
*   **可扩展**: 通过丰富的 API 和机器人进行功能扩展。

## 2. 快速部署指南

本指南将帮助您快速部署 PrivChat 服务器，包括内网运行和通过 Cloudflare Tunnel 暴露到公网。

### 2.1. 前提条件

*   一台 Linux VPS (例如 Ubuntu)。
*   已安装 Python 3.8+ 和 Poetry。
*   已安装 `cloudflared` 客户端。
*   一个 Cloudflare 账户，用于管理您的公网域名。

### 2.2. 部署步骤

**1. 克隆仓库并初始化 Git**

如果您尚未克隆仓库，请先克隆。我们已经为您重新初始化了 Git 仓库并推送了代码。

**2. 生成配置文件**

使用您计划的服务器名称（例如 `privchat.shijian` 用于内网，或您的公网域名）生成 `homeserver.yaml`。

```bash
PrivChat/bin/python -m synapse.app.homeserver --server-name privchat.shijian --config-path homeserver.yaml --generate-config --report-stats=no
```

**3. 配置服务器**

编辑 `homeserver.yaml` 文件，确保以下配置：

*   **允许注册**:
    ```yaml
    enable_registration: true
    enable_registration_without_verification: true # 仅在信任环境中启用，否则请配置验证方式
    ```
*   **监听地址**: 确保 `bind_addresses` 包含 `127.0.0.1`。
    ```yaml
    listeners:
      - port: 8008
        tls: false
        type: http
        x_forwarded: true
        bind_addresses: ['::1', '127.0.0.1']
        resources:
          - names: [client, federation]
            compress: false
    ```
*   **抑制密钥服务器警告**:
    ```yaml
    trusted_key_servers:
      - server_name: "matrix.org"
    suppress_key_server_warning: true
    ```

**4. 安装依赖**

在项目根目录，使用 Poetry 安装所有 Python 依赖：

```bash
PrivChat/bin/pip install poetry
PrivChat/bin/poetry install --no-root
```

**5. 启动 Synapse 服务器**

使用 `synctl` 在后台启动服务器：

```bash
PrivChat/bin/synctl start
```

**6. 创建管理员用户**

在服务器运行后，打开一个新终端，创建您的第一个管理员用户：

```bash
PrivChat/bin/register_new_matrix_user -c homeserver.yaml http://localhost:8008
```
按照提示输入用户名和密码，并选择 `yes` 设为管理员。

### 2.3. Cloudflare Tunnel 公网部署

**1. 修复 VPS DNS (如果遇到连接问题)**

如果 `cloudflared` 无法连接 Cloudflare，请临时修复 VPS 的 DNS 解析：

```bash
echo "nameserver 1.1.1.1" | sudo tee /etc/resolv.conf
```

**2. 获取 Cloudflare Tunnel Token**

*   登录 [Cloudflare Zero Trust 控制台](https://one.dash.cloudflare.com/)。
*   导航到 `Access` -> `Tunnels`。
*   创建一个新隧道，获取其运行命令中的 `--token` 字符串。

**3. 运行 Cloudflare Tunnel**

使用您获取的令牌和您的公网域名（例如 `privchat.shijian.qzz.io`）启动隧道：

```bash
cloudflared tunnel --no-autoupdate run --token YOUR_CLOUDFLARED_TOKEN --url http://localhost:8008
```
**请保持此终端运行，以维持隧道连接。**

**4. 配置 Cloudflare DNS (手动)**

由于自动 DNS 路由可能失败，请在 Cloudflare DNS 设置中手动添加 CNAME 记录：

*   **Type**: `CNAME`
*   **Name**: `privchat.shijian` (您的子域名部分)
*   **Target**: `YOUR_TUNNEL_UUID.cfargotunnel.com` (请替换为您的隧道 UUID，例如 `b08a27e2-e1e9-4935-bb3e-d54cc4203eec.cfargotunnel.com`)
*   **Proxy status**: 确保是橙色云朵 (Proxied)。

### 2.4. 测试公网访问

完成 DNS 配置后，等待几分钟，然后通过 Element Web 客户端访问：

*   Homeserver URL: `https://privchat.shijian.qzz.io`
*   使用您创建的管理员用户登录。

## 3. 安全注意事项

Matrix 在某些 API 中提供原始的用户提供数据，特别是内容仓库端点。虽然我们尽力缓解 XSS 攻击，但 Matrix homeserver 不应托管在托管其他 Web 应用程序的域上。理想情况下，homeserver 应该位于一个完全不同的注册域（eTLD+1）。

## 4. 贡献与开发

我们欢迎社区对 PrivChat (Synapse) 的贡献！请参考原始 Synapse 项目的贡献指南。

## 5. 许可证

本项目遵循 GNU Affero General Public License (AGPLv3) 协议。
