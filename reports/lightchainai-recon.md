# lightchainai.com 前端逆向侦察报告

> 工具：REA 6.7.0（rea-agents）· `capture-browser-scenario`
> 证据：`capture-lightchain.json`（335 个事件，0 丢弃，evidence_id: `ev_fc4a1208...`）
> 方式：无头 Chromium 无痕配置（1280×720 基准、blocked service workers、reduced motion），纯被动观察，未做任何写操作

## 1. 技术栈判定（证据充分）

| 维度 | 结论 | 证据 |
|---|---|---|
| 框架 | **Next.js App Router** | `_next/static/chunks/` 产物、`app/[local]/(pages)/layout-*.js` 路由 chunk、`main-app-*.js` |
| 路由组织 | 路由组 `(pages)` + 动态段 `[local]`（按语言/地区本地化） | chunk 路径 `app/%5Blocal%5D/(pages)/page-*.js` |
| 渲染模式 | RSC（React Server Components）+ Link 预取 | 请求带 `?_rsc=i3ce3` 参数；社区/学院/关于/定价四个路由被自动预取 |
| 入口行为 | 根路径 302 → `/global`（按地区重定向） | step 0 before/after URL 对比 |
| 静态资源 | 阿里云 OSS（杭州）+ `static-cn.linkaigc.com` CDN | 网络请求主机分布 |

## 2. 后端 API 地图（XHR/Fetch 共 20 条，去重后）

| 端点 | 方法 | 用途推断 |
|---|---|---|
| `/api/light-chain-guanwang/guanwang/auth/get-login-ticket` | GET | 登录票据获取 |
| `/api/light-chain-guanwang/guanwang/auth/login-with-secret-form` | POST | 账号密码登录 |
| `/api/light-chain-yunxiang-saas-server/aliyun/sts/oss/tokens?internal=0` | GET | **STS 临时凭证**——前端直传 OSS 架构 |
| `/api/light-chain-tracking/track/batch` | POST | 埋点批量上报（有失败重试逻辑） |

API 命名规律：`/api/{业务线}/{模块}/{功能}`，业务线包含 `light-chain-guanwang`（官网）、`light-chain-yunxiang-saas-server`（云享 SaaS）、`light-chain-tracking`（埋点）。

## 3. 第三方服务清单

| 服务 | 用途 |
|---|---|
| 纷享销客 fxiaoke.com | CRM 营销集成（身份识别、会员校验、可访问网站列表——有会员制内容门控） |
| GA4（G-8V5YN9P1P2）+ GTM（GTM-TWDVGVTJ） | 全量分析 |
| 百度统计（hm.baidu.com） | 国内流量 |
| Microsoft Clarity | 会话热力图 |
| DoubleClick | 广告转化 |
| 伙伴云营销脚本（a9.fspage.com/ec/kemai） | 官网访问归因 |

## 4. 控制台与异常（对 loomix 有直接参考价值）

- **站点自身 bug**：`[Tracking] Failed to send events: TypeError: Failed to fetch` → `Retrying in 5000ms (attempt 1/3)`——埋点上报失败后有 5 秒退避重试，定位在 chunk `3235-*.js`
- 8 个 `net::ERR_ABORTED`：RSC 预取请求被中止（Next.js Link 预取在导航/竞态下的正常行为）+ 埋点批量请求失败

## 5.  bundle 结构（63 个脚本，约 35 个唯一 chunk）

- `webpack-*.js` → 运行时
- `4bd1b696-*.js` → framework（React）
- `main-app-*.js` → App Router 入口
- `1255/6328/2619/1356/4909/1474/5553/8720/1648/7283/949/5075/6876/3775/1911/4277/6641/5889/6758/3235/3269-*.js` → 按路由/组件懒加载
- 页面级：`app/[local]/(pages)/layout-*`、`page-*`、`not-found-*`

## 6. 证据边界（REA 自带的诚实声明）

- 未捕获响应体（本次未开 `response_body`），API 返回结构未知
- Playwright 事件不暴露请求发起栈，收件顺序 ≠ 因果顺序
- 捕获结束时未完成的请求，后续活动未知
- 未做任何登录态操作（secrets 为空）

## 7. 下一步可深入的方向

1. **JS 静态分析**：`export-web-scripts` 导出全部脚本 → `analyze-javascript-application` 重建模块图与导出面
2. **Source map 还原**：`analyze-web-bundle --fetch-source-maps` 尝试拿回原始源码结构
3. **交互场景**：带 click/fill 的场景捕获登录流程、定价页交互（密钥走环境变量声明，不落盘）
4. **WebMCP 探测**：`discover-webmcp-tools` 看站点是否声明了 MCP 工具
5. **版本对比**：`compare-web-captures` 对比两个时间点的捕获，看迭代差异

---
*复现命令：`rea capture-browser-scenario scenario-lightchain.json --format json`（需 Node ≥22.19，本环境为 24.22）*
