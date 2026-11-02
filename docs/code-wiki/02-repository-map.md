# 02 — 仓库地图

## 1. 顶层目录树

```text
diagram-design/
├── .agents/plugins/marketplace.json      # Codex marketplace 目录
├── .claude-plugin/
│   ├── marketplace.json                  # Claude marketplace 目录
│   └── plugin.json                       # Claude 插件清单
├── .codex-plugin/plugin.json             # Codex 清单（含 OpenAI 目录 interface）
├── .factory-plugin/
│   ├── marketplace.json                  # Factory Droid marketplace
│   └── plugin.json                       # Factory Droid 清单
├── .qoder-plugin/plugin.json             # Qoder 原生插件清单
├── .github/
│   ├── workflows/ci.yml                  # 86 条 Gate 的验证矩阵
│   ├── workflows/auto-bump.yml           # 合并后版本自增
│   ├── workflows/pages.yml               # Gallery 部署
│   ├── workflows/ci-configuration.yml    # zizmor action 审计
│   ├── ci/python/                        # 固定 Playwright + Pillow 项目
│   ├── ci/node/                          # 固定 claude-code CLI
│   ├── ISSUE_TEMPLATE/ PULL_REQUEST_TEMPLATE.md CODEOWNERS ...
│   ├── pr-previews/                      # PR 视觉证据
│   └── pr-visual-proof/                  # 历史 PR 截图
├── commands/                             # Claude 插件 slash command 定义
│   ├── doctor.md  export-diagram.md
│   ├── import-drawio.md  import-mermaid.md  import-excalidraw.md
│   └── profile.md
├── prompts/                              # Pi prompt 模板
│   ├── doctor.md  export-diagram.md
│   ├── import-mermaid.md  import-excalidraw.md
│   └── profile.md
├── skills/diagram-design/                # 技能本体（规范安装根）
│   ├── SKILL.md                          # 路由 + 哲学 + 品味 Gate
│   ├── references/                       # 45 个类型参考 + 跨领域文档
│   ├── assets/                           # 模板、220 个示例、gallery、图标
│   └── scripts/                          # 导入器、导出器、self-check
├── scripts/                              # 仓库验证与构建工具
│   ├── lint-skin.py  lint-render.py
│   ├── verify-*.py                       # 33 个不变量验证器
│   ├── test-*.py                         # 43 个对抗回归测试
│   ├── build-*.py                        # 确定性工件构建器
│   ├── axonometry.py  screenshot_catalog.py  plugin_version_history.py
│   ├── fixtures/                         # draw.io、Mermaid、Excalidraw 输入
│   └── vendor/icons/                     # 图标源 SVG
├── docs/
│   ├── code-wiki/                        # 本文档集
│   ├── adr/                              # 架构决策记录
│   ├── cookbook.md                       # 操作手册
│   ├── screenshots/                      # 规范全尺寸 PNG + manifest.json
│   ├── screenshots/thumbs/               # README 用 WebP 预览 + manifest
│   ├── hero/                             # README hero/demo/社交图片
│   ├── pr-previews/                      # 大型 PR 前后对比图
│   └── superpowers/                      # 设计规格与实现计划
├── assets/                               # 插件清单使用的品牌 logo
├── README.md  CONTRIBUTING.md  AGENTS.md  CLAUDE.md
├── CHANGELOG.md  PRIVACY.md  SECURITY.md  CODE_OF_CONDUCT.md
├── THIRD_PARTY_LICENSES.md  LICENSE
└── .maintainer-policy.json  .gitattributes  .gitignore  .zizmor.yml
```

### 技能包内的数量

| 区域 | 数量 |
|---|---|
| 可视化类型参考（`references/type-*.md`） | 45 |
| 跨领域参考（style、layout、语义、动效、导入导出、onboarding、profiles、doctor） | 18 |
| `assets/` 下 HTML 文件 | 220 |
| 模板（`template*.html`） | 5：light、dark、full、motion、terminal |
| 图标定义 | 87 |
| 技能内随附脚本 | 6 |

---

## 2. 目录职责

### 2.1 分发清单（`.claude-plugin/`、`.codex-plugin/`、`.factory-plugin/`、`.qoder-plugin/`、`.agents/`）

每个原生宿主获得一个清单，包含相同的身份字段和同步的 `version`。清单都指向仓库根（或共享的 `skills/` 目录），因此一棵技能树服务所有宿主。`verify-plugin-package.py` 检查名称、描述、关键词、路径和版本是否一致。

