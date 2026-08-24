# Flag 模块新手教程

## 这是什么
Flag 模块集中管理运行时开关（如 auto-share、client 类型），可以通过环境变量或 CLI 参数快速切换行为。

## 为什么需要它
Feature Flag 让我们无需改代码即可打开/关闭实验功能，也方便不同部署环境使用不同策略。

## 关键概念
- **Feature Flag**：动态控制某个功能是否启用的布尔/枚举配置。

## 主要文件速查
- **flag.ts**：集中读取 CLI 参数和环境开关，映射为运行时 Feature Flags。

## 原理速览
- Flag 模块读取环境变量与 CLI 参数得出最终值。
- ToolRegistry、Server、Session 会根据 flag 决定是否启用特定能力。

## 入门练习
- 设置 `OPENCODE_AUTO_SHARE=1`，运行 CLI 观察新会话是否自动分享。

## 常见坑与调试建议
- flag 名称全部大写，避免与普通配置混淆。
