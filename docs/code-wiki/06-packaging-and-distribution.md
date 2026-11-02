# 06 — 打包与分发

技能通过四个原生宿主包和若干独立安装器分发。所有渠道共用一棵规范技能树。

ADR 0008：各宿主清单共享一个插件根。ADR 0009：版本在合并后于 `main` 上自增，绝不在 PR 中修改。

---

## 1. 包面

| 面 | 文件 | 作用 |
|---|---|---|
| Claude Code | `.claude-plugin/plugin.json`、`.claude-plugin/marketplace.json` | 插件清单 + marketplace 目录 |
| Codex | `.codex-plugin/plugin.json`、`.agents/plugins/marketplace.json` | 插件清单 + marketplace 目录 |
| Factory Droid | `.factory-plugin/plugin.json`、`.factory-plugin/marketplace.json` | 插件清单 + marketplace 目录 |
| Qoder | `.qoder-plugin/plugin.json` | 原生插件清单；无独立 marketplace |
| Agent Skills CLI / 通用宿主 | 仓库根 + `skills/diagram-design/` | `npx skills add` 解析内部技能 |
| OpenAI 插件目录 | 生成的 ZIP | ChatGPT/Codex listing 上传 |

共享插件根包含：

```text
.claude-plugin/ plugin.json + marketplace.json
commands/         六个 slash command 定义
skills/           规范技能树
assets/           logo.png + composer-icon.png
LICENSE           每个副本都随附
```

清单指向仓库根（`./`）或 `./skills/`；不复制技能本体。

---

## 2. 清单身份

四个 `plugin.json` 必须一致：

`name`、`description`、`version`、`author`、`homepage`、`repository`、`license`、`keywords`。

| 字段 | 约束 |
|---|---|
| `name` | `diagram-design` |
| `description` | ≤500 字符（Cowork 安装兼容）；必须点名每种可视化类型的词汇 hook |
| `version` | 严格 `MAJOR.MINOR.PATCH` semver；四个同步 |
| `keywords` | 四个完全一致 |
| `skills` | Codex/Qoder 风格清单声明 `./skills/`；Claude 与 Factory 依赖根发现 |
| `commands` | Qoder 声明 `./commands/` |

Codex 的 `interface` block 同时是 OpenAI 目录 listing：

| 字段 | 限制 |
|---|---|
| `displayName` | 30 字符 |
| `shortDescription` | 30 字符 |
| `longDescription` | 4,000 字符 |
| `logo` / `composerIcon` | 必须是存在的声明资源 |

`verify-plugin-package.py` 与 `build-openai-plugin-zip.py` 强制这些限制。

---

## 3. 版本管理

### 3.1 同步版本

四个清单携带同一版本。`SKILL.md` 只带 MAJOR.MINOR（`2.6`），因为安装后的技能 frontmatter 不需要 patch 号。

| 位置 | 当前值 |
|---|---|
| 四个 `plugin.json` | `2.6.136` |
| `SKILL.md` `metadata.version` | `"2.6"` |

如果 SKILL 元数据偏离清单 MAJOR.MINOR，`verify-plugin-package.py` 失败。

### 3.2 PR 规则

贡献者不得修改任何清单版本。CI 运行：

```bash
python3 scripts/verify-plugin-package.py --require-no-bump HEAD^1
```

`HEAD^1` 是合并提交的第一父提交（当前 base tip），避免把 base 之后发生的 auto-bump 当作 PR 修改。push 到 `main` 时运行 `--current-only`。

### 3.3 合并后自增

`.github/workflows/auto-bump.yml`：

```text
push 到 main
  → guard：tip 是否已改变同步版本？
  → 找到上次 bump；检查其后的每个 commit 的 release 标签
  → part = major > minor > patch（最强标签获胜）
  → bump-plugin-version.py --major|--minor|(patch)
       • 四个清单
       • SKILL.md metadata.version（MAJOR.MINOR）
  → verify-plugin-package.py <source sha>
  → 断言 diff 只包含四个清单（+ SKILL.md）
  → 上传 version-bump.patch artifact
  → publish job：重新检查 main 未前进
  → 重新应用 patch，独立校验路径和精确 JSON/YAML 改动
  → 用窄权限 token 推送到受保护 main
```

连续合并可能合并为一次 bump。Workflow 使用 `concurrency: auto-bump-main` 且 `cancel-in-progress: false` 串行化。

### 3.4 Bump helper

`scripts/bump-plugin-version.py`：

- 读取四个清单；版本不同步则拒绝；
- 解析严格 semver；
- 按 `--major`、`--minor` 或默认 patch 计算下一版本；
- **先渲染**所有清单和 SKILL.md 修改，再写文件；SKILL.md 格式错误会中止整个 bump；
- 只写允许的五个文件。

`plugin_version_history.py` 提供 workflow 使用的 git helper（`changed(base, head)`、`last-bump(sha)`）。

---

## 4. 包验证

`verify-plugin-package.py` 有三种模式：

| 模式 | 使用方 | 含义 |
|---|---|---|
| `<base-ref>` | auto-bump | 版本必须相对 base 增加 |
| `--require-no-bump <base-ref>` | PR CI | 版本必须相对 base 不变 |
| `--current-only` | push CI | 只验证当前树 |

检查：

1. 四个清单存在、可解析、版本同步。
2. 版本是严格 semver。
3. 身份字段跨清单一致。
4. marketplace `name` 为 `diagram-design`；插件条目存在；source 为本地路径且不逃出 marketplace 根。
5. 共享技能路径 `skills/diagram-design/SKILL.md` 存在。
6. Qoder `commands` 路径存在。
7. SKILL.md 恰好有一个带引号的 `metadata.version`，匹配清单 MAJOR.MINOR。

