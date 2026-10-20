# 逆向一切

**Reverse Engineer Anything** — 基于 [REA](https://github.com/morluto/rea)（MIT License）构建的软件逆向研究工具箱。

看到任何软件里的功能，搞懂它怎么实现的——从网站前端到二进制产物，全程本地分析，每个结论带证据链。

## 目录结构

```
├── scenarios/     # 可复用的 REA 场景模板（JSON）
├── reports/       # 侦察报告
└── evidence/      # 原始证据文件（REA Evidence JSON）
```

## 收录的研究

### 1. lightchainai.com 网站侦察

对一个 Next.js 商业站点的完整被动侦察，产出：

- **技术栈判定**：Next.js App Router、路由组 `(pages)` + 动态段 `[local]`、RSC + Link 预取、地区重定向
- **后端 API 地图**：登录体系、STS 临时凭证（前端直传 OSS 架构）、埋点批量上报
- **第三方服务清单**：CRM 营销集成、GA4/GTM、百度统计、Clarity 等
- **站点自身 bug**：埋点上报失败的退避重试逻辑
- **证据**：335 个事件零丢弃，480KB 带溯源的 Evidence JSON

### 2. REA 自我逆向

用 REA 的 JS 静态分析引擎拆解 REA 自己的 npm 发布包：

- 6,857 个文件、113MB 代码、1,074 万个 AST 节点，6 分钟完成
- 从产物侧证实分层架构：application → provider adapters → domain → CLI/MCP
- 安全结论：零硬编码外部端点（纯本地工具）、自身非 Electron 应用
- 555 个 source map——可继续还原至接近源码形态

## 使用方法

本仓库的方法论可复现。环境要求：Node.js ≥ 22.19。

```bash
# 安装 REA
npm install -g rea-agents

# 网站被动侦察（用本仓库的场景模板）
rea capture-browser-scenario scenarios/lightchain-home-recon.json --format json

# JS 应用静态分析（针对 Electron/打包应用/JS bundle）
rea analyze-javascript-application /path/to/target --format json
```

场景模板按目标站点修改 `start_url` 和 `actions` 即可复用。完整命令列表：`rea --help`（155 个 CLI 命令 / 207 个 MCP 工具）。

## 方法论要点

1. **被动优先**：无头浏览器 + 确定性环境配置（固定 viewport/locale/时区、禁用 service worker），只观察不注入
2. **证据链**：每个结论附出处，区分「观察到的」与「推测的」，限制条件显式声明
3. **诚实边界**：未捕获响应体就不猜 API 结构；请求未完成就标注未知
4. **本地分析**：目标文件不出本机

## 致谢

- [REA](https://github.com/morluto/rea) by morluto — MIT License，本仓库全部研究基于该工具完成
- 侦察目标仅用于技术学习与竞品分析

## License

本仓库的研究报告与场景文件以 MIT License 发布。
