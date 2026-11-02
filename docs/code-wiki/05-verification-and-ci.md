# 05 — 验证 Gate 与 CI

## 1. 为什么这个仓库有这么多 Gate

产品是一套指令加生成的 HTML。Agent 可能产出一个**看起来完成**、却违反自身声明的图表：treemap 面积与标签不符、waterfall 期末合计对不上、连接器被后绘制的节点遮住、gallery 链接指向不存在的示例、清单版本与兄弟清单漂移。

因此仓库把验证当作产品的一部分。每个可确定性检查的不变量都有脚本，每个脚本都有对抗式伙伴证明它确实会失败。

### 四类 Oracle

| Oracle | 测量对象 | 示例 |
|---|---|---|
| **源码** | 文本、markup、token 表、链接、包元数据 | `lint-skin.py`、`verify-docs-sync.py`、`verify-plugin-package.py` |
| **像素** | Chromium 实际绘制的画面 | `lint-render.py` |
| **几何/数学** | 重算的位置、弧线、面积、守恒 | `verify-geometry.py`、`verify-treemap.py`、`verify-waterfall.py`、`verify-sankey.py` |
| **字节** | Digest、清单身份、ZIP 内容 | `verify-screenshot-freshness.py`、`build-readme-thumbs.py --check`、`build-openai-plugin-zip.py --check` |

无法建立其主张的 Gate 必须以具名发现失败关闭。

---

## 2. `verify-*` / `test-verify-*` 模式

每个不变量成对存在：

- `scripts/verify-<subject>.py`：检查已发布资源或指定文件；退出 `0` 干净、`1` 有发现、`2` 用法错误。
- `scripts/test-verify-<subject>.py`：在内存中构造或变异 fixture，断言**合法用例通过**且**合理的错误用例失败**。测试脚本自我描述为覆盖“both polarities”。

规则：

- 绝不放宽测试来消除失败。修正事实来源或图形。
- 新的数值或结构主张需要验证器、一个完整合法示例，以及至少一个合理的渲染错误。
- 如果验证器或其对抗测试没有出现在 CI 支持的本地 Gate 列表中，`test-maintainer-policy.py` 会失败。
- 验证器证明**内部一致性**，不能证明正确数字匹配外部预算表或 API。该边界需要独立源数据文件与随技能打包的验证器。

---

## 3. Gate 目录

可执行列表是 `.maintainer-policy.json` 的 `gates.local_commands`，共 **86 条命令**。CI 与 `test-maintainer-policy.py` 与该文件保持一致。

### 3.1 包、发布与安装安全

| 命令 | 检查 |
|---|---|
| `python3 scripts/test-plugin-package.py` | 版本与 marketplace 包验证的回归测试，含对抗用例 |
| `python3 scripts/test-maintainer-policy.py` | 维护策略与原生清单、当前 CI Gate 一致 |
| `python3 scripts/test-build-openai-plugin-zip.py` | OpenAI 目录 listing 限制与上传 ZIP 内容 |
| `python3 scripts/verify-plugin-package.py --require-no-bump origin/main` | PR 未修改四个清单版本；marketplace 路径合法；技能已打包 |
| `npm ci --prefix .github/ci/node && …/claude plugin validate . --strict` | Claude marketplace + 插件 schema，警告视为错误 |
| `python3 scripts/test-install-scan.py` | 技能 HTML 注释不会触发 Hermes Agent 的 `html_comment_injection` 扫描规则 |
| `python3 scripts/test-build-icons-quoted-attributes.py` | URL SVG 归一化与引号写法无关 |
| `python3 scripts/test-build-icons-devicon.py` | Devicon viewBox 保留 |
| `python3 scripts/test-fix-mojibake.py` | Mojibake 修复工具行为 |

### 3.2 Skin、无障碍、动效与单文件安全