Marketplace 路径必须是 `.` 或 `./…`；绝对路径、`..`、逃逸 symlink 都被拒绝。

---

## 5. OpenAI 插件 ZIP

`scripts/build-openai-plugin-zip.py` 从 **git index**（而非工作树）构建上传包，因此 ZIP 可由已提交内容复现。

```bash
python3 scripts/build-openai-plugin-zip.py --check     # 只校验
python3 scripts/build-openai-plugin-zip.py             # 写 dist/diagram-design-<version>-openai.zip
python3 scripts/build-openai-plugin-zip.py --draft     # 私有测试允许缺失 logo/icon
```

行为：

- 从 staged blob 读取 `.codex-plugin/plugin.json`；
- 校验 name/version 是安全文件名片段，并符合 Agent Plugins 名称模式；
- 检查 `displayName` / `shortDescription` / `longDescription` 限制；
- 解析 `logo` 和 `composerIcon` 并确认已 staged；
- 要求 `skills` 指向目录（不能是根）且存在 staged `SKILL.md`；
- 要求 `LICENSE`；
- 从 Codex 清单派生上传包的根 `plugin.json`，把 `interface` 移到 `extensions["com.openai"]`；
- 使用固定时间戳写出确定性 ZIP。

OpenAI 目录 listing 由维护者发布；贡献者不得自行提交插件。

---

## 6. 安装渠道

### 6.1 跨宿主 Agent Skills CLI

```bash
npx skills add cathrynlavery/diagram-design
npx skills update diagram-design
```

CLI 检测 `skills/diagram-design/SKILL.md` 并安装整个技能（references、assets、scripts）。选择多个根时推荐 symlink 一个规范副本；`--copy` 创建独立副本。独立安装不跟随 marketplace 更新。

### 6.2 原生 marketplace

| 宿主 | 命令 |
|---|---|
| Claude Code | `/plugin marketplace add cathrynlavery/diagram-design` → `/plugin install diagram-design@diagram-design`；在 `/plugin` 中开启 auto-update |
| Codex | `codex plugin marketplace add …` → `codex plugin add diagram-design@diagram-design` |
| GitHub Copilot | `copilot plugin marketplace add …` → `copilot plugin install …` |
| Factory Droid | `droid plugin marketplace add …` → `droid plugin install … --scope user` |
| Qoder | `qoder plugins validate ./diagram-design` → `qoder plugins install ./diagram-design`；或上传由 `.qoder-plugin`、`commands`、`skills`、`LICENSE` 构建的受限 ZIP |
| Pi | `pi install https://github.com/cathrynlavery/diagram-design` |
| Kiro | 导入仓库子目录 URL `…/tree/main/skills/diagram-design` |
| OpenCode | 将 `skills/diagram-design/` 复制/symlink 到 `.opencode/skills/` 或 `~/.config/opencode/skills/` |
| Claude Cowork | 把公开仓库镜像到组织私有仓库，再添加为组织 GitHub marketplace |

README 是各宿主安装细节的权威。Codex/Claude marketplace 按 Git commit 追踪；`npx skills` 安装是独立的。

### 6.3 可编辑 checkout

托管安装可能覆盖 `references/style-guide.md`。要直接定制，克隆仓库并把内部技能链接到宿主发现根：

```bash
ln -s ~/code/diagram-design/skills/diagram-design ~/.claude/skills/diagram-design
```

Windows 使用目录 junction（`mklink /J`），不要用文件 symlink。`~/.diagram-design/profiles/` 中的 profile 不受包更新影响；带 `.diagram-design` marker 的项目也不受影响。

---

## 7. Command 与 Prompt 面

每个插件命令都是指向一个 reference 的薄路由器。Docs-sync Gate 检查两端。

| 面 | Claude command | Pi prompt | 路由 reference |
|---|---|---|---|
| Export | `commands/export-diagram.md` | `prompts/export-diagram.md` | `references/export.md` |
| draw.io import | `commands/import-drawio.md` | — | `references/import-drawio.md` |
| Mermaid import | `commands/import-mermaid.md` | `prompts/import-mermaid.md` | `references/import-mermaid.md` |
| Excalidraw import | `commands/import-excalidraw.md` | `prompts/import-excalidraw.md` | `references/import-excalidraw.md` |
| Profile | `commands/profile.md` | `prompts/profile.md` | `references/profiles.md` |
| Doctor | `commands/doctor.md` | `prompts/doctor.md` | `references/doctor.md` |

`commands/export-diagram.md` 拥有参数契约：默认 SVG+PNG、`--svg-only`、`--png-only`、`--scale=1..4`、`--output`、`--registry`。单独 `--registry` 只输出 JSON sidecar，且不需要 Playwright。

---

## 8. 发布 Gallery

`.github/workflows/pages.yml` 在每次 push 到 `main` 或手动触发时，把 `skills/diagram-design/assets` 发布到 GitHub Pages。使用 `actions/configure-pages`、`upload-pages-artifact` 和 `deploy-pages`，环境为 `github-pages`。因此每个合并的 PR 都会更新线上 gallery，无需额外部署步骤。

---

## 9. 包不变量

1. 一个版本、四个清单、一个 SKILL 元数据 MAJOR.MINOR。
2. 所有地方的 description 与 keywords 一致。
3. Marketplace 路径留在仓库内。
4. 共享技能保持在 `skills/diagram-design/SKILL.md`。
5. PR 绝不 bump 版本。
6. OpenAI ZIP 只含 staged、白名单内容加派生根清单。
7. Commands 与 prompts 路由到随附 reference。
8. 生成 listing 资源是真实声明的文件。
