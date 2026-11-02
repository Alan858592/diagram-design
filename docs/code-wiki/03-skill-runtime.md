# 03 — 技能运行时：`SKILL.md`、References 与设计系统

安装后的技能是一个 Markdown 指令目录加 assets 和 scripts。绘制者是 Agent，这些文件负责定义约束。

规范安装根：`skills/diagram-design/`。

---

## 1. `SKILL.md` 作为路由器

`SKILL.md` 是 Agent 在被发现后唯一保证会加载的文件。它的 YAML frontmatter 包含：

- `name: diagram-design`
- 一个触发词丰富的 `description`，点名所有可视化类型（发现面；技能格式上限 1,024 字符，CI 检查其词汇覆盖）
- `license: MIT`
- `metadata.version`（`2.6`，跟随清单的 MAJOR.MINOR）

正文分为 13 节：

| 节 | 负责内容 |
|---|---|
| §0 首次设置 | 样式指南 Gate、项目 marker/profile 解析 |
| §1 哲学 | 删除优先、密度目标 4/10、accent 作为编辑信号 |
| §2 何时使用 | 使用/不使用判断、表格 vs 图表的取舍 |
| §3 选择 | 语义模式 → 可视化类型；45 行类型指南；绘制前确认 |
| §4 通用反模式 | 共享的 "AI slop" 失败模式 |
| §5 设计系统 | 语义角色、字体、节点处理、非拉丁标签 |
| §6 核心 SVG 原语 | 箭头、节点框、标签、图例、连接器规则 |
| §7 布局与间距 | 4px 网格、复杂度预算、拆分规则 |
| §8 Summary card 模式 | 变宽度的编辑式卡片 |
| §9 输出前检查表 | 品味 Gate，包含技术与字体检查 |
| §10 模板与变体 | 该复制哪个脚手架 |
| §11 导入 | draw.io / Mermaid / Excalidraw、四个旋钮、fidelity ledger |
| §12 输出 | 单个自包含 HTML、无障碍 SVG 契约、导出指引 |

拆分为 `primitives-core.md` 和 `layout-budget.md` 是 ADR 0004 的要求：`SKILL.md` 只保留路由词汇，把大块 markup 和预算表移出热路径。`verify-docs-sync.py` 检查拆分后的路由仍然存在。

### 渐进披露

| 请求 | 加载内容 |
|---|---|
| “画一个流程图” | `SKILL.md` + `type-flowchart.md` |
| “画架构图” | `SKILL.md` + `type-architecture.md` |
| “展示这次迁移改了什么” | `SKILL.md` + `type-architecture-delta.md` |
| “对比这两个策略请求为什么不同” | `SKILL.md` + `semantic-patterns.md` + `type-flowchart.md` |
| “给这个策略轨迹加动效” | 之前的选择 + `animation.md` |
| “把技能 onboard 到我的网站” | `SKILL.md` + `onboarding.md` + `style-guide.md` |
| “使用我保存的 Acme profile” | `SKILL.md` + `profiles.md` + `~/.diagram-design/profiles/acme.md` |
| “重画这个 .drawio 文件” | `SKILL.md` + `import-drawio.md` + `output-spec.md` + 选定类型 |
| 普通静态图表 | `SKILL.md` + 一个 type reference（仅在需要精确 markup 或预算行时加 `primitives-core.md` / `layout-budget.md`） |

这就是为什么新增类型不会增加已有类型的上下文成本。

---

## 2. Reference 目录

### 2.1 跨领域参考（18 个）