| 命令 | 检查 |
|---|---|
| `python3 scripts/test-lint-a11y.py` | 无障碍 SVG linter 的合法与对抗单元测试 |
| `python3 scripts/lint-skin.py --all --baseline` | 所有示例/模板：调色板、字体、a11y、资源、脚本、controller 身份 |
| `python3 scripts/test-verify-contrast.py` + `verify-contrast.py` | 所有 token 的 WCAG 比值、两种 skin、半透明合成 |
| `python3 scripts/verify-semantic-motion.py --markdown-only` | 语义模式路由词汇与 SKILL.md 体积上限 |
| `python3 scripts/verify-semantic-motion.py --example-only` | 动效示例结构与无障碍 |
| `python3 scripts/verify-motion.py --shipped` | 所有已发运动效模板/示例 |
| `python3 scripts/test-verify-motion.py` | 动效契约：fallback、控件、预算、确定性、controller 变异 |
| `python3 scripts/test-verify-semantic-motion.py` | 语义动效验证器的对抗行为 |
| `python3 scripts/lint-skin.py skills/diagram-design/assets/example-policy-trace-animated.html` | 针对动效示例的专项检查 |

### 3.3 渲染布局

| 命令 | 检查 |
|---|---|
| `python3 scripts/lint-render.py --self-test` | 检查器自身：裁剪检测、误报用例、网络隔离、DOM 恢复 |
| `python3 scripts/lint-render.py --all` | 渲染所有示例和模板；捕获 SVG 内容裁剪、SVG 塌陷、页面横向溢出、缺失本地资源、JS 错误 |
| `python3 scripts/test-export-wait.py` | PNG 导出器的停滞 webfont fallback（正常加载、超时、非超时错误）；需要 Playwright |

`lint-render.py` 使用**像素 oracle**，不是 `getBoundingClientRect()`：先按作者意图截图，再逐层释放裁剪祖先截图，然后 diff 两张图。网络在浏览器 resolver 层切断，请求路由作为第二层。`--fonts` 只放行两个 Google Fonts 主机且仅 HTTPS，用于本地测真实字体；CI 不启用（使用 fallback 字体度量，确定性更强）。

### 3.4 文档与路由同步

| 命令 | 检查 |
|---|---|
| `python3 scripts/verify-docs-sync.py` + `test-verify-docs-sync.py` | 17 类漂移：description hook、gallery 可达性、README 树、reference 链接/锚点、command/prompt 路由、描述长度、Factory 安装面、打包支持路径、类型计数、High-Level 检查表、dark Line token、字号契约、字体 link 一致性、西里尔 fallback 顺序、尺寸预设一致性、ADR 0004 拆分路由、Loop station origin |
| `python3 scripts/verify-sequence-oauth.py` + test | Sequence 组合片段语法与预算 |
| `python3 scripts/verify-doctor.py` + `test-verify-doctor.py` | Doctor 诊断契约与对抗用例 |

### 3.5 导入路径

| 命令 | 检查 |
|---|---|
| `python3 scripts/verify-drawio-import.py` + `test-verify-drawio-import.py` | 真实提取器 vs 四种容器格式 fixture；文档同步；对抗输入 |
| `python3 scripts/verify-mermaid-import.py` | 所有支持语法、多 block Markdown、对抗标签、信任边界、上限、具名失败 |
| `python3 scripts/verify-excalidraw-import.py` + `test-verify-excalidraw-import.py` | 场景解析、绑定标签、group/frame、对抗标签、上限、具名失败 |

### 3.6 通用几何与规范工件

| 命令 | 检查 |
|---|---|
| `python3 scripts/verify-geometry.py --all` + test | 标签遮罩 vs 后绘制节点；正交连接器；不贴边框/角；端口间距；堆叠路线 |
| `python3 scripts/verify-screenshot-freshness.py` + test | canonical 源与 PNG digest 匹配 `docs/screenshots/manifest.json`；源文件 hash 容忍 CRLF |
| `python3 scripts/test-build-readme-thumbs.py` + `build-readme-thumbs.py --check` | WebP 预览匹配 PNG、manifest、尺寸、全尺寸链接（Pillow 12.1.1） |
| `python3 scripts/test-self-check.py` | 打包自检通过合法文件并捕获对抗/截断/离线用例 |
| `python3 scripts/test-export-svg-standalone.py` | SVG 导出携带 CSS、命名隔离 defs ID、遵守 `--system-fonts` |

### 3.7 类型专属主张验证器

每一行是一个验证器加 `test-verify-<type>.py` 伙伴；多数支持 `--all` 检查所有已发布示例，少数（如 `polar`、`heatmap`、`dumbbell`）运行固定 canonical 集。

