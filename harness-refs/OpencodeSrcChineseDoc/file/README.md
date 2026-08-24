# File 模块新手教程

## 这是什么
File 模块统一处理文件系统操作：忽略规则、watcher、时间戳与 ripgrep 搜索，
让所有工具在访问磁盘时都有一致的安全策略。

## 为什么需要它
直接操作文件容易越界或遗漏忽略列表，集中到 File 模块可以确保路径检查、编码与日志一致。

## 关键概念
- **Glob**：带 *、? 的通配符路径匹配语法。
- **Ignore 规则**：类似 `.gitignore` 的排除列表。

## 主要文件速查
- **ignore.ts**：解析 `.gitignore`/`.opencodeignore` 等规则并判定文件过滤。
- **index.ts**：聚合文件读写、路径解析和安全检查的统一入口。
- **ripgrep.ts**：封装 ripgrep 搜索调用与结果解析。
- **time.ts**：跟踪文件时间戳/版本号以支持缓存与增量操作。
- **watcher.ts**：实现文件系统监听，供 TUI、计划模式等实时刷新使用。

## 原理速览
- ignore.ts 合并 gitignore 与用户自定义规则。
- watcher.ts 以增量方式监听文件变化供 TUI/plan 使用。
- ripgrep.ts 封装 rg 调用并处理路径与编码差异。

## 入门练习
- 运行 glob/grep 工具体验忽略规则如何生效。
- 修改 ignore 配置后观察 watcher 的日志。

## 常见坑与调试建议
- Windows 路径分隔符为 `\`，记得在 glob 模式中转换。
- 监听超大仓库可能耗尽文件句柄，需要适当过滤。