| 文件 | 职责 |
|---|---|
| `style-guide.md` | token、字体、对比度、节点处理、非拉丁标签规则的单一事实来源 |
| `layout-budget.md` | 4px 网格表、各类型复杂度、页面布局、summary card |
| `output-spec.md` | format × size × detail × audience、各尺寸字号阶梯、fidelity ledger、检查表 |
| `semantic-patterns.md` | 九个行为优先模式、组合规则 |
| `animation.md` | 动效模式、静态优先契约、控件、reduced motion、预算 |
| `primitives-core.md` | 背景、marker、连接器规则、节点框、标签遮罩、图例、a11y |
| `primitive-annotation.md` | 编辑式 callout 原语 |
| `primitive-sketchy.md` | 手绘 turbulence/displacement 滤镜 |
| `primitive-terminal.md` | CLI 窗口 chrome 与 terminal skin |
| `primitive-icons.md` | 87 个图标目录与使用规则 |
| `onboarding.md` | URL / skill / folder → token 提取、映射、预览、应用 |
| `profiles.md` | 命名客户 profile 与 `.diagram-design` marker 解析 |
| `doctor.md` | 环境诊断流程与报告格式 |
| `import-drawio.md` | draw.io 重画流程 |
| `import-mermaid.md` | Mermaid 重画流程 |
| `import-excalidraw.md` | Excalidraw 重画流程 |
| `export.md` | SVG/PNG 导出流程、尺寸、边界情况 |
| `export-registry.md` | block metadata JSON sidecar 契约 |

### 2.2 可视化类型参考（45 个）

按家族分组：

| 家族 | 类型 |
|---|---|
| **结构与系统** | Architecture、Architecture delta、IT current-state、High-level、Deployment、Dependency graph、Layer stack、Nested、Medallion |
| **流程与行为** | Flowchart、Sequence、State machine、Swimlane、Process、Loop |
| **数据** | ER / data model、Database schema、Data flow、DP integration、DP security matrix |
| **定量图表** | Bar、Line（含 slopegraph、ridgeline、streamgraph、bump）、Scatter（含 bubble、beeswarm、box plot）、Treemap（含 marimekko）、Heatmap、Polar、Radar、Waterfall、Gantt、Sankey |
| **层级与关系** | Tree、Org chart、Venn、Pyramid / funnel、Quadrant、Fishbone、Wardley map、Story map、Kanban |
| **时间与体验** | Timeline、User journey、Service blueprint |
| **空间与物理对象** | Exploded axonometric、Axonometric plan |
| **软件设计** | UML class |

合计 45 种。Architecture containment 和 traceable block decomposition 是**已有类型内部的语法**（分别为 Architecture 和 Tree），不是额外类型；它们在语义模式路由表中被选择。

权威选择表（含使用场景与链接）在 `SKILL.md` §3。各类型的权威限制在 `layout-budget.md`。

### 2.3 语义模式（9 个）

当行为、状态、强制或风险承载含义时，先选模式，再选可视化类型。

1. Fan-in queue / bottleneck → Data flow
2. Stage framework with semantic slots → Process
3. Unstructured input → structured artifact → Data flow
4. Paired policy-evaluation traces → Flowchart
5. Secure paved road → Architecture
6. Governance / control catalog → Layer stack
7. Compensating security layers → Layer stack
8. Traceable block decomposition → Tree（带 `data-block-*` 元数据）
9. Lifecycle phase map → State machine

模式拥有语义原语和更紧的预算；类型拥有布局语法。ADR 0002 禁止模式变成第二套分类。

---

## 3. 设计系统

### 3.1 语义角色

| 角色 | 用途 | 默认 light | 默认 dark |
|---|---|---|---|
| `paper` | 页面背景 / 默认节点填充 | `#f5f5f5` | `#2d3142` |
| `paper-2` | 容器/次级填充 | `#ececec` | `#393e53` |
| `ink` | 主文本/描边 | `#2d3142` | `#f5f5f5` |
| `muted` | 次级文本/默认箭头 | `#4f5d75` | `#bfc0c0` |
| `soft` | 副标签/边界标签 | `#5a6580` | `#949eb2` |
| `rule` | 细线 | `rgba(45,49,66,0.12)` | `rgba(245,245,245,0.12)` |
| `accent` | 1–2 个焦点元素 | `#bf4520` | `#f08a59` |
| `accent-tint` | accent 边框盒的填充 | `rgba(191,69,32,0.08)` | `rgba(240,138,89,0.10)` |
| `link` | HTTP/API/外部箭头 | `#2e5aa8` | `#739fdf` |

另有 `series-1` … `series-5`，仅用于多系列图表类型（如 Radar）。`accent` 始终标记焦点系列。

### 3.2 字体

