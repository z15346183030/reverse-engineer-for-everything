# REA 自我逆向报告：用 REA 拆 REA

> 方法：`rea analyze-javascript-application /usr/local/lib/node_modules/rea-agents`（rea-agents 6.7.0）
> 证据：`rea-self-analysis.json`（362MB，evidence_id `ev_ad310df9...`，root digest `9e4781f4...`）
> 性质：npm 发布包是编译后的 dist 产物，无源码——这是标准逆向场景，REA 用自己的 JS 静态分析引擎拆了自己

## 1. 分析规模（REA 自报统计）

| 指标 | 数值 |
|---|---|
| 相关文件 | 6,857 |
| 读取文本 | 113.4 MB |
| 解析成功的 JS 文件 | 5,883（失败 29，失败率 0.5%——诚实上报） |
| 遍历 AST 节点 | 10,742,548 |
| 发现（findings） | 27,327 |
| 识别模块 | 304 |
| 嵌套 ASAR | 0 |
| 耗时 | 6 分 17 秒 |

## 2. 应用图（23,024 节点 / 30,244 边）

| 节点类型 | 数量 | 含义 |
|---|---|---|
| javascript-asset | 5,530 | 脚本/样式等静态资源 |
| source-module | 3,119 | 带源码信息的模块 |
| javascript-module | 2,416 | ES/CJS 模块 |
| **source-map** | **555** | 发布包自带 source map——可进一步还原原始结构 |
| artifact | 284 | 二进制附件（bridge 脚本等） |
| package | 215 | package.json 容器 |
| endpoint | 44 | 网络/路由端点候选 |
| static-parse-scope(unknown) | 10,854 | 未能归类的解析作用域 |

**模块目录分布**（逆向重建的分层架构）：

```
application/   787  ← 应用工作流层（setup、provider 编排）
cli-runtime/   365  ← CLI 运行时
browser/       293  ← Playwright/CDP 浏览器分析
artifacts/     162  ← 证据物管理
android/        54  ← APK 分析
adb/            46  ← 设备桥接
apktool/        26  ← 资源解码
captures/        9  ← 抓包
（另有 domain/ contracts/ server/ 等通过 source-module 体现）
```

## 3. 语义图（更深一层）

- 语义节点 **100,000**（触顶，coverage 状态 partial——预算内尽力而为）
- 语义关系 **98,985** 条：aliases 35,731 / reads 26,148 / defines 11,271 / reads-property 9,159 / captures 6,735 / **calls 1,602** / awaits 943 / writes 490
- 函数指纹 **2,250** 个，全部 complete（参数元数 + 归一化 AST 摘要，用于跨版本比对同一函数）

## 4. 安全相关结论（证据支持）

| 检查项 | 结果 | 证据 |
|---|---|---|
| 硬编码外部端点 | **无** | 44 个 network 端点全是候选字面量（HTTP 头名、Android 属性名如 `ro.kernel.qemu`），无任何 URL |
| 电话回家（phone-home） | **未发现** | 同上——纯本地工具的实证 |
| 自身是否 Electron 应用 | **不是** | browser_windows=0、IPC=0、preload=0、context_bridge=0；4 个 electron-renderer 节点只是 node_modules 里 playwright/pino 的 HTML 引用 |
| sender 校验观察点 | 68 处 | 对 IPC sender 的校验逻辑（防御性设计的痕迹） |

## 5. 证据链完整性

- 根产物 SHA-256 记录在案，**完整性矛盾 0 条**
- 清单 manifest ID + 图 ID + 图摘要全链路可复现
- 每个结论带 confidence / authority 标注，限制条件显式声明（如 CJS/ESM 绑定仅在产物容器内解析）

## 6. 这次自我逆向证明了什么

1. **REA 的 JS 分析引擎是真实力**：6857 文件、113MB、1074 万 AST 节点，6 分钟跑完，还诚实报告 29 个解析失败和 10 万节点的预算截断——不装全能
2. **分层架构从产物侧得到证实**：application → provider adapters → domain → CLI/MCP 的依赖流向，在模块目录分布里清晰可见
3. **「本地优先」不是口号**：整包扫完没有任何硬编码外部端点
4. **source map 是金矿**：555 个 source map 意味着理论上还能继续往下还原到更接近源码的形态（后续可用 `recover_javascript_sources` 深入）

## 7. 局限（REA 自己的声明）

- 语义图 10 万节点触顶，coverage=partial，非全量
- CommonJS/ESM 绑定关系仅从惰性语法恢复，只在产物容器内解析
- 29 个文件解析失败（未列名，需查 parse_failures 详情）
- 证据边界：「观察到」≠「推测出」，本报告所有数字均出自证据文件，未做外推

---
*复现：`rea analyze-javascript-application /usr/local/lib/node_modules/rea-agents --format json --max-heap-mb 2048`*
