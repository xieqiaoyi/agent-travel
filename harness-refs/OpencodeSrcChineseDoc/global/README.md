# Global 模块新手教程

## 这是什么
Global 模块存储本机级别的路径（home/state/config/log）和一次性状态，供其他模块随时读取。

## 为什么需要它
统一的路径寄存器避免各模块重复调用 `os.homedir()` 并手动拼目录。

## 关键概念
- **Path Registry**：集中记录常用目录位置并缓存结果。

## 主要文件速查
- **index.ts**：定义全局路径（home/state/config/data）与一次性的进程级状态。

## 原理速览
- Global.Path 在启动时根据平台计算并缓存。
- Server `/path` 路由直接返回 Global 中记录的信息。

## 入门练习
- 打印 `Global.Path.config`/`state`，确认定位是否与预期一致。

## 常见坑与调试建议
- 若用户移动 `.opencode` 目录，需要重新初始化 Global。
