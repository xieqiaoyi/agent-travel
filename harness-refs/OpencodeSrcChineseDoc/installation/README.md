# Installation 模块新手教程

## 这是什么
Installation 模块记录 CLI 的版本、安装路径，并处理 postinstall 动作（复制平台特定二进制）。

## 为什么需要它
当 CLI 遇到兼容问题或需要提示升级时必须知道自身版本；集中管理也方便脚本读取。

## 关键概念
- **Postinstall**：npm 安装完成后自动执行的脚本，用于额外准备工作。

## 主要文件速查
- **index.ts**：记录 CLI 版本与安装状态，支持 postinstall、自更新等功能。

## 原理速览
- Installation.VERSION 与 package.json 保持同步，CLI 入口会引用它。
- postinstall 根据平台判断是否需要下载本地二进制。

## 入门练习
- 修改 Installation.VERSION 并运行 `opencode --version` 验证输出。

## 常见坑与调试建议
- 别忘了同步更新 npm 包版本，避免用户看到的版本与实际不符。
