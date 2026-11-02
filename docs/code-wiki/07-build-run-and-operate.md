# 07 — 构建、运行与操作

## 1. 前置条件

| 要求 | 版本 | 用途 |
|---|---|---|
| Python | 3.10+（技能 doctor 契约） | 生成图表、导入器、导出器、自检 |
| Python（CI 矩阵） | 3.11、3.12 | 完整仓库 Gate 套件 |
| Python 兼容腿 | 3.9 | `test-lint-a11y.py`、`lint-skin.py`、`verify-mermaid-import.py` |
| Playwright | CI lockfile 中 1.62.0 | PNG 导出、渲染 linter、截图渲染器、export-wait 测试 |
| Chromium | 由 Playwright 安装 | 同上 |
| Pillow | 12.1.1 | README WebP 缩略图及其测试 |
| Node.js | 22 | 仅 `claude plugin validate` |
| Git | 较新版本 | 版本/打包检查；`--require-no-bump` 需要完整历史 |

安装后的技能中，extractor、SVG exporter 和 self-check 只用标准库。只有 PNG 导出需要 Playwright。

### 一次性本地安装

```bash
python3 -m pip install playwright
python3 -m playwright install chromium

# 仅在需要重新生成 README 缩略图时
python3 -m pip install Pillow==12.1.1

# 仅在运行 Claude 包校验时
npm ci --prefix .github/ci/node
```

---

## 2. 安装技能

### 2.1 跨宿主（多数宿主推荐）

```bash
npx skills add cathrynlavery/diagram-design
```

更新：`npx skills update diagram-design`。

### 2.2 原生 marketplace

```text
# Claude Code
/plugin marketplace add cathrynlavery/diagram-design
/plugin install diagram-design@diagram-design

# Codex
codex plugin marketplace add cathrynlavery/diagram-design
codex plugin add diagram-design@diagram-design

# GitHub Copilot
copilot plugin marketplace add cathrynlavery/diagram-design
copilot plugin install diagram-design@diagram-design

# Factory Droid
droid plugin marketplace add https://github.com/cathrynlavery/diagram-design
droid plugin install diagram-design@diagram-design --scope user

# Pi
pi install https://github.com/cathrynlavery/diagram-design
```

Qoder、Kiro、OpenCode、Claude Cowork 和可编辑安装详见 README。

### 2.3 可编辑 checkout

```bash
git clone https://github.com/cathrynlavery/diagram-design.git ~/code/diagram-design
ln -s ~/code/diagram-design/skills/diagram-design ~/.claude/skills/diagram-design
# 对宿主实际扫描的根重复
```

Windows：使用 `cmd /c mklink /J <target> <source>`。不要把技能复制到托管插件目录之上。

---

## 3. 运行 Gallery 与模板

```bash
# 浏览所有示例，带 light / dark / full 标签
open skills/diagram-design/assets/index.html        # macOS
xdg-open skills/diagram-design/assets/index.html    # Linux

# 从模板新建图表
cp skills/diagram-design/assets/template.html my-diagram.html
cp skills/diagram-design/assets/template-full.html my-diagram.html
cp skills/diagram-design/assets/template-motion.html my-diagram.html
```

没有构建步骤。HTML 可直接在浏览器打开；唯一网络请求是 Google Fonts stylesheet（`system` 字体来源可完全移除）。

---

## 4. 通过 Agent 生成图表

1. 发现技能，让宿主加载 `SKILL.md`。
2. **样式 Gate。** 项目内首次使用时，Agent 检查 `style-guide.md` 是否仍是出厂默认值。若是，暂停并提供 onboarding、profile、粘贴 token 或默认值。
3. **选择。** 行为重要时先选语义模式，再选可视化类型。只加载匹配的 type reference。
4. **确认。** 绘制前说明类型、模式、尺寸预设和计划裁剪。
5. **写文件。** 从最接近的模板开始，替换 eyebrow、标题、SVG 主体和 `[diagram-slug]` 占位符。
6. **品味 Gate。** 执行 `SKILL.md` §9（类型匹配、删除测试、信号、技术、字体）。
7. **校验。**
   ```bash
   python3 skills/diagram-design/scripts/self_check.py my-diagram.html
   python3 scripts/lint-skin.py my-diagram.html              # 仅 checkout
   python3 scripts/verify-geometry.py my-diagram.html        # 仅 checkout
   ```

预期性质：

- 一个自包含 `.html`、内联 SVG、内嵌 CSS；
- 图表 SVG 有无障碍名称和描述；
- 除非拆分，≤9 节点；accent ≤2 个元素；
- 正交连接器、标签有遮罩；
- 完整静态帧；仅在请求时使用动效，且只用受审查 controller。

---

## 5. 导入已有图表

### 5.1 提取 IR

