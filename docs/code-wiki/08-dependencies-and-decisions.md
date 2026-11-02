# 08 — 依赖、决策与不变量

## 1. 内部依赖图

```text
                     ┌──────────────────────────────────┐
                     │  插件清单 + marketplace 文件       │
                     └──────────────┬───────────────────┘
                                    │ 解析
                                    ▼
                     ┌──────────────────────────────────┐
                     │ skills/diagram-design/SKILL.md    │
                     └───────┬───────────┬──────────────┘
             路由到          │           │  复制
                            ▼           ▼
              references/*.md         assets/template*.html
              （18 个跨领域 + 45 个类型参考）
                            │           │
                            │           └──► 生成的 .html
                            │                     │
                            └── 拥有 token ───────┤
                                                  ▼
                              scripts/self_check.py、export_svg.py、export_png.py
                                                  │
                                                  ▼
                              scripts/verify-*.py  ◄── 读取 assets、docs、manifests
                                                  │
                              scripts/test-verify-*.py（对抗伙伴）
```

### 模块级 import

只有五个仓库脚本 import 另一个本地模块：

| Importer | Import | 原因 |
|---|---|---|
| `build-exploded-examples.py` | `axonometry` | 所有 exploded 示例共用同一投影 |
| `build-axonometric-plan-examples.py` | `axonometry` | 平面图使用同一投影 |
| `render-canonical-screenshots.py` | `screenshot_catalog` | 规范 slug 顺序与 digest helper |
| `verify-screenshot-freshness.py` | `screenshot_catalog` | 同一规范顺序与 hash |
| `test-verify-screenshot-freshness.py` | `screenshot_catalog` | 构造 fixture |

其余脚本刻意保持独立。尤其是：

- **技能随附脚本不 import 仓库脚本**。安装后只依赖技能目录即可运行。
- **投影验证器不 import `axonometry`**。它们故意重新实现投影，避免构建器 bug 与检查器共享。
- **JSON/DOM 验证器不 import 渲染器**。它们用标准库解析源文本，因此除非明确是像素 Gate，否则不依赖浏览器。

### 文档依赖

| 消费方 | 依赖 |
|---|---|
| 每个 type reference | `style-guide.md` 的角色 token；`layout-budget.md` 的预算 |
| `SKILL.md` §3 | 每个 `type-*.md` 链接 |
| `commands/*.md`、`prompts/*.md` | 其路由 reference（`export.md`、`import-*.md`、`profiles.md`、`doctor.md`） |
| `README.md` | gallery、截图、manifest 计数、文件树 |
| `assets/index.html` | 所有已发布示例；每类型一个编号 tab |
| `docs/screenshots/manifest.json` | canonical 示例源与 renderer 设置 |
| `docs/screenshots/thumbs/manifest.json` | 源 PNG 与预览设置 |

`verify-docs-sync.py` 把这些文档依赖转为 CI 检查。

---

## 2. 外部依赖

| 依赖 | 版本 / 来源 | 使用者 | 是否必需 |
|---|---|---|---|
| Python 标准库 | 3.10+（3.9 兼容子集） | 所有脚本 | 是 |
| Playwright | `1.62.0`（CI lockfile） | `export_png.py`、`lint-render.py`、`render-canonical-screenshots.py`、`test-export-wait.py`、doctor 探测 | 仅浏览器功能 |
| Chromium | Playwright 安装的构建 | PNG 导出、渲染 lint、截图渲染 | 仅浏览器功能 |
| Pillow | `12.1.1` | `build-readme-thumbs.py`、`test-build-readme-thumbs.py` | 仅 README 预览 |
| Node.js | 22 | `claude plugin validate` | 仅包校验 |
| `@anthropic-ai/claude-code` | `2.1.229` | `claude plugin validate . --strict` | 仅包校验 |
| Git | 较新版本 | 版本历史、包检查、auto-bump、`--require-no-bump` | 维护者 Gate 必需 |
| GitHub Actions | hosted runners | CI、auto-bump、Pages | 仅发布 |
| Google Fonts | `fonts.googleapis.com` / `fonts.gstatic.com` | 默认 web 字体来源，生成的 HTML 与导出 SVG | 可选；`system` 可移除 |
| 上游图标源 | Tabler（MIT）、Simple Icons（CC0）、Devicon、logz、vendor URL | `build-icons.py` | 仅构建时 |

安装后的技能**没有自己的包管理器清单**，也不 vendoring Playwright 或 Pillow。宿主缺少它们时 PNG 导出不可用，doctor 只警告，不安装。

### 依赖策略