| 角色 | 字体 | 字号 | 字重 |
|---|---|---|---|
| `title` | Instrument Serif | 1.75rem | 400 |
| `node-name` | Geist sans | 12px | 600 |
| `sublabel` | Geist Mono | 9px | 400 |
| `eyebrow` | Geist Mono | 7–8px | 500，字距 0.18em，大写 |
| `arrow-label` | Geist Mono | 8px | 400，字距 0.06em |
| `callout` | Instrument Serif italic | 14px | 400 |

规则：名称用 Geist sans，技术值（端口、URL、命令、字段类型）用 Geist Mono，页面标题用 Instrument Serif，绝不把 JetBrains Mono 当作万能“开发者”字体。

### 3.3 字体来源与对比度

- **`web`**（默认）：一次 Google Fonts stylesheet 请求，然后加载字体文件。
- **`system`**：无字体 link，无网络请求；字体栈回退到平台 serif/sans/mono。

对比度契约由 `scripts/verify-contrast.py` 机器检查：

- 常规文本对其实际背景 4.5:1；
- 大文本 3:1；
- 必要非文本标记 3:1；
- 半透明颜色先合成再测量；
- 同一 token 承担多个角色时，每个角色都必须通过。

### 3.4 网格、圆角、描边

| Token | 值 |
|---|---|
| 节点宽/高 | 80, 96, 112, 120, 128, 140, 144, 160, 180, 200, 240, 320 |
| 间距 | 20, 24, 32, 40, 48 |
| 内边距 | 8, 12, 16 |
| 圆角 | 4, 6, 8（README 一段式摘要写最大 10） |
| 描边 | 0.8 细、1 默认、1.2 强调 |
| 网格 | 4（仅结构坐标） |

字号、数据推导位置、文本基线、marker 和 1px 描边偏移刻意不在网格上。

### 3.5 复杂度预算

每张图的通用限制：**9 个节点、12 条箭头/转移、2 个 accent 元素、2 个 annotation callout**，外加动效限制（8 步、12 个标记项、2 个同时项）。各类型限制（lifeline、lane、bar、series、stage、table 等）在 `layout-budget.md`；超出时应拆成 overview + detail。

### 3.6 连接器六条硬规则

1. 只能正交：非轴线连接使用 `r=8` 圆角直角折线（紧凑布局最小 `r=6`）；禁止斜线。
2. 每个箭头标签都有不透明遮罩，与描边保留 6–10px 可见间距。
3. 不允许重叠或堆叠描边；平行路线偏移 ≥12px；单一交叉点使用 bridge/hop。
4. 同一节点边上的连接器各自取点：`L * k / (N + 1)`，间距 ≥12px（小节点 ≥8px）。
5. 不允许从非端点盒子背后穿过；不可避免时必须虚线、标签放在可见端。
6. 标签遮罩不能与后绘制的节点重叠（完全位于节点内部的徽章合法）。

`verify-geometry.py` 从几何上检查第 1–6 条。

### 3.7 无障碍 SVG 契约

1. `<svg>` 带 `role="img"` 和 `aria-labelledby`，同时命名 `<title>` 与 `<desc>`。
2. `<title>` 是第一个子元素，位于 `<defs>` 之前。
3. ID 按图表和变体加前缀：`<slug>-title` / `<slug>-desc`；裸 `title`/`desc` 被拒绝。
4. `<title>` 是主题短名（约等于 `<h1>`，≤60 字符）。
5. `<desc>` 是一句描述内容而非几何的句子。
6. 纯装饰 SVG 改用 `aria-hidden="true"`。

---

## 4. 模板与变体

| 变体 | 文件 | 使用场景 |
|---|---|---|
| Minimal light（默认） | `assets/template.html`、`example-<type>.html` | 可直接截图；图表 + 标题 |
| Minimal dark | `assets/template-dark.html`、`example-<type>-dark.html` | 深色站点、幻灯片、高对比 |
| Full editorial | `assets/template-full.html`、`example-<type>-full.html` | 长文中图表作为主角 |
| Motion | `assets/template-motion.html` | 仅在请求动效时 |
| Terminal | `assets/template-terminal.html` | CLI 窗口 chrome；不做品牌 token 化 |
| Consultant（quadrant） | `example-quadrant-consultant.html` | BCG/McKinsey 风格 2×2 |

