# 01 — 系统架构

## 1. 架构性质

Diagram Design 是**带验证工具链的技能包**，不是服务。它没有运行时服务器、数据库或应用二进制。架构有两个产物：

1. **安装后的技能**：Markdown 指令、设计参考、HTML 模板和 6 个小型 Python 脚本。AI Agent 解释指令并写出图表文件。
2. **仓库工具链**：导入器、导出器、构建器、验证器、截图渲染器和打包脚本，让技能的主张可测试。

两者共享同一棵源码树。这种共享是刻意的：references、示例、验证器和清单通过 CI Gate 保持同步，而不是靠约定。

### 核心原则

| 原则 | 体现位置 |
|---|---|
| **渐进披露（Progressive disclosure）** | `SKILL.md` 是路由器。普通图表只加载一个 type reference；语义、动效、导入、导出参考仅在相关时加载。 |
| **验证优先于评审** | 几何、对比度、数值编码、文档同步、包身份均由脚本检查（`scripts/verify-*.py`）。 |
| **每个关注点单一事实来源** | 颜色/字体 → `references/style-guide.md`；字号阶梯与尺寸预设 → `references/output-spec.md`；4px 网格与各类型预算 → `references/layout-budget.md`；Gate 列表 → `.maintainer-policy.json`。 |
| **默认静态输出** | 默认图表是静态 HTML + 内联 SVG。动效按需开启，且必须保留完整的无 JS 帧。 |
| **输入不可信** | 导入器把源文件当作数据，绝不渲染、抓取或执行源内容。 |
| **声明即数据** | 图表类型用 `data-*` 属性声明数值；验证器重算图形，而不是信任绘制。 |

---

## 2. 分层视图

```text
┌──────────────────────────────────────────────────────────────────────┐
│ 5. 分发层                                                             │
│    .claude-plugin/  .codex-plugin/  .factory-plugin/  .qoder-plugin/ │
│    .agents/plugins/marketplace.json  commands/  prompts/             │
├──────────────────────────────────────────────────────────────────────┤
│ 4. 技能运行时层（安装进宿主的部分）                                    │
│    skills/diagram-design/SKILL.md                                    │
│    references/  （路由、45 个类型、原语、语义、动效）                   │
│    assets/      （5 个模板、220 个示例、gallery、icons）               │
│    scripts/     （3 个导入器、2 个导出器、self_check）                 │
├──────────────────────────────────────────────────────────────────────┤
│ 3. 仓库验证与构建层                                                    │
│    scripts/verify-*.py   （33 个不变量验证器）                        │
│    scripts/test-*.py     （43 个对抗测试）                            │
│    scripts/lint-*.py     （skin + 渲染布局 linter）                   │
│    scripts/build-*.py    （icons、截图、缩略图、ZIP、投影构建器）        │
├──────────────────────────────────────────────────────────────────────┤
│ 2. 工件层                                                             │
│    docs/screenshots/*.png + manifest.json                            │
│    docs/screenshots/thumbs/*.webp + manifest.json                    │
│    README.md, CHANGELOG.md, docs/cookbook.md, docs/adr/              │
├──────────────────────────────────────────────────────────────────────┤
│ 1. CI / 发布自动化                                                    │
│    .github/workflows/ci.yml              （86 条 Gate 套件）          │
│    .github/workflows/auto-bump.yml       （合并后版本自增）             │
│    .github/workflows/pages.yml           （Gallery 发布）              │
│    .github/workflows/ci-configuration.yml（zizmor 审计）              │
└──────────────────────────────────────────────────────────────────────┘
```

### 组件职责

| 组件 | 职责 | 关键文件 |
|---|---|---|
| **Router** | 决定加载哪个 reference；承载哲学、选择表、反模式、品味 Gate | `SKILL.md` |
| **Design system** | 语义颜色角色、字体、字体来源、节点处理、对比度契约 | `references/style-guide.md` |
| **Layout owner** | 4px 网格、各类型复杂度预算、页面布局、summary card | `references/layout-budget.md` |
| **Output owner** | 格式 × 尺寸 × 细节 × 受众旋钮；字号阶梯；fidelity ledger | `references/output-spec.md` |
| **Type grammars** | 每种可视化类型一套布局语言 | `references/type-*.md`（45） |
| **Semantic routing** | 行为优先的模式与预算 | `references/semantic-patterns.md`（9 个模式） |
| **Motion contract** | 静态优先增强、模式、控件、reduced motion | `references/animation.md` |
| **Importers** | 源文件 → 归一化 IR digest/JSON | `scripts/drawio_extract.py`、`mermaid_extract.py`、`excalidraw_extract.py` |
| **Exporters** | HTML → 独立 SVG；HTML → PNG | `scripts/export_svg.py`、`export_png.py` |
| **Self-check** | 无需第三方依赖的安装后输出校验 | `scripts/self_check.py` |
| **Gallery** | 所有示例的 light/dark/full 可浏览目录 | `assets/index.html` |
| **Verifiers** | 证明 skin、几何、数值、打包、文档的不变量 | `scripts/verify-*.py` |
| **Builders** | 确定性生成投影示例、图标、截图、缩略图 | `scripts/build-*.py`、`scripts/axonometry.py` |
| **Packaging** | 清单身份、版本同步、marketplace 路径、OpenAI ZIP | `scripts/verify-plugin-package.py`、`bump-plugin-version.py`、`build-openai-plugin-zip.py` |