- Claude：`marketplace.json` + `plugin.json`
- Codex：`.codex-plugin/plugin.json` + `.agents/plugins/marketplace.json`
- Factory Droid：`marketplace.json` + `plugin.json`
- Qoder：仅 `plugin.json`；直接加载规范技能和共享 commands

详见 [06-packaging-and-distribution.md](06-packaging-and-distribution.md)。

### 2.2 命令与 Prompt 面（`commands/`、`prompts/`）

它们是薄包装：把一次 Agent 调用路由到一个 reference 文件，不重新实现逻辑。如果命令或 prompt 丢失了目标 reference，`verify-docs-sync.py` 会失败。注意 Pi 没有 `import-drawio` prompt 模板；Claude 命令集包含全部六个面。

| 面 | 路由到 | Claude | Pi |
|---|---|---|---|
| `export-diagram` | `references/export.md` | 有 | 有 |
| `import-drawio` | `references/import-drawio.md` | 有 | 无 |
| `import-mermaid` | `references/import-mermaid.md` | 有 | 有 |
| `import-excalidraw` | `references/import-excalidraw.md` | 有 | 有 |
| `profile` | `references/profiles.md` | 有 | 有 |
| `doctor` | `references/doctor.md` | 有 | 有 |

### 2.3 技能本体（`skills/diagram-design/`）

- **`SKILL.md`**：启动时唯一加载的文件。包含 frontmatter description（发现面）、样式 Gate、哲学、语义/类型选择表、通用反模式、设计系统摘要、连接器规则、预算、品味 Gate、模板说明、导入旋钮和输出契约。
- **`references/`**：按需加载。45 个 `type-*.md` 拥有布局语法。跨领域文件负责共享关注点：
  - `style-guide.md`：颜色、字体、对比度契约、节点处理、CJK/西里尔规则
  - `layout-budget.md`：4px 网格、各类型预算、页面布局、summary card
  - `output-spec.md`：format/size/detail/audience 旋钮、字号阶梯
  - `semantic-patterns.md`：9 个行为模式
  - `animation.md`：动效模式与无障碍契约
  - `primitives-core.md`：精确 SVG markup 与连接器规则
  - `primitive-annotation.md`、`primitive-sketchy.md`、`primitive-terminal.md`、`primitive-icons.md`
  - `onboarding.md`、`profiles.md`、`doctor.md`
  - `import-drawio.md`、`import-mermaid.md`、`import-excalidraw.md`
  - `export.md`、`export-registry.md`
- **`assets/`**：五个模板、45 个 canonical light 示例及 dark/full 对应版本、命名变体、动效示例、导入示例、`index.html`（gallery）、`icons.html`（图标浏览器）。
- **`scripts/`**：安装后的运行时：三个 extractor、两个 exporter、`self_check.py`。除 `export_png.py` 需要 Playwright 外，其余必须仅用标准库。

### 2.4 仓库工具（`scripts/`）

根 `scripts/` 目录**不会随技能安装**（除自检脚本的蒸馏子集）。内容包括：

- **Linters：** `lint-skin.py`（源码级 skin、a11y、单文件安全）与 `lint-render.py`（Chromium 像素 oracle）。
- **Verifiers：** 每个不变量家族一个 `verify-<subject>.py`，共 33 个。
- **对抗测试：** 每个 verifier 一个 `test-verify-<subject>.py`，加上构建/打包回归测试，共 43 个。
- **Builders：** 图标、README 缩略图、规范截图、exploded/axonometric 示例、OpenAI ZIP。
- **共享 helper：** `axonometry.py`（2:1 dimetric 投影）、`screenshot_catalog.py`（规范 slug 顺序与 digest）、`plugin_version_history.py`（感知 git 的版本检查）。
- **Fixtures：** draw.io/Mermaid/Excalidraw 输入样本与故意错误的 HTML。
- **Vendor icons：** 按上游许可证/来源分组的源 SVG。

### 2.5 文档（`docs/`）

- `adr/`：已定案策略。修改相关策略前先阅读。
- `cookbook.md`：操作手册（安装、首次图表、onboarding、导入、导出、Gate 检查、prompt pack）。
- `screenshots/`：45 张 canonical light 图表，按 1440×1000 @2× 从第一个 SVG 渲染，附 `manifest.json` 记录源/截图 digest。
- `screenshots/thumbs/`：README 展示的 WebP 预览，由 `build-readme-thumbs.py` 生成。
- `hero/`：README banner、demo GIF、社交预览。
- `pr-previews/` 与 `.github/pr-*`：大型改动的视觉证据。
- `superpowers/`：重大特性的设计规格与实现计划。
- `code-wiki/`：本文档集。