1. **标准库优先。** 新随附脚本必须论证任何第三方依赖。
2. **浏览器功能显式降级。** Doctor 把 Playwright 报为 `warn`，绝不自动安装。
3. **固定像素工具链。** Playwright 与 Chromium 通过 uv lockfile 一起固定，因为渲染 linter 的 oracle 是像素。
4. **固定缩略图工具链。** Pillow 固定为 `12.1.1`，因为 WebP 编解码属于已提交工件契约。
5. **网络是记录在案的例外。** 生成文件可以请求 Google Fonts；`system` 是无网络模式。导入器从不抓取源内容。

---

## 3. 平台假设

| 假设 | 处理 |
|---|---|
| Linux、macOS、Windows | CI 矩阵覆盖三种；需要时 shell 步骤使用 `bash` |
| Windows 可能没有 symlink | Cookbook 使用目录 junction（`mklink /J`） |
| Windows `python3` 可能是 Store stub | Doctor 先 `python3` 再 `python`，要求能报告工作版本 |
| Windows 旧代码页 | Extractor 调用 `_configure_stdout_utf8()` |
| CRLF checkout | `screenshot_catalog.sha256_text` 对 LF 归一化内容做 hash；`.gitattributes` 将 `SKILL.md` 固定为 LF |
| 字体 fallback 因 OS 而异 | 像素 Gate 只在 Linux Chromium 运行；源码 Gate 在所有平台运行 |
| 文件系统大小写敏感 | 检查中 ID、文件名、gallery slug 按大小写敏感处理 |
| 裁剪的 git 历史 | `verify-plugin-package.py <base>` 需要 base ref；CI package job 使用 `fetch-depth: 0` |

---

## 4. 架构决策记录

| ADR | 决策 | 实际后果 |
|---|---|---|
| 0001 | 默认静态；只有一个固定 motion controller | 禁止任意脚本；动效是按需增强 |
| 0002 | 语义模式不扩展可视化类型分类 | 模式增加行为并收紧预算，绝不成为新类型 |
| 0003 | `reveal` 是唯一允许的自动播放 | 加载时不会出现意外循环动画 |
| 0004 | SKILL.md 体积上限 + 触发词丰富 description | 大块 markup/预算移到 references；description 保留所有类型 hook |
| 0005 | 标签几何被验证，而非评审 | `verify-geometry.py` 是合并 Gate |
| 0006 | 客户 profile + marker 优先解析 | `.diagram-design` marker 优先于已安装样式指南 |
| 0007 | 新布局语法必须成为新类型 | 真正的新语法需要 reference、计数、示例与预算 |
| 0008 | 各宿主清单共享一个插件根 | 一棵技能树服务 Claude、Codex、Factory、Qoder |
| 0009 | 版本在合并后于 `main` 自增 | PR 不碰版本；auto-bump 校验精确 patch |
| 0010 | Block registry 元数据契约 | `data-block-*` ID/父级/名称是结构化的，并被机器检查 |
| 0011 | Waterfall 是 running-total 语法 | 守恒、carry、正负号处理被验证 |
| 0012 | Architecture delta 拓扑 + ledger | 稳定 ID、Before/Changes/After、完整 ledger |
| 0013 | Exploded axonometric 投影 | 一个投影函数；等间距；标签单列 |
| 0014 | Axonometric plan 投影与深度顺序 | footprint、不重叠、正确的从后到前绘制、每房间一个标签 |
| 0015 | Service blueprint visibility-line 语法 | customer/frontstage/backstage 层与 visibility line |
| 0015（state note） | State note 终止 | 状态图上的 note 正确终止 |
| 0016 | 合并需要同 head 的 Greptile 评分 | 评审针对最终合并的确切 head |

修改相关策略前先读 ADR。若 PR 定下了新策略，应在同一 PR 中添加 ADR。

---

## 5. 不变量

这些是仓库视为不可协商的陈述。每条都对应 Gate 或书面规则。

### 输出不变量

1. 一个自包含 HTML 文件；内联 SVG；除批准的 Google Fonts stylesheet 外无外部资源。
2. 结构几何在 4px 网格上；字号在角色阶梯上。
3. 每图最多 9 节点 / 12 箭头 / 2 accent 元素 / 2 callout（各类型预算可更紧）。
4. 每个图表 SVG 都有可解析的无障碍名称和描述，ID 带前缀。
5. 正交连接器；标签遮罩与描边 6–10px；无堆叠路线；遮罩不被后绘制节点裁剪。
6. 动效可选、静态优先、reduced-motion 安全，且只用一个受审查 controller。
7. 导入重画内容；绝不继承源坐标、调色板、字体或渲染器布局。
8. 每次导入都报告 fidelity ledger。

### 仓库不变量