---

## 3. 运行流程 A — 生成图表

这是主要产品路径。绘制者是 Agent，不是程序。

```text
用户请求
   │
   ▼
宿主发现 diagram-design（仅可见 name + description）
   │
   ▼
加载 SKILL.md
   │
   ├── §0  样式指南 Gate（默认 token？项目 marker？profile？）
   │        └── 需要时执行 onboarding / profile 解析
   ├── §3  先选语义模式（行为重要时），再选可视化类型
   │        └── 路由时加载 references/semantic-patterns.md
   │        └── 只加载一个 references/type-<name>.md
   ├── §6  需要精确 markup 时加载 references/primitives-core.md
   ├── §7  加载 references/layout-budget.md 查该类型预算
   ├── §10 复制 template.html / template-dark.html / template-full.html
   │        （仅在请求动效时使用 template-motion.html）
   ├── §11 导入时设定四个旋钮并运行 extractor
   └── §9  运行品味 Gate
   │
   ▼
写一个自包含 .html（内联 SVG + 内嵌 CSS，可选受审查的 controller）
   │
   ├── python3 scripts/self_check.py <file>          （安装后的技能）
   ├── python3 scripts/lint-skin.py <file>           （仓库 checkout）
   └── python3 scripts/verify-geometry.py <file>     （仓库 checkout）
   │
   ▼
交付文件；仅在明确要求时导出
```

关键架构约束：Agent 的上下文是稀缺资源。因此 `SKILL.md` 不包含完整类型目录的布局规则；它只有选择表和链接。`verify-docs-sync.py` 强制路由保持完整，`verify-semantic-motion.py` 限制 `SKILL.md` 体积。

---

## 4. 运行流程 B — 导入已有图表

```text
.drawio / .drawio.xml / .drawio.png / .drawio.svg
.mmd / .mermaid / 含 fenced Mermaid block 的 Markdown
.excalidraw / .excalidraw.json
        │
        ▼
  extractor（仅标准库，输入视为数据）
        │  默认 Markdown digest
        │  --json 输出完整 IR
        ▼
  IR：nodes、edges、containers、hubs、sources/sinks、cycles、budget flags
        │
        ▼
  设定四个旋钮：format · size · detail · audience
        │
        ▼
  选择目标类型 → 加载该类型 reference
        │
        ▼
  Redraw（不是 convert）：丢弃源坐标/颜色/字体；
  保留组件、关系、分组、方向
        │
        ▼
  报告 fidelity ledger（merged / collapsed / dropped）
```

提取器**不做设计决策**。类型选择、细节裁剪与措辞仍由 Agent 负责。这种拆分让确定性部分可测试。

---

## 5. 运行流程 C — 导出

```text
diagram.html
   │
   ├── export_svg.py ──► 独立 .svg
   │     提取第一个顶层 SVG
   │     将页面 CSS 带入 <defs><style>
   │     将选择器作用域收拢到 #<slug>-root
   │     用文件 slug 给 <defs> ID 加命名空间
   │     为严格 SVG 消费者归一化 rgba()/transparent
   │     拒绝有 class 样式但没有图表 CSS 的片段
   │
   └── export_png.py ──► .png
         Headless Chromium（Playwright）
         加载 ?motion=static，等待字体，取消停滞加载
         按 viewBox 尺寸 × device scale 截取第一个 SVG
```

两种导出都**只包含图表**：`-full` 变体中的编辑包装、标题卡片和 footer 会被刻意丢弃。

---

## 6. 运行流程 D — 发布

```text
Pull request
   │
   ▼
CI（Linux / Windows / macOS × Python 3.11 / 3.12）
   ├── plugin-package job   （Python 3.12 + Node 22）
   ├── python39-compat job  （兼容性子集）
   └── validate job         （源码 Gate + Ubuntu 3.12 上的像素 Gate）
   │
   ▼
合并到 main（squash）
   │
   ├── Auto Version Bump workflow
   │     同步自增四个清单 + SKILL.md metadata.version
   │     校验精确 patch，然后推送到受保护 main
   │
   └── Deploy gallery workflow
         将 skills/diagram-design/assets 发布到 GitHub Pages
```

