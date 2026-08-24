# IDE 集成模块新手教程

## 这是什么
IDE 模块为 VS Code、Zed 等编辑器提供桥接函数，负责启动 CLI、转发命令与日志，是插件与 backend 的通道。

## 为什么需要它
统一入口可以让不同 IDE 共用同一 backend，不必重复实现业务逻辑。

## 关键概念
- **Language Client**：IDE 端 LSP 客户端，负责与 CLI 启动的 LSP server 通讯。

## 主要文件速查
- **index.ts**：IDE 集成入口，暴露语言服务、命令和调试钩子给 VS Code/Zed 等客户端。

## 原理速览
- IDE 模块暴露 TypeScript API，供 VS Code/Zed 扩展直接调用。
- 它与 LSP、Server、PTY 协作，把日志与命令结果回传给 IDE。

## 入门练习
- 阅读 `sdks/vscode`，了解 IDE 模块如何被导入使用。

## 常见坑与调试建议
- IDE 插件生命周期与 CLI 不同，要记得释放资源避免僵尸进程。