9. 四个清单共享一个版本；SKILL.md 跟随 MAJOR.MINOR。
10. PR 绝不修改清单版本。
11. Marketplace 路径留在仓库内。
12. 各原生清单的插件 description 与 keywords 保持一致。
13. Gallery 可达所有已发布示例；每个编号 tab 都有匹配的 type reference。
14. 面向读者的类型计数等于 `type-*.md` 文件数量。
15. 每个路由 reference 链接可解析；每个打包支持路径存在。
16. 规范截图与 README 缩略图按 digest 匹配源文件。
17. 生成图标与生成器一致。
18. 数值或结构主张有验证器与对抗测试。

---

## 6. 新增图表类型

遵循 `CONTRIBUTING.md` 的 *Adding a new diagram type*。以下是被 Gate 强制执行的简化清单：

1. **编写** `skills/diagram-design/references/type-<name>.md`。
2. **路由** 到 `SKILL.md` §3，并把词汇 hook 加入 frontmatter description。
3. **添加** 三个示例：`example-<name>.html`、`-dark.html`、`-full.html`。
4. **注册** 到 `assets/index.html` 的下一个编号；同类型额外示例是字母变体（`28c`），紧跟父类型，使用 `data-parent-type` 和字母 eyebrow。
5. **提升计数**：
   - `scripts/verify-docs-sync.py`（`VISUAL_TYPE_COUNT`）
   - `scripts/verify-semantic-motion.py`（`VISUAL_TYPE_COUNT`）
   - `scripts/verify-screenshot-freshness.py` 中硬编码的 45
   - `scripts/render-canonical-screenshots.py` 中硬编码的 45
6. **修复套件报告的所有计数：** `README.md`、`SKILL.md`、`CONTRIBUTING.md`、`references/onboarding.md`、`references/semantic-patterns.md`，并在 ADR 0002 中添加新修正案。
7. **保持清单一致：** `.claude-plugin`、`.codex-plugin`、`.factory-plugin`、`.qoder-plugin` 的 `description` 与 `keywords` 相同。若选择表名称与 description 词汇不同，在 `verify-docs-sync.py` 添加 `DESCRIPTION_ALIASES` 条目。
8. **不要修改版本。**
9. **README 交付物：** 渲染 canonical 截图、添加缩略图 cell、运行 `build-readme-thumbs.py`、把命名变体加入变体句、把 `type-*.md` 加入文件树、更新 “01 to N”。
10. **预算行：** 把该类型行加入 `references/layout-budget.md`（不是 `SKILL.md`）。
11. **运行 `.maintainer-policy.json` 中的完整 Gate 套件。**

类型合并后，三个外部面需要手动更新计数：仓库 GitHub About description、Cathryn 的 profile README、diagramdesign.dev。CI 不覆盖它们。

---

## 7. 新增图表验证器

1. 识别主张（running total、面积份额、流量守恒、rank 等）。
2. 把每个显示值绑定到机器可读 `data-*` 属性。
3. 从声明重算图形。
4. 添加一个完整合法示例。
5. 至少添加一个**合理的渲染错误** fixture。
6. 命名成对文件：`scripts/verify-<type>.py` 与 `scripts/test-verify-<type>.py`。
7. 把两条命令加入 `gates.local_commands` 和 CI。
8. 如果这对文件没有出现在策略支持的列表中，`test-maintainer-policy.py` 会失败。

### Gate 编写规则

- 对无法解析的数据失败关闭；读不到的连接器不算通过。
- 面积主张用相对误差，声明总和用精确算术。
- 不要为让错误图形通过而放宽容差；修正图形或声明。
- 绝不把新示例加入 `lint-skin-baseline.txt`。

---

## 8. 被检查术语表

| 术语 | 精确含义 |
|---|---|
| **Visual type** | 有编号、有独立 `type-*.md` 和 gallery 编号的布局语法 |
| **Variant** | 共享某类型语法、但样式或命名不同的示例 |
| **Semantic pattern** | 带原语和更紧预算的行为优先路由层 |
| **Skin** | 当前 token 集；light/dark/terminal 都是 skin |
| **Shape family** | 导入分析使用的归一化节点形状 |
| **IR** | Extractor 的归一化 nodes/edges/containers/degrees |
| **Digest** | IR 的人类可读 Markdown 摘要 |
| **Fidelity ledger** | 导入的合并、折叠、丢弃报告 |
| **Gate** | 有通过/失败契约的确定性检查 |
| **Pair** | 一个 `verify-*` 加一个 `test-verify-*`，证明检查器能通过也能失败 |
| **Canonical example** | README 截图使用的 minimal-light `example-<type>.html` |
| **Snapshot** | Architecture delta 中的一个拓扑面板 |
| **Registry** | `data-block-*` 元数据 JSON sidecar |
| **Static frame** | 无 JS/无动效时的完整渲染 |
