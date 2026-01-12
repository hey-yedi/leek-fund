# Leek Fund (韭菜盒子) 代码库指南

## 项目概览
Leek Fund (韭菜盒子) 是一个用于实时监控股票、基金和期货的 VS Code 扩展。它深度集成了 VS Code 的原生 UI 功能（树视图、状态栏、Webview）。

## 核心架构 (Architecture)
- **入口 (Entry Point)**: `src/extension.ts`
  - 负责插件激活 (`activate`)。
  - 注册所有 `TreeDataProvider` (Fund, Stock, Binance, Forex)。
  - 启动后台服务：`LoopTimer` (轮询)、`FlashNewsDaemon` (快讯)、`ProxyServer` (代理)。
  - **重要**: `globalState` 初始化发生在激活早期，务必先于其他服务调用。

- **数据层 (Data Layer)**:
  - **Service**: 位于 `src/service/` (底层 HTTP 请求) 和 `src/explorer/*Service.ts` (业务逻辑)。
    - **MVVM 模式**: Service 负责 fetching/parsing 数据，Provider (`src/explorer/*Provider.ts`) 负责将数据转换为 `LeekTreeItem` 供 VS Code 渲染。
  - **Proxy Server**: `src/webview/proxyService/`
    - **机制**: 启动一个本地 Node.js Server (默认端口 16100+)。
    - **目的**: 解决 Webview 中访问外部金融接口 (EastMoney, Sina) 的 CORS 问题。Webview 请求本地代理 -> 代理请求远程接口。

- **UI 组件**:
  - **Explorer**: 实现 VS Code `TreeDataProvider`。
  - **Webviews**:
    - 代码位于 `src/webview/`。
    - HTML 模板位于 `template/` (传统 HTML/JS) 和 `template-packages/leek-center` (React 项目)。
    - **管理**: `src/webview/ReusedWebviewPanel.ts` 实现了 Webview 面板的复用池，避免重复创建。
  - **Status Bar**: `src/statusbar/` 管理底部状态栏的循环滚动行情 (`StatusBar` 类) 和盈亏统计 (`ProfitStatusBar` 类)。

## 关键模式与开发规范 (Key Patterns)
- **全局状态 (Global State)**:
  - 文件: `src/globalState.ts`
  - 用途: 存储 `ExtensionContext`、`Telemetry` 实例、`deviceId`、以及运行时缓存 (如 `fundAmount`)。
  - **原则**: 优先从 `globalState` 获取环境上下文，减少参数透传。

- **配置管理 (Configuration)**:
  - 文件: `src/shared/leekConfig.ts` (`LeekFundConfig`)
  - **原则**: 统一使用 `LeekFundConfig` 读写配置。它封装了 `workspace.getConfiguration`，处理了复杂的数组去重和配置组逻辑。
  - **监听**: 配置变更通常通过 `src/shared/utils.ts` 中的 `events` (EventEmitter) 通知各组件刷新。

- **数据模型**:
  - 所有树节点继承自 `LeekTreeItem` (`src/shared/leekTreeItem.ts`)。
  - 必须包含 `info` 属性存储原始数据，以便在命令回调中获取上下文。

- **容错与稳定性**:
  - **API 不稳定性**: 上游 API (Sina/EastMoney) 经常变动或限流。
  - **处理策略**: 数据获取失败通过 `try-catch` 捕获，**决不允许** Crash 插件。必须返回“暂无数据”或空对象，保证 UI 依然可用。
  - **数据解析**: 大量使用正则解析非标准 JSON (如 JSONP)。修改正则需谨慎，必须覆盖多种返回格式。

- **AI Agent 集成**:
  - 入口: `src/utils/aiAnalysisPanel.ts` (展示分析结果面板)。
  - 配置: `src/webview/ai-config.ts` (管理 OpenAI API Key 等)。
  - 交互: 通过 Webview 展示 Markdown 格式的 AI 分析报告。

## 开发工作流 (Workflow)
- **环境**: 推荐 Node.js 18+ (与 `package.json` 依赖匹配)。
- **安装**: `npm install` (根目录) 和 `yarn` (部分子包)。
- **构建**:
  - `npm run watch`: 增量编译 TypeScript。
  - React Webview: 需在 `template-packages/leek-center` 下单独构建。
- **调试**: 使用 VS Code "Run Extension" Launch Config。
- **测试**: `npm test` (集成测试)。

## 常见问题排查 (Troubleshooting)
- **npx/undici 错误 ("Process exited with code 1")**:
  - **现象**: 运行构建或测试时出现 `undici/lib/web/fetch/util.js` 报错。
  - **原因**: Node.js 版本与 `undici` (npm 依赖的底层库) 不兼容，或 `npx` 缓存损坏。
  - **解决**:
    1. 确保使用 Node.js LTS (v18/v20)。
    2. 清理缓存: `rm -rf ~/.npm/_npx`。
    3. 重新安装依赖: `rm -rf node_modules && npm install`。
- **Webview 白屏**: 检查 Proxy Server 是否启动成功 (查看 Output Panel)，或端口是否被占用 (默认 16100)。

## 常用文件速查
- `src/extension.ts`: 插件主入口
- `src/explorer/fundService.ts`: 基金核心逻辑
- `src/shared/leekConfig.ts`: 配置中心
- `src/webview/proxyService/proxyService.ts`: 本地代理服务
- `template/`: Webview 静态资源
- `template-packages/leek-center`: 韭菜中心 (React App)