所有模板都定义 paper 背景和箭头 marker（`arrow`、`arrow-accent`、`arrow-link`）。`template-motion.html` 只定义自己的前缀 marker，并携带唯一受审查 controller。

动效是增强层，不是新类型：

- 模式：`none`（默认）、`reveal`、`step`、`loop`。
- 首帧静态且无 JS 时必须承载完整含义。
- reduced motion 显示完整静态帧并隐藏/禁用控件。
- controller 必须逐字节复制自 `template-motion.html`；任意或修改过的脚本会被拒绝。
- 截图与导出使用 `?motion=static` 捕获。

---

## 5. 品牌 Onboarding、Profiles 与 Doctor

### 5.1 Onboarding

来源：网站 URL、已安装技能、本地文件夹/设计系统、粘贴 token，或保留默认。URL 流程：

1. 用户明确确认信任后抓取首页。
2. 从 body/背景/文本/卡片/CTA 提取颜色，从 `<h1>`/`<body>`/`<code>` 提取字体。
3. 将检测值映射为语义角色（`paper`、`ink`、`muted`、`paper-2`、`accent`、`link`）。
4. 校验对比度；对失败 token 提出调整值。
5. 展示 diff，并写入 `style-guide.md`。
6. 输出 fidelity receipt，列明采样 URL、颜色、字体、字体来源 URL 和 fallback。

### 5.2 Profiles

- 保存的 profile 位于 `~/.diagram-design/profiles/<slug>.md`。
- 项目 marker `.diagram-design` 内写 `profile: <slug>`，在样式 Gate 之前解析。
- 内置 `default` profile 受保护，不能被覆盖。
- marker 优先解析让并行客户工作区使用不同品牌，而无需修改已安装的 `style-guide.md`。

### 5.3 Doctor

一次性、无副作用的就绪报告：

- Python ≥3.10；优先 `python3`，再 `python`。
- Playwright import 和默认 headless Chromium 启动（只警告，绝不自动安装）。
- 维护者 checkout 模式：检查预期脚本、command/prompt 路由、常见路径错误。
- 参数：`--strict`（警告视为失败）、`--json`。

---

## 6. 导入旋钮与输出契约

### 6.1 四个旋钮

| 旋钮 | 选项 | 默认 |
|---|---|---|
| Format | `html`、`svg`、`png`、`html+png` | `html` |
| Size | `doc-inline`、`doc-wide`、`slide-16x9`、`slide-4x3`、`social-og`、`social-square`、`print-a4-landscape`、`print-a3-landscape`、`print-letter-landscape`、`fit` | `doc-inline` |
| Detail | `faithful`（≤24 节点，分带）、`balanced`（≤12）、`simplified`（≤7） | `balanced` |
| Audience | `engineer`、`mixed`、`executive` | `mixed` |

尺寸同时决定 `viewBox` 与字号阶梯。细节遵循固定降级阶梯：装饰 → 重复 → 叶子簇 → 基础设施。受众只改变措辞，不改变数量。

每次导入都以 fidelity ledger 结束：

```text
Detail: balanced · 12 source nodes → 8 drawn
Collapsed: "Token valid?" decision → edge label on Gateway → Auth
Dropped:   1 sticky note (unconnected in source)
Kept in full: the request path
```

### 6.2 输出契约

- 一个自包含 `.html` 文件。
- 内嵌 CSS；除批准的 Google Fonts stylesheet 外无外部资源。
- 内联 SVG；无外部图片。
- 默认静态；只有显式动效时才使用受审查 controller。
- 仅在明确要求时导出独立 SVG/PNG。
- `self_check.py` 必须对生成文件输出 `OK`（加 `--offline` 会在仍有字体 link 时失败）。

---

## 7. 技能刻意不做什么

- 不提供纯文本/unicode 图表工具。
- 不渲染 Mermaid 的自动布局；它把内容重画到自己的网格。
- 不承诺可编辑源格式；HTML/SVG 即为交付物。
- 不自动导出。
- 不自动安装依赖、抓取网站，或在没有明确动作时运行任意源文件。