| 家族 | 不变量 |
|---|---|
| `polar` | 定量 polar 编码与变体一致性 |
| `heatmap` | 单调不透明度 ramp、完整 N×M 网格、≤1 焦点 cell |
| `architecture-delta` | 稳定 ID、快照成员、状态、签名、位置、端点、ledger 覆盖 |
| `architecture-containment` | ID、父子、几何、虚线 admin band、实线 traffic scope、PaaS 位置、peering |
| `exploded` | 轮廓、等间距、层级、引线、标签列、动画抬升 |
| `axonometric-plan` | footprint、plate 边界、共享地面、绘制顺序、每房间/建筑一个标签 |
| `block-registry` | 唯一 block ID、可解析父级、名称、无环父链 |
| `treemap` | 面积匹配标签值；标签适配 |
| `marimekko` | 列宽、段份额、共享绘图高度/gutter、系列顺序 |
| `dumbbell` | 有限域解析与标记 3:1 对比度 |
| `slopegraph` | 共享轴比例与端点值 |
| `streamgraph` | 层厚度、堆叠铺满、对称基线、曲线顶点 |
| `ridgeline` | 共享振幅、基线间距、bin、打印范围 |
| `sankey` | 流量守恒与 ribbon 几何 |
| `waterfall` | running-total 守恒、带符号桥、carry、打印值、正负号处理 |
| `service-blueprint` | visibility line、stage grid、层清单 |
| `bubble` | 共享比例、`r = K·√size`、绘制顺序、accent 唯一性 |
| `bump` | rank 网格、排列 rank、双轴标签位置 |
| `beeswarm` | 数值位置、确定性避让、无叠印 |
| `box-plot` | 五数顺序、共享比例、Tukey 栅栏、离群点在须外 |
| `skin-polarity` | 图例语气声明与实际 ramp 一致，覆盖所有 skin |

### 3.8 构建器一致性 Gate

| 命令 | 检查 |
|---|---|
| `python3 scripts/build-exploded-examples.py --check` | 已提交 exploded 示例与构建器输出一致 |
| `python3 scripts/build-axonometric-plan-examples.py --check` | 已提交 plan 示例与构建器输出一致 |
| `python3 scripts/build-icons.py && git diff --exit-code -- …/icons.html …/primitive-icons.md` | 生成的图标资源是最新的 |

### 3.9 历史基线

`scripts/lint-skin-baseline.txt` 只为 2.0 之前、确实早于当前 skin 的示例存在。这些文件仍会接受 a11y 检查。新示例不得加入基线。

---

## 4. 主要 linter 测什么

### 4.1 `lint-skin.py`（源码 oracle）

类别：

- `color`：颜色必须来自 `style-guide.md` 的调色板/角色表；拒绝纯黑。
- `font-family`：只允许白名单字体；Google Fonts 只允许批准的 `/css2` 端点。
- `a11y`：无障碍 SVG 契约：`role="img"`、可解析 `aria-labelledby`、首子 title/desc、前缀化 ID。
- `external-asset`：拒绝远程 `src`、CSS `@import`、非 fragment `url()`。
- `script`：只允许一个受审查 motion controller，换行归一化后逐字节一致；拒绝事件处理器、`srcdoc`、可执行 URL、额外脚本。

运行方式：

```bash
python3 scripts/lint-skin.py <file.html>          # 单个文件
python3 scripts/lint-skin.py --all --baseline     # 所有示例 + 模板
```

### 4.2 `lint-render.py`（像素 oracle）

- 启动捆绑 Chromium（或 `DIAGRAM_LINT_BROWSER_CHANNEL` 指定的 channel）。
- 在 resolver 层阻断所有主机；`--fonts` 仅放行 `fonts.googleapis.com` 与 `fonts.gstatic.com`，且必须 HTTPS。
- 分阶段 diff 作者绘制与释放裁剪后的绘制（SVG 自身，然后每个裁剪祖先）。
- 还覆盖移动端宽度、gallery 交互、marimekko/waterfall/excalidraw/模板，以及 PNG 导出路径。
- `--self-test` 断言检测能力、误报、网络隔离，以及测量后 DOM 字节完全一致。

