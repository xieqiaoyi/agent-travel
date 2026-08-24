# Project 模块新手教程

## 这是什么
Project 模块识别当前目录的项目信息（git 仓库、worktree、项目 ID 等），并通过 Instance API 提供统一上下文。

## 为什么需要它
没有准确的项目边界就无法判断哪些文件属于任务，也无法安全创建 plan 或 patch。

## 关键概念
- **Instance**：一次 CLI 命令执行时维护的上下文，包含目录、worktree、Project.Info。
- **Sandbox**：opencode 创建的隔离目录，存放 plan 与临时文件。

## 主要文件速查
- **bootstrap.ts**：在实例化项目时执行的初始化流程（依赖、配置、worktree）。
- **instance.ts**：缓存目录→项目上下文，暴露 `Instance.provide` 等实用方法。
- **project.ts**：检测仓库信息、派生 `Project.Info`（id、工作区、配置路径等）。
- **state.ts**：基于目录作用域的状态管理器，支持自动创建/销毁。
- **vcs.ts**：封装版本控制（Git）信息获取（branch、root、status）。

## 原理速览
- Instance.provide 缓存目录→Project 的映射，避免重复初始化。
- Instance.dispose 在命令结束后释放状态，防止跨命令污染。

## 入门练习
- 在非 git 目录运行 CLI，观察 Project 如何 fallback 到简单模式。

## 常见坑与调试建议
- 沙盒默认在 `.opencode/sandboxes`，磁盘不足时可在配置中调整。
