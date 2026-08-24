# Format 模块新手教程

## 这是什么
Format 模块管理格式化器生命周期（启动/停止/状态查询），让 CLI 可以异步调用 prettier、ruff 等工具。

## 为什么需要它
自动格式化能提升提交质量，但不希望每次都手动执行，因此需要常驻 worker 响应格式化请求。

## 关键概念
- **Formatter Worker**：负责格式化任务的子进程或任务。

## 主要文件速查
- **formatter.ts**：描述具体格式化任务的执行方式与状态机。
- **index.ts**：调度格式化服务并暴露给 Server/Session 查询状态的接口。

## 原理速览
- formatter.ts 维护格式化器列表和配置。
- index.ts 对外提供 status/health API 供 server 查询。

## 入门练习
- 在 formatter.ts 中新增一个示例 formatter，查看 `/formatter` 路由输出。

## 常见坑与调试建议
- 格式化任务耗时较长，需在后台执行并汇报状态。
