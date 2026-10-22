# MarginNote 插件开发规则（AI 必读）

## 总则

- 本目录是 MarginNote 插件开发项目，运行环境与前端 / Node 不同，不要按浏览器假设做实现。
- 保持谨慎：把每次输出当作“尝试”，先验证再扩展；优先小步改动、可回滚。

## 文档查阅与证据

- 仅在本次改动涉及不确定的 MarginNote 原生 API 或 JavaScriptCore 行为时查阅对应文档；纯文案、CSS、普通 WebView 逻辑和规则文本维护不触发原生文档加载。
- 先检查相关本地实现、`Frameworks` 头文件和现有测试；需要文档时使用可用的 mn-docs MCP，或查阅 https://mn-docs.museday.top 的相关章节。已核实且未变化的内容在同一任务中复用，无需固定先读两篇概览。
- MCP 缺失不阻塞已有证据支持的工作；仅在任务确实需要且其他证据不足时处理文档接入。
- 不猜测未知 API 的形状或副作用。查证后仍缺少决定性证据时，只暂停依赖它的部分，继续独立工作，并向用户提出一个具体问题。

## 运行时与能力差异（不要按前端思维）

- 插件运行在 JavaScriptCore 环境：没有浏览器的 `window`、`document`、`fetch`、`localStorage`、`setTimeout`、`setInterval` 等。
- 网络请求不要用 `fetch`：按文档使用系统导出的网络 API（如 NSURLConnection 相关）与回调，响应体常见为 `NSData`。
- 环境无 Base64 解码等常用工具；涉及 `NSData` 转文本或 JSON 时，严格按文档做，不要自行臆断可用 API。

## 持久化规则（强制）

- 少量开关、数字、字符串、面板状态用 `NSUserDefaults`，key 必须带插件级前缀。
- 结构化数据、列表、用户生成内容、导出预设用 `Application.sharedInstance().documentPath + "/<AddonId>/"` 下的 JSON 文件。
- 临时文件用 `Application.sharedInstance().tempPath + "/<AddonId>/"`，不要混进长期数据目录。
- JSON 文件读写优先使用 `NSJSONSerialization` 与 `NSData.writeToFileAtomically(path, true)`，写入后检查返回值。
- 写入 Native API 前过滤 `undefined`、`null`、函数、循环对象和不可桥接对象，避免 iPad 上 JSBridge nullability 崩溃。
- 需要修改 MarginNote 笔记数据库时，用 `UndoManager.undoGrouping(...)` 包裹，成功后调用 `Application.sharedInstance().refreshAfterDBChanged(topicid)`；不要把 MarginNote 正文数据另存一份当长期真源。

## 结构与加载规则（强制）

- `main.js` 只做入口与导入：只允许在 `main.js` 调用 `JSB.require(...)`，且只允许使用 `JSB.require(...)`，不得使用 `require` 或 `import`。`JSB.require(...)` 的引入进入全局作用域，作用于所有脚本。
- 不要在 `main.js` 里定义业务函数或方法；所有实现放到独立文件，再由 `main.js` 通过 `JSB.require(...)` 导入。
- 除 `main.js` 外，任何文件禁止调用 `JSB.require(...)`，避免重复或污染全局导入行为。
- 优先用 ES6 语法（除非与运行时不兼容）；保持文件职责单一，不要把 UI、数据、命令处理混在一起。

## 全局与入口对象要点

- 入口通常是 `JSB.newAddon=function(mainPath){...}`，并返回插件实例。
- `self` 仅在实例方法内可用，代表当前插件实例；不要在模块顶层假设 `self` 存在。
- 常用全局注入对象以文档为准：`JSB`、`Application`、`Database`、`Note`、`UndoManager` 等；不确定就先查文档再用。

## 需求澄清

- 先从上下文、现有入口、配置和代码推断需求。明确且可逆的局部任务直接执行。
- 仅当无法推断的信息会实质改变产物、范围或安全性时，先问一个关键问题；回答后简短复述并继续，不追加确认回合。
- 已授权按假设继续时说明假设即可；外部写入、发布、破坏性操作仍需相应授权。

## 调试与验证

- 日志统一用 `console.log`，不要用 `JSB.log`。
- 构建无语法校验功能；改动后至少做一次人工检查（重新阅读代码）。
