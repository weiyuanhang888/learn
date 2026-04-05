# SSL 证书验证失败问题解决记录

## 问题描述

在使用 Claude Code 时遇到以下错误：

```
Unable to connect to API: SSL certificate verification failed. Check your proxy or corporate SSL certificates 

Retrying in 14 seconds… (attempt 9/10) 

· Meandering… (2m 11s) 

⎿  Tip: Use /btw to ask a quick side question without interrupting Claude's current work
```

## 解决方案

通过修改 shell 配置文件，永久设置环境变量来绕过 SSL 证书验证：

### 操作步骤

1. **编辑 .zshrc 配置文件**：
   ```bash
   echo 'export NODE_TLS_REJECT_UNAUTHORIZED=0' >> ~/.zshrc
   ```

2. **重新加载配置**：
   ```bash
   source ~/.zshrc
   ```

3. **验证环境变量**：
   ```bash
   echo $NODE_TLS_REJECT_UNAUTHORIZED
   # 输出：0
   ```

4. **运行 Claude Code**：
   ```bash
   claude
   ```

## 问题原因分析

### 1. SSL 证书验证失败的根本原因

- **VPN 影响**：使用 VPN 时，VPN 服务会拦截网络流量并使用自己的 SSL 证书进行重新加密
- **代理服务器**：公司网络中的代理服务器可能会使用自定义的 SSL 证书
- **证书链不完整**：本地系统缺少必要的根证书，无法验证服务器证书的有效性
- **TLS 拦截**：企业网络中的 TLS 拦截防火墙会修改 SSL 证书

### 2. 环境变量的作用

- `NODE_TLS_REJECT_UNAUTHORIZED` 是 Node.js 的环境变量
- 设置为 `0` 时，会禁用 Node.js 的 SSL 证书验证
- 这样 Claude Code（基于 Node.js）就不会因为证书验证失败而拒绝连接

## 技术原理

1. **SSL/TLS 握手过程**：
   - 客户端向服务器发送 SSL/TLS 握手请求
   - 服务器返回证书链（包括服务器证书和中间证书）
   - 客户端验证证书的有效性（检查证书链、有效期、签名等）
   - 验证通过后，建立加密连接

2. **证书验证失败的情况**：
   - 证书不是由受信任的证书颁发机构签发
   - 证书链不完整
   - 证书已过期
   - 证书域名与实际域名不匹配

3. **环境变量的工作原理**：
   - `NODE_TLS_REJECT_UNAUTHORIZED=0` 会告诉 Node.js 忽略证书验证错误
   - 这会跳过证书链验证步骤，直接建立连接

## 优缺点分析

### 优点
- **简单有效**：只需修改一次配置，永久解决问题
- **无需每次手动设置**：设置后每次运行 Claude Code 都会自动应用
- **适用于所有 Node.js 应用**：不仅限于 Claude Code，其他 Node.js 应用也会受益

### 缺点
- **安全风险**：禁用 SSL 证书验证会降低通信安全性，可能受到中间人攻击
- **全局影响**：设置为全局环境变量后，所有 Node.js 应用都会禁用证书验证
- **掩盖根本问题**：没有解决证书链不完整的根本问题

## 替代解决方案

### 1. 临时解决方案（推荐）

仅在需要时临时设置环境变量：

```bash
NODE_TLS_REJECT_UNAUTHORIZED=0 claude
```

### 2. 创建专用脚本

创建一个专门用于运行 Claude Code 的脚本：

```bash
#!/bin/bash
# 仅为 Claude Code 禁用 SSL 验证
export NODE_TLS_REJECT_UNAUTHORIZED=0
claude
```

### 3. 安装根证书

在企业环境中，联系 IT 部门获取并安装企业根证书：

```bash
# 假设根证书文件为 company-ca.crt
sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain company-ca.crt
```

### 4. 配置 VPN 绕过

在 VPN 设置中添加 `api.anthropic.com` 到绕过列表，避免 VPN 拦截该域名的流量。

## 安全建议

1. **仅在必要时使用**：禁用 SSL 验证只应在无法解决证书问题的情况下使用
2. **使用专用脚本**：优先使用专用脚本而不是全局环境变量
3. **定期检查**：定期检查网络环境，看是否可以恢复 SSL 验证
4. **网络安全**：在公共网络或不安全网络中，建议恢复 SSL 验证

## 验证方法

### 验证环境变量设置

```bash
# 查看环境变量
echo $NODE_TLS_REJECT_UNAUTHORIZED
# 输出：0

# 验证 Claude Code 连接
claude
# 应该能够成功连接，不再出现 SSL 证书验证错误
```

### 测试其他 Node.js 应用

```bash
# 测试其他 Node.js 应用是否受影响
node -e "https.get('https://api.anthropic.com', (res) => console.log('Connected successfully'))"
```

## 总结

通过在 `.zshrc` 文件中添加 `export NODE_TLS_REJECT_UNAUTHORIZED=0` 环境变量，可以永久解决 Claude Code 的 SSL 证书验证失败问题。虽然这种方法存在一定的安全风险，但在特定网络环境下（如使用 VPN 或企业代理）是必要的解决方案。

建议在网络环境允许的情况下，优先考虑安装根证书或配置 VPN 绕过等更安全的解决方案，以确保通信安全。