### 2.6 CI（`.github/`）

`ci.yml` 有三个 job：`plugin-package`（Python 3.12 + Node 22）、`python39-compat`（三项兼容检查）、`validate`（Linux/Windows/macOS × Python 3.11/3.12 矩阵）。像素 Gate 只在 Ubuntu 3.12 运行，因为抗锯齿和字体 fallback 因操作系统而异。`auto-bump.yml` 负责合并后版本自增；`pages.yml` 将 `assets/` 发布到 GitHub Pages；`ci-configuration.yml` 用 zizmor 审计 workflow。

---

## 3. 命名约定

| 模式 | 含义 |
|---|---|
| `references/type-<slug>.md` | 一种可视化类型的布局语法 |
| `assets/example-<slug>.html` | canonical minimal-light 示例 |
| `assets/example-<slug>-dark.html` | minimal dark 变体 |
| `assets/example-<slug>-full.html` | full editorial 变体 |
| `assets/example-<slug>-terminal.html` | terminal 窗口变体（部分类型） |
| `assets/example-<slug>-animated.html` | 动效示例（按需） |
| `assets/example-<slug>-<variant>.html` | 与父类型共享 reference 的命名变体 |
| `assets/template*.html` | 新图表的脚手架 |
| `scripts/verify-<subject>.py` | 确定性不变量检查器 |
| `scripts/test-verify-<subject>.py` | 该检查器的对抗/合法 fixture |
| `scripts/build-<artifact>.py` | 确定性工件生成器 |
| `scripts/test-build-<artifact>.py` | 该生成器的回归测试 |
| `scripts/lint-<layer>.py` | 源码或渲染 linter |
| `commands/<surface>.md` | Claude slash command |
| `prompts/<surface>.md` | Pi prompt 模板 |

Gallery 编号：每种**类型**一个编号，`01` 到 `45`。同类型的额外示例是字母变体（`04a`、`04b`），紧跟父类型之后，并带 `data-parent-type`。

---

## 4. 修改应落在哪里

| 我想要… | 修改位置 | 随后运行 |
|---|---|---|
| 新增图表类型 | `references/type-<name>.md`、`SKILL.md` 选择表与 description、三个示例、gallery、`layout-budget.md`、README 缩略图/文件树、计数 | 完整 Gate 套件（见 [07](07-build-run-and-operate.md)） |
| 修改颜色或字体 | `references/style-guide.md` | `verify-contrast.py`、`lint-skin.py --all --baseline` |
| 修改布局规则或预算 | `references/layout-budget.md` | `verify-docs-sync.py`、`lint-render.py --all` |
| 修改 SVG markup 指南 | `references/primitives-core.md` | `verify-geometry.py --all`、`lint-skin.py` |
| 修改导入行为 | `scripts/<source>_extract.py`、对应 reference、command/prompt、verifier | `verify-<source>-import.py` + tests |
| 修改导出行为 | `scripts/export_svg.py` / `export_png.py`、`references/export.md` | `test-export-svg-standalone.py`、`test-export-wait.py` |
| 修改动效 | `assets/template-motion.html`、`references/animation.md`、示例 | `verify-motion.py --shipped`、`test-verify-motion.py` |
| 修改打包/版本 | 清单、`.maintainer-policy.json`、`scripts/*plugin*` | `test-plugin-package.py`、`verify-plugin-package.py` |
| 修改 gallery 或 README | `assets/index.html`、`README.md`、截图 | `verify-docs-sync.py`、`verify-screenshot-freshness.py` |
| 修改图标 | `scripts/vendor/icons/**`、`scripts/build-icons.py` | `build-icons.py` + `git diff --exit-code` |

---

## 5. 入口

| 入口 | 类型 | 面向 |
|---|---|---|
| `skills/diagram-design/SKILL.md` | Agent 发现 / 路由 | Agent 宿主 |
| `skills/diagram-design/assets/index.html` | 可浏览 gallery | 人 |
| `skills/diagram-design/scripts/*.py` | CLI 脚本 | Agent 与维护者 |
| `scripts/*.py` | 仓库 Gate 与构建器 | 维护者 / CI |
| `.claude-plugin/plugin.json` | 包身份 | Claude marketplace |
| `.codex-plugin/plugin.json` | 包身份 + OpenAI listing | Codex / ChatGPT |
| `.github/workflows/ci.yml` | 验证 | CI |
| `.maintainer-policy.json` | 可执行 Gate 列表与发布策略 | 维护者 / 自动化 |