```bash
# draw.io — 原始 XML、deflate+base64、.drawio.png、.drawio.svg
python3 skills/diagram-design/scripts/drawio_extract.py platform.drawio
python3 skills/diagram-design/scripts/drawio_extract.py platform.drawio --page all --json

# Mermaid — .mmd、.mermaid 或含 fenced mermaid block 的 Markdown
python3 skills/diagram-design/scripts/mermaid_extract.py README.md --diagram all
python3 skills/diagram-design/scripts/mermaid_extract.py architecture.mmd --json

# Excalidraw — 仅 .excalidraw / .excalidraw.json 场景
python3 skills/diagram-design/scripts/excalidraw_extract.py whiteboard.excalidraw
python3 skills/diagram-design/scripts/excalidraw_extract.py whiteboard.excalidraw --json

# 写文件
python3 skills/diagram-design/scripts/drawio_extract.py file.drawio --out ir.json --json
```

通用参数：`--json`、`--max-rows N`、`--out PATH`。draw.io 另有 `--page`；Mermaid 另有 `--diagram`。退出码 2 表示不可读/不支持。

### 5.2 设定旋钮并重画

绘制前决定 format、size、detail、audience（见 [03](03-skill-runtime.md) §6）。然后加载选定类型的 reference 并重画。提取器从不自行选类型或删内容。

### 5.3 报告 fidelity ledger

说明合并、折叠或丢弃了什么。源坐标、调色板、字体和布局怪癖被丢弃；组件、关系、分组、方向被保留。

---

## 6. 导出

导出只能手动触发。

```bash
cd skills/diagram-design

# 同时生成 SVG 和 PNG（无格式参数时的默认）
python3 scripts/export_svg.py ../../my-diagram.html
python3 scripts/export_png.py ../../my-diagram.html ../../my-diagram.png 2

# 仅 SVG
python3 scripts/export_svg.py my-diagram.html --system-fonts

# 不同倍率的 PNG（1..4，可小数）
python3 scripts/export_png.py my-diagram.html out.png 1.5
```

命令面：

```text
/diagram-design:export-diagram path/to/diagram.html
/diagram-design:export-diagram path/to/diagram.html --svg-only
/diagram-design:export-diagram path/to/diagram.html --png-only --scale=3
/diagram-design:export-diagram path/to/diagram.html --registry
```

关键规则：

- 默认 SVG + PNG；`--svg-only` / `--png-only` 互斥。
- 单独 `--registry` 只生成 `<basename>.registry.json`，不需要 Playwright。
- 拒绝 gallery `assets/index.html`（多个 SVG）。
- 导出只含图表；编辑包装被丢弃。
- 动效图表的 PNG 使用 `?motion=static`，并要求完整静态帧。

---

## 7. Doctor

让 Agent 运行 `doctor` / `/doctor` / `/diagram-design:doctor`，或运行仓库验证器：

```bash
python3 scripts/verify-doctor.py
python3 scripts/verify-doctor.py --strict
python3 scripts/verify-doctor.py --json
```

检查 Python、Playwright/Chromium、预期脚本（维护者模式）、command/prompt 路由、常见路径错误。绝不安装包或修改文件。

---

## 8. 运行 Gate 套件

权威列表在 `.maintainer-policy.json` → `gates.local_commands`（86 条）。`CONTRIBUTING.md` 也列出同一列表。

```bash
# 按策略顺序运行全部 Gate，遇到首个失败停止
python3 - <<'PY'
import json, subprocess, sys
commands = json.load(open('.maintainer-policy.json'))['gates']['local_commands']
for index, command in enumerate(commands, 1):
    print(f"[{index}/{len(commands)}] {command}", flush=True)
    result = subprocess.run(command, shell=True)
    if result.returncode:
        sys.exit(result.returncode)
print("All gates passed")
PY
```

更快定向循环：

```bash
python3 scripts/lint-skin.py --all --baseline
python3 scripts/verify-geometry.py --all
python3 scripts/verify-docs-sync.py && python3 scripts/test-verify-docs-sync.py
python3 scripts/verify-motion.py --shipped
python3 scripts/verify-<type>.py --all && python3 scripts/test-verify-<type>.py
```

### 生成资源检查

```bash
python3 scripts/build-icons.py
git diff --ignore-space-at-eol --exit-code -- \
  skills/diagram-design/assets/icons.html \
  skills/diagram-design/references/primitive-icons.md

python3 scripts/build-exploded-examples.py --check
python3 scripts/build-axonometric-plan-examples.py --check
```

### 像素 Gate

```bash
python3 scripts/lint-render.py --self-test
python3 scripts/lint-render.py --all
python3 scripts/lint-render.py --fonts --all     # 本地真实字体度量
```

`--fonts` 用于本地字体检查；CI 阻断网络并使用 fallback 度量。

---

## 9. 重新生成已提交工件

