# Auth 模块新手教程

## 这是什么
Auth 模块像内置密码管家，统一存放 Provider、MCP、插件所需的 API Key 与 OAuth token。

## 为什么需要它
集中管理凭证可以避免在代码里硬编码密钥，也方便在 CLI 中更换账号或刷新 token。

## 关键概念
- **API Key**：模型服务商颁发的密钥，用来识别调用者。
- **OAuth Token**：通过浏览器授权获取的访问令牌。

## 主要文件速查
- **index.ts**：封装 OAuth/API Key 等凭证的存取、更新与缓存逻辑。

## 原理速览
- Auth.set 会把凭证写入 `Global.Path.config` 并刷新 Provider 缓存。
- Auth.get 先读内存再读磁盘，减少重复 IO。
- 删除凭证时要同步通知相关 Provider/插件刷新状态。

## 入门练习
- 调用 Auth.set 写入 OpenAI key，然后尝试使用 openai 模型。
- 触发一次 OAuth 登录流程，观察 token 如何持久化。

## 常见坑与调试建议
- Windows 若把配置目录放在 OneDrive，注意同步冲突导致丢失。
- OAuth 回调依赖本地 HTTP 端口，防火墙拦截会导致授权失败。
