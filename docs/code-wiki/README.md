# Diagram Design — Code Wiki

本 Code Wiki 记录 **Diagram Design** 仓库：一个把编辑级质量图表绘制成单个自包含 HTML + 内联 SVG 文件的 Agent Skill。它不是常规 Web 应用，而是**文档驱动的技能包**加一套**确定性 Python 工具链**，用于导入、导出、构建并验证技能产出的工件。

本页是入口。每一章相对独立，但都会链接回相关源码。

> 本 Wiki 基于仓库当前 checkout 编写（插件版本 `2.6.136`）。若之后代码或 Gate 发生变化，以仓库与 `CONTRIBUTING.md` 为准。

---

## 1. 项目是什么

Diagram Design 给 AI Agent 提供一套写下来的设计系统、45 种布局语法和少量本地脚本。Agent 只加载需要的文档，然后写一个 `.html` 文件：内联 SVG、内嵌 CSS、无需构建步骤。仓库同时提供：

- 三个导入器，把 draw.io、Mermaid、Excalidraw 转为统一的中间表示（IR）；
- 两个导出器，生成独立 SVG 和栅格 PNG；
- 一个安装后 Agent 可对自身产出运行的自检脚本；
- 33 个不变量验证器和 43 个对抗式测试，保证图表声明、几何、打包与文档保持同步；
- Claude Code、Codex、Factory Droid、Qoder 四个原生插件清单；
- README 使用的可浏览 Gallery 与规范截图。

设计目标是把**编辑品味编码为约束**：一个强调色、1–2 个焦点元素、1px 细线、无阴影、所有结构坐标可被 4 整除。

### 快速事实

| 事实 | 值 |
|---|---|
| 项目名称 | Diagram Design |
| 作者 / 发布者 | Cathryn Lavery / LittleMight |
| 许可证 | MIT |
| 插件清单当前版本 | `2.6.136`（四个清单同步） |
| 技能 frontmatter 版本 | `2.6`（`SKILL.md` 的 `metadata.version`，跟随 MAJOR.MINOR） |
| 可视化图表类型 | 45（`skills/diagram-design/references/type-*.md`） |
| 语义行为模式 | 9（`references/semantic-patterns.md`） |
| 动效模式 | 4：`none`（默认）、`reveal`、`step`、`loop` |
| 已发布 HTML 资源 | `skills/diagram-design/assets/` 下 220 个文件 |
| 图标 | 87 个单色图标 |
| 仓库验证脚本 | 33 个 `scripts/verify-*.py` |
| 对抗测试脚本 | 43 个 `scripts/test-*.py` |
| 维护者 Gate 命令 | 86 条（`.maintainer-policy.json` 的 `gates.local_commands`） |
| 运行时依赖 | Python 3.10+；安装后技能脚本仅用标准库（PNG 导出需 Playwright） |
| 输出格式 | 单个自包含 `.html`；可选独立 `.svg`、`.png` 或 block registry `.json` |

---

## 2. 文档地图

| 章节 | 回答的问题 |
|---|---|
| [01-system-architecture.md](01-system-architecture.md) | 分层结构、两条主要运行流程、信任模型、关键架构决策 |
| [02-repository-map.md](02-repository-map.md) | 各目录与文件组的职责、命名约定、修改应落在哪里 |
| [03-skill-runtime.md](03-skill-runtime.md) | `SKILL.md` 如何路由、每个 reference 负责什么、设计系统、模板、无障碍与动效 |
| [04-python-modules.md](04-python-modules.md) | 提取器、导出器、自检、构建器、打包工具中的关键类与函数 |
| [05-verification-and-ci.md](05-verification-and-ci.md) | 各 Gate 家族验证什么、`verify-*` / `test-verify-*` 模式、CI 矩阵 |
| [06-packaging-and-distribution.md](06-packaging-and-distribution.md) | 四个清单、三个 marketplace、版本同步、auto-bump、OpenAI ZIP、安装面 |
| [07-build-run-and-operate.md](07-build-run-and-operate.md) | 前置条件、安装、生成、导入、导出、运行全部 Gate、重新生成工件、故障排查 |
| [08-dependencies-and-decisions.md](08-dependencies-and-decisions.md) | 内部依赖图、外部依赖、ADR 索引、不变量、新类型清单 |

### 推荐阅读路径

- **新贡献者：** [02](02-repository-map.md) → [03](03-skill-runtime.md) → [05](05-verification-and-ci.md)。
- **工具开发者：** [04](04-python-modules.md) → [05](05-verification-and-ci.md) → [07](07-build-run-and-operate.md)。
- **发布 / 打包负责人：** [06](06-packaging-and-distribution.md) → [08](08-dependencies-and-decisions.md)。
- **Agent Skill 作者：** [03](03-skill-runtime.md) → [07](07-build-run-and-operate.md)。

---

## 3. 三十秒快速开始

```bash
# 浏览全部图表（无需构建，无 JS 依赖）
open skills/diagram-design/assets/index.html        # macOS
xdg-open skills/diagram-design/assets/index.html    # Linux

# 在安装后的技能目录中校验一个生成的图表
python3 skills/diagram-design/scripts/self_check.py my-diagram.html

# 导出（手动触发，绝不自动执行）
python3 skills/diagram-design/scripts/export_svg.py my-diagram.html
python3 skills/diagram-design/scripts/export_png.py my-diagram.html my-diagram.png 2

# 运行仓库 Gate 套件（维护者 checkout）
python3 scripts/lint-skin.py --all --baseline
python3 scripts/lint-render.py --all
```

正式安装使用 `npx skills add cathrynlavery/diagram-design`。详见 [07-build-run-and-operate.md](07-build-run-and-operate.md)。

---

## 4. 术语

| 术语 | 含义 |
|---|---|
| **Agent Skill** | 一个目录（`skills/diagram-design/`），包含 `SKILL.md` 入口及随附 references/assets/scripts，由 Agent Skills 宿主发现 |
| **Type reference** | `references/type-<name>.md`；拥有一种可视化类型的布局语法、反模式和示例 |
| **Semantic pattern** | 行为优先的路由层（`references/semantic-patterns.md`）；只增加语义和更紧的预算，绝不新增第二套布局分类 |
| **Skin** | `references/style-guide.md` 中当前的颜色/字体 token 集；可手改，也可由 onboarding 生成 |
| **Variant** | `example-<type>.html`（minimal light）、`-dark`、`-full`；另有可选 terminal 与 motion 变体 |
| **IR** | 导入器输出的统一中间表示：节点、边、容器、度数、预算标志 |
| **Fidelity ledger** | 导入时报告的合并、折叠、丢弃清单 |
| **Gate** | 本地与 CI 运行的确定性检查 |
| **Oracle** | Gate 的判定依据：源文本、渲染像素、几何、数值不变量或包字节 |

---

## 5. 本 Wiki 不替代什么

仓库本身是事实来源。若 Wiki 与代码冲突，以代码和被纳入 Gate 的文档为准。具体权威文件：

- `skills/diagram-design/SKILL.md`：路由与品味 Gate；
- `skills/diagram-design/references/style-guide.md`：颜色与字体；
- `CONTRIBUTING.md`：Gate 套件与修改流程；
- `.maintainer-policy.json`：可执行 Gate 列表与发布策略；
- `docs/adr/`：已经定案的设计决策。