| 工件 | 命令 | 提交内容 |
|---|---|---|
| 规范截图 | `python3 scripts/render-canonical-screenshots.py` | 45 张 PNG + `docs/screenshots/manifest.json` |
| README WebP 缩略图 | `python3 scripts/build-readme-thumbs.py` | `docs/screenshots/thumbs/*.webp` + manifest |
| 图标 | `python3 scripts/build-icons.py` | `assets/icons.html`、`references/primitive-icons.md` |
| Exploded 示例 | `python3 scripts/build-exploded-examples.py` | `example-exploded*.html` |
| Axonometric 平面图 | `python3 scripts/build-axonometric-plan-examples.py` | `example-axonometric-plan*.html` |
| OpenAI ZIP | `python3 scripts/build-openai-plugin-zip.py` | 不提交；上传 artifact |
| 版本自增 | `python3 scripts/bump-plugin-version.py [--minor|--major]` | 仅维护者/auto-bump，绝不在 PR |

规范截图渲染器：1440×1000 viewport、device scale 2、第一个 SVG、`document.fonts.ready`，写 digest manifest。

缩略图渲染器：网格预览 600px/q82，三个宽图 1400px/q90；`--check` 比较 digest 并解码每个接受的缩略图。

---

## 10. 常用操作配方

| 我想要… | 操作 |
|---|---|
| 查看所有类型 | 打开 `assets/index.html`；搜索默认展开 |
| 制作第一张图 | 复制模板，然后按匹配的 `type-*.md` 操作 |
| 匹配品牌 | 说 “onboard diagram-design to https://yoursite.com” |
| 跨项目复用品牌 | 保存 profile，在 `.diagram-design` 写 `profile: <slug>` |
| 修改默认 skin | 编辑 `references/style-guide.md`（或使用 profile） |
| 在安装后的技能中校验输出 | `python3 scripts/self_check.py <file> [--offline]` |
| 检查连接器和标签遮罩 | `python3 scripts/verify-geometry.py <file>` |
| 重画已有图表 | 运行对应 extractor，再按 import reference 操作 |
| 展示 block 树元数据 | 用 `--registry` 导出 |
| 调试 PNG 导出 | 运行 `test-export-wait.py`；检查 Playwright/Chromium |
| 确认文档没漂移 | `verify-docs-sync.py` + 其测试 |
| 准备发布 | 合并；auto-bump 处理版本；Pages 重新发布 gallery |

---

## 11. 故障排查

| 症状 | 原因 / 处理 |
|---|---|
| `ModuleNotFoundError: playwright` | 安装 Playwright 和 Chromium；技能不得自动安装 |
| 代理网络下 PNG 导出卡住 | 导出器 15s 后 `window.stop()` 取消停滞加载，并警告使用了 fallback 字体 |
| PNG 导出拒绝动效 | 页面未达到 `data-frame="static"`；打开 `?motion=static` 并确保完整静态帧 |
| Windows 有 `python3` 但不可用 | Microsoft Store 别名；doctor 会回退到可用的 `python`，不把无效命令当成功 |
| 渲染 linter 报告文字适配不同 | CI 用 fallback 度量；本地 `--fonts` 测真实字体 |
| 干净 checkout 后截图新鲜度失败 | 源 hash 已做 LF 归一化，CRLF 不是原因；源或 PNG 确实变了 |
| `verify-plugin-package.py` 报告版本变化 | 移除 PR 中的清单修改；版本在 `main` 自增 |
| 新增类型后 `verify-docs-sync.py` 失败 | 同时更新 gallery、README 树、计数、description hook 和文件树 |
| `self_check.py` 失败 | 阅读具名发现；常见原因：缺少 `role="img"`、裸 `title`/`desc` ID、远程资源、额外脚本、内容超出 viewBox |
| Gallery 链接 404 | 某个 tab 指向缺失示例；`verify-docs-sync.py` 会点名 |
| 构建后图标 diff | `scripts/vendor/icons/` 或生成器变了；重跑 `build-icons.py` 并提交两个生成文件 |
| `bump-plugin-version.py` 拒绝 | 清单版本不同步、semver 非法，或 SKILL.md 缺少唯一带引号的 `metadata.version` |

---

## 12. 先去哪里看

| 问题 | 文件 |
|---|---|
| 图表应该长什么样？ | `skills/diagram-design/references/style-guide.md` |
| 该用哪种类型？ | `skills/diagram-design/SKILL.md` §3 |
| 这种类型的布局？ | `skills/diagram-design/references/type-<name>.md` |
| 能放多少节点？ | `skills/diagram-design/references/layout-budget.md` |
| 导入用什么尺寸/viewBox？ | `skills/diagram-design/references/output-spec.md` |
| 怎么运行 Gate？ | `CONTRIBUTING.md` 与 `.maintainer-policy.json` |
| 为什么策略是这样？ | `docs/adr/` |
| 如何操作 checkout？ | `docs/cookbook.md` |