贡献者永不修改清单版本。由 `verify-plugin-package.py --require-no-bump` 强制，详见 ADR 0009。

---

## 7. 信任与安全边界

| 边界 | 规则 | 强制位置 |
|---|---|---|
| 导入源文件 | 视为不可信文本/XML/JSON。不渲染、不抓取、不执行。拒绝 DTD/entity 声明。解压体积与元素数量有上限。 | 三个 extractor + 对应 `verify-*-import.py` |
| 生成的 HTML | 只允许一个受审查的 motion controller。禁止远程资源、CSS `@import`、非 fragment `url()`、事件处理器、`srcdoc`、可执行 URL。 | `lint-skin.py`、`self_check.py`、`verify-motion.py` |
| 网络 | 默认输出只发起一次 Google Fonts 请求；`system` 字体来源完全不联网。Onboarding 只有在用户明确确认后才抓取网站。 | `style-guide.md`、`onboarding.md`、`PRIVACY.md` |
| 导出 | 拒绝 gallery 文件、空/无 SVG 源、有 class 样式但未带入 CSS 的 SVG。 | `export_svg.py`、`export_png.py`、`export.md` |
| 发布 | PR 不可改版本；marketplace 路径必须在仓库内；推送 bump 前独立复验。 | `verify-plugin-package.py`、`auto-bump.yml` |

---

## 8. 关键架构决策（ADR 摘要）

| ADR | 决策 |
|---|---|
| [0001](https://github.com/cathrynlavery/diagram-design/blob/main/docs/adr/0001-static-by-default-single-pinned-controller.md) | 默认静态；只有一个固定 motion controller |
| [0002](https://github.com/cathrynlavery/diagram-design/blob/main/docs/adr/0002-semantic-patterns-do-not-expand-the-taxonomy.md) | 语义模式不扩展可视化类型分类 |
| [0003](https://github.com/cathrynlavery/diagram-design/blob/main/docs/adr/0003-reveal-is-the-only-sanctioned-autoplay.md) | `reveal` 是唯一允许的自动播放模式 |
| [0004](https://github.com/cathrynlavery/diagram-design/blob/main/docs/adr/0004-skill-md-byte-cap-and-trigger-rich-description.md) | `SKILL.md` 体积上限 + 触发词丰富的 description |
| [0005](https://github.com/cathrynlavery/diagram-design/blob/main/docs/adr/0005-label-geometry-is-verified.md) | 标签几何必须被验证，而不是靠评审 |
| [0006](https://github.com/cathrynlavery/diagram-design/blob/main/docs/adr/0006-client-profiles-and-marker-first-resolution.md) | 客户 profile 与 marker 优先解析 |
| [0007](https://github.com/cathrynlavery/diagram-design/blob/main/docs/adr/0007-new-layout-grammars.md) | 新布局语法需要新可视化类型 |
| [0008](https://github.com/cathrynlavery/diagram-design/blob/main/docs/adr/0008-native-host-manifests-share-one-plugin-root.md) | 各宿主清单共享一个插件根 |
| [0009](https://github.com/cathrynlavery/diagram-design/blob/main/docs/adr/0009-versions-are-bumped-on-main-after-merge.md) | 版本在合并后于 `main` 上自增 |
| [0010](https://github.com/cathrynlavery/diagram-design/blob/main/docs/adr/0010-block-registry-metadata-contract.md) | block registry 元数据契约 |
| [0011](https://github.com/cathrynlavery/diagram-design/blob/main/docs/adr/0011-waterfall-is-a-running-total-grammar.md) | Waterfall 是 running-total 语法 |
| [0012](https://github.com/cathrynlavery/diagram-design/blob/main/docs/adr/0012-architecture-delta.md) | Architecture delta 拓扑与 ledger |
| [0013](https://github.com/cathrynlavery/diagram-design/blob/main/docs/adr/0013-exploded-axonometric.md) | Exploded axonometric 投影 |
| [0014](https://github.com/cathrynlavery/diagram-design/blob/main/docs/adr/0014-axonometric-plan.md) | Axonometric plan 投影与深度排序 |
| [0015](https://github.com/cathrynlavery/diagram-design/blob/main/docs/adr/0015-service-blueprint-is-a-visibility-line-grammar.md) | Service blueprint 的 visibility-line 语法 |
| [0016](https://github.com/cathrynlavery/diagram-design/blob/main/docs/adr/0016-merges-need-a-same-head-greptile-score.md) | 合并需要同 head 的 Greptile 评分 |

源码树中另有一份 [state-note termination ADR](https://github.com/cathrynlavery/diagram-design/blob/main/docs/adr/0015-state-note-termination.md)。

---

## 9. 一句话总结架构

Markdown 路由器选择 45 种布局语法之一和一套 skin，Agent 画出一个自包含 HTML 文件，Python 工具链则证明该图形、文档、打包和已发布截图彼此一致。