### 4.3 `verify-geometry.py`

基于已发布模板的形状启发式：

- 节点 = 带描边的 `<rect>`，≥60×40；
- 标签遮罩 = `<rect>`，宽 20–200、高 8–14；
- 完全位于节点内的遮罩是合法徽章；
- 连接器 = 带箭头 marker 的 `<path>`/`<line>`；
- 曲线和弧线展平为弦。

报告：斜向直段、贴着节点边框的段、端点距角 8px 内、端口过密、堆叠路线、不可解析 path、被后绘制节点裁剪的遮罩。

---

## 5. CI

### 5.1 Workflow

| Workflow | 触发 | 作用 |
|---|---|---|
| `.github/workflows/ci.yml` | push/PR 到 `main` | Gate 套件 |
| `.github/workflows/auto-bump.yml` | push 到 `main` | 合并后同步版本自增 |
| `.github/workflows/pages.yml` | push 到 `main`、手动 | 发布 `skills/diagram-design/assets/` 到 GitHub Pages |
| `.github/workflows/ci-configuration.yml` | `.github/**` 变更 | zizmor workflow 安全审计 |

### 5.2 `ci.yml` job

| Job | 运行时 | 范围 |
|---|---|---|
| `plugin-package` | Ubuntu、Python 3.12、Node 22 | 包工具、维护策略、OpenAI ZIP、清单版本保护、`claude plugin validate --strict` |
| `python39-compat` | Ubuntu、Python 3.9 | `test-lint-a11y.py`、`lint-skin.py --all --baseline`、`verify-mermaid-import.py` |
| `validate` | Linux/Windows/macOS × Python 3.11/3.12 | 所有源码 Gate；步骤级 `if: always()` 让失败一起报告；生成 Markdown 汇总表 |
| `validate`（像素腿） | 仅 Ubuntu、Python 3.12 | Playwright、`test-export-wait.py`、`lint-render.py --self-test`、`lint-render.py --all`、README 缩略图 |

为什么只有一条像素腿：抗锯齿和字体 fallback 因 OS 而异；六条腿比较像素等于比较六个 oracle。阈值按 Linux Chromium 校准。

### 5.3 版本保护

PR 上 CI 检查**合并提交的第一父提交**（当前 base tip）：

```bash
python3 scripts/verify-plugin-package.py --require-no-bump HEAD^1
```

push 到 `main` 时运行 `--current-only`。避免把 PR 记录 base 之后发生的 auto-bump 误判为 PR 修改。

### 5.4 合并后

`auto-bump.yml`：

1. 如果 `main` tip 已改变同步版本字段，则跳过。
2. 检查自上次 bump 以来的每个 commit 是否带 `release:major` / `release:minor` 标签。
3. 默认 patch；否则最强标签获胜。
4. 校验精确 patch（路径与语义改动），上传 artifact。
5. 再检查 `main` 未前进，然后应用并独立复验 patch。
6. 只把 token 暴露给最后一步已验证的 push。

---

## 6. 运行与排查

```bash
# 完整本地套件与 .maintainer-policy.json 相同
python3 - <<'PY'
import json, subprocess
for command in json.load(open('.maintainer-policy.json'))['gates']['local_commands']:
    subprocess.run(command, shell=True, check=True)
PY
```

排查原则：

| 失败 | 第一反应 |
|---|---|
| `verify-plugin-package.py` 版本变化 | 移除清单修改；版本在 `main` 自增 |
| `lint-skin.py` 颜色/字体 | 在 `style-guide.md` 解析角色；绝不内联新 hex |
| `verify-geometry.py` | 把标签移到连接器空段或改路线；不要缩小遮罩 |
| `verify-screenshot-freshness.py` | 重渲染规范截图，并同时提交 PNG 与 manifest |
| `build-readme-thumbs.py --check` | 重新生成 WebP + manifest |
| `verify-docs-sync.py` | 修正路由文档或链接；不要删除路由词汇 |
| `verify-<type>.py` | 让几何与声明表达同一事实；绝不为适配标签挪动数据 |
| 图标 diff | 重跑 `build-icons.py` 并提交生成文件 |

逐项失败处理详见 `CONTRIBUTING.md` 的 *If a gate fails*。
