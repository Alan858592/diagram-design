# 04 — Python 模块：关键类与函数

仓库有两类 Python 代码：

1. **技能随附脚本**：`skills/diagram-design/scripts/`，随技能安装，必须能在 Agent 宿主内运行。
2. **仓库工具**：`scripts/`，供维护者与 CI 使用，不随技能安装。

除非特别说明，脚本只使用 Python 标准库。

---

## 1. CLI 脚本的共同约定

| 约定 | 说明 |
|---|---|
| 入口 | `def main(argv: list[str] | None = None) -> int`，由 `if __name__ == "__main__"` 保护 |
| 退出码 | `0` 成功；`2` 不可读/不支持输入或用法错误（验证器用 `1` 表示发现违规） |
| 默认输出 | 人类可读的 Markdown digest |
| 完整输出 | `--json` 输出完整 IR |
| 有界读取 | 输入大小、元素数量、解压后大小、单条语句长度都有上限 |
| 信任边界 | 输入作为数据解析；不渲染、不抓取、不执行 |
| UTF-8 | `_configure_stdout_utf8()` 避免 Windows 旧代码页输出问题 |
| 确定性 | 无时间戳；提交工件不依赖无序输出；测试比较字节/digest |

---

## 2. `drawio_extract.py` — draw.io → IR

**目的：** 解码 draw.io 的各种容器格式，把 `mxGraphModel` 展平为绝对坐标节点和边，并报告结构信号。它不做任何设计决策。

### 2.1 常量与解码

| 名称 | 值 / 作用 |
|---|---|
| `PNG_MAGIC` | `b"\x89PNG\r\n\x1a\n"` |
| `MAX_INPUT_BYTES` | 32 MiB |
| `MAX_XML_BYTES` | 解压后 64 MiB |
| `MAX_PAGES` | 100 |
| `MAX_CELLS_PER_PAGE` | 10,000 |
| `PayloadTooLarge` | 压缩数据解压超过上限时抛出 |
| `_reject_unsafe_xml` | 拒绝 `<!DOCTYPE` / `<!ENTITY` |
| `_decompress_limited(data, wbits, limit)` | 流式 zlib 解压并硬性限制输出体积 |
| `_inflate(payload)` | 还原 draw.io 的 base64 → raw-deflate → URL-encoding 流程（`wbits` 试 -15/15/47） |
| `_png_embedded_xml(data)` | 读取 `tEXt` / `zTXt` / `iTXt` 中的 `mxfile` chunk |
| `_svg_embedded_xml(text)` | 读取 `content="..."` 属性中的 `mxfile` |
| `load_mxfile(path)` | 按 magic/扩展名分派：PNG、原始 XML、SVG 或压缩 payload |

### 2.2 样式、标签与 IR

```python
@dataclass
class Node:
    id: str
    label: str = ""
    shape: str = "rect"
    parent: str | None = None
    depth: int = 0
    x: float = 0.0; y: float = 0.0; w: float = 0.0; h: float = 0.0
    fill: str = ""; stroke: str = ""; font_color: str = ""
    dashed: bool = False; rounded: bool = False; container: bool = False
    children: list[str]
    link: str = ""
    attrs: dict[str, str]
    in_degree: int = 0; out_degree: int = 0

@dataclass
class Edge:
    id: str
    source: str | None
    target: str | None
    label: str = ""
    dashed: bool = False
    bidirectional: bool = False
    undirected: bool = False
    style_name: str = ""
    waypoints: int = 0
    stroke: str = ""

@dataclass
class Page:
    id: str; name: str; index: int
    nodes: list[Node]; edges: list[Edge]

    @property
    def node_map(self) -> dict[str, Node]:
        return {n.id: n for n in self.nodes}
```

| 函数 | 行为 |
|---|---|
| `parse_style(style)` | 将 `key=value;...` 解析为字典；无值 token 记为 `"1"` |
| `clean_label(value, is_html=False)` | 展平 HTML 标签/实体，但不解释内容 |
| `classify_shape(style)` | 将 draw.io 样式键映射为归一化形状名 |
| `shape_family(shape)` | 将 `icon:*` / `shape:*` 折叠为稳定家族，用于分析 |
| `parse_page(diagram, index)` | 三趟处理：收集原始 cell（解开 `<object>` / `<UserObject>`）、构建顶点并解析绝对几何、构建边；同时记录父子关系与深度 |
| `parse_file(path)` | 返回所有 page |
| `_has_cycle(nodes, edges)` | DFS 环检测 |
| `_aligned(boxes, tolerance=8.0)` | 检测共享左/上边和尺寸的 lane 状兄弟堆叠 |

### 2.3 `analyze(page)` 输出

`analyze` 只是建议，不是权威决策。它计算：

- 总数：`nodes_total`、`nodes_drawable`、`containers`、`leaves`、`edges_total`、`edges_labeled`、`edges_dangling`；
- `max_depth`、`shapes` 直方图、`has_cycle`；
- `hubs`（度数前 5）、`entry_points`、`terminals`、`orphans`；
- `type_candidates`（按信号强弱排序）：sequence、ER、swimlane、flowchart、state、architecture、nested、tree；
- `collapsible_groups`（子节点全为叶子的容器）；
- 预算标志：`over_node_budget`（>9）、`over_edge_budget`（>12）。

### 2.4 输出与 CLI

`digest(path, pages, selected, max_rows)` 渲染 Markdown digest；`to_json` 输出完整 IR。`select_pages` 接受索引、名称或 `all`。

```bash
python3 drawio_extract.py file.drawio [--page N|NAME|all] [--json] \
                              [--max-rows N] [--out PATH]
```

---

## 3. `mermaid_extract.py` — Mermaid → IR

**目的：** 不依赖 Mermaid 渲染器，把支持的 Mermaid 语法解析成与 draw.io 提取器相同的 digest 形状。

### 3.1 支持语法与上限

| 名称 | 值 |
|---|---|
| `SUPPORTED_KINDS` | `flowchart`、`sequenceDiagram`、`stateDiagram-v2`、`erDiagram` |
| `UNSUPPORTED_KINDS` | pie、mindmap、gitgraph、quadrantChart、timeline、C4、sankey、gantt、journey、classDiagram 等 |
| `MAX_SOURCE_BYTES` | 4 MiB |
| `MAX_NODES` / `MAX_EDGES` | 2,000 / 5,000 |
| `MAX_STATEMENT_CHARS` | 4,096 |
| `MAX_AUTONUMBER_DIGITS` | 15 |
| `MARKDOWN_SUFFIXES` | `.md`、`.markdown`、`.mdown`、`.mkd` |
| `MERMAID_SUFFIXES` | `.mmd`、`.mermaid` |

### 3.2 类

```python
class ExtractError(Exception)

@dataclass
class Node:
    id: str; label: str = ""; shape: str = "rect"
    parent: str | None = None; depth: int = 0; container: bool = False
    children: list[str]; fields: list[str]
    in_degree: int = 0; out_degree: int = 0

@dataclass
class Edge:
    id: str; source: str; target: str
    label: str = ""; style: str = "solid"; arrowhead: str = "arrow"
    bidirectional: bool = False; undirected: bool = False; order: int = 0

@dataclass
class Diagram:
    index: int; kind: str; source_line: int; direction: str = "TD"
    nodes: list[Node]; edges: list[Edge]
    fragments: list[dict]     # sequence alt/opt/loop/par 区域
    notes: list[str]
    autonumber: list[dict]    # start/step 指令
    discarded: dict[str, int]
```

`Diagram.add_node(...)` 按 ID 去重，更新 label/shape/parent，计算深度，并把子 ID 挂到父节点。`Diagram.add_edge(...)` 分配递增 ID 并限制边数量。`node_map` 是 ID 查找表。

### 3.3 解析流水线

| 函数 | 职责 |
|---|---|
| `load_blocks(path)` | 直接读取 `.mmd`/`.mermaid`，或从 Markdown 提取 fenced Mermaid block（处理 BOM） |
| `clean_label(value)` | 展平 Mermaid 标签 markup 和空白 |
| `_prepared_lines(block)` | 去除 frontmatter、注释、指令行；跟踪源行号 |
| `_kind_and_direction(...)` | 识别图表种类与 `TD`/`LR`/… 方向 |
| `_top_level_mask(text)` | 遮罩引号/括号内容，使顶层拆分安全 |
| `_logical_statements(...)` | 合并多行语句，拆分同一行中的多条语句 |
| `classify_shape(expression)` | 将 Mermaid 括号形式映射为形状（`rhombus`、`ellipse`、`stadium`、`subroutine`、`cylinder` 等） |
| `_operator_style(token)` | 将箭头 operator 解码为 style/arrowhead/bidirectional/undirected |
| `_edge_operators(text)` | 按顶层 operator 拆分链式表达式 |
| `_parse_flowchart`、`_parse_sequence`、`_parse_state`、`_parse_er` | 各语法解析；sequence 跟踪 actor、message、note、fragment、autonumber；state 跟踪描述和 `[*]` 端点；ER 跟踪实体和关系基数 |
| `parse_block(block)` | 按 kind 分派并计算最终度数 |
| `_finalize_degrees(diagram)` | 填充入/出度 |
| `_has_cycle(nodes, edges)` | 环检测 |
| `analyze(diagram)` | 与 draw.io 同类的结构 digest |
| `digest(...)`、`to_json(...)` | 人类 digest / 完整 IR |
| `select_blocks(blocks, selector)` | 默认第一个 block、数字索引或 `all` |

### 3.4 CLI

```bash
python3 mermaid_extract.py file.mmd|file.md [--diagram N|all] [--json] \
                                  [--max-rows N] [--out PATH]
```

---

## 4. `excalidraw_extract.py` — Excalidraw → IR

**目的：** 把 `.excalidraw` / `.excalidraw.json` 场景解析为节点、边、容器和分组。仅解析文本；拒绝 `.excalidraw.png` 和 `.excalidraw.svg` 导出文件。

### 4.1 常量

| 名称 | 值 |
|---|---|
| `MAX_INPUT_BYTES` | 16 MiB |
| `MAX_ELEMENTS` / `MAX_NODES` / `MAX_EDGES` | 10,000 / 2,000 / 5,000 |
| `NODE_SHAPES` | rectangle→rect、ellipse→ellipse、diamond→rhombus、image、embeddable/iframe→embed |
| `EDGE_TYPES` | arrow、line |
| `CONTAINER_TYPES` | frame、magicframe |

### 4.2 类与函数

```python
@dataclass
class Node:
    id: str; label: str = ""; shape: str = "rect"
    parent: str | None = None; depth: int = 0
    x: float; y: float; w: float; h: float
    fill: str = ""; stroke: str = ""; dashed: bool = False
    container: bool = False
    children: list[str]; groups: list[str]
    in_degree: int = 0; out_degree: int = 0

@dataclass
class Edge:
    id: str; source: str | None; target: str | None
    label: str = ""; dashed: bool = False
    bidirectional: bool = False; undirected: bool = False
    waypoints: int = 0; stroke: str = ""

@dataclass
class Scene:
    name: str; nodes: list[Node]; edges: list[Edge]
    discarded: dict[str, int]   # freedraw、images、links、embeds、deleted、unknown
```

| 函数 | 行为 |
|---|---|
| `load_scene_data(path)` | 校验 JSON 对象结构、限制输入大小、拒绝导出后缀、拒绝非有限数值 |
| `parse_scene(path, document)` | 从 rectangle/ellipse/diamond/image/embed 元素构建节点，合并绑定文本为标签，解析 group/frame，构建箭头/线并绑定起止 ID |
| `_has_cycle(nodes, edges)` | 环检测 |
| `analyze(scene)` | Digest：总数、形状、容器、分组、hubs、入口/终端、候选类型、预算标志 |
| `scene_bounds(scene)` | 已放置元素的包围盒 |
| `digest(path, scene, max_rows)` | Markdown digest |
| `to_json(path, scene)` | 完整 IR |

### 4.3 CLI

```bash
python3 excalidraw_extract.py scene.excalidraw [--json] [--max-rows N] [--out PATH]
```

---

## 5. `export_svg.py` — HTML → 独立 SVG

**目的：** 提取第一个顶层图表 SVG，并让它作为独立文件、在 Figma/Illustrator/PowerPoint 中、以及多个导出内联到同一页面时都能正确渲染。

### 5.1 转换流水线

```text
read HTML
  → extract_first_svg()             # 平衡扫描顶层 SVG，跳过注释/script/style
  → xmlify_attributes()             # 无值属性、无引号值 → XML 语法
  → normalize_html_entities()       # 文本和属性中的命名实体 → XML 安全文本
  → ensure_xmlns() + ensure_viewbox()
  → set_root_id(f"{slug}-root")
  → _html_css_rules()               # 将页面 <style> 解析为 _CssRule 树
  → _diagram_css_rules()            # 作用域化、丢弃页面 chrome、继承 body 颜色/字体
  → namespace_inline_animations()   # keyframe 名绑定到 root id
  → merge_style_into_defs()         # 单个 <defs><style>，除非 --system-fonts 否则带 Google Fonts @import
  → namespace_defs_ids(slug)        # marker/pattern/gradient/filter/mask ID + url()/href 引用
  → xml_safe_comments()
  → normalize_rgba_presentation_attrs()   # 在安全时把 rgba() 拆成 fill + fill-opacity
  → assert_export_gate()            # 有 class= 但无图表 CSS 时拒绝
  → 用 xml.etree.ElementTree 做良构性校验
```

### 5.2 关键函数

| 函数 | 职责 |
|---|---|
| `strip_css_comments(css)` | 删除真实注释，同时保留引号字符串和转义 |
| `html_tokens(source)` | 查找 SVG 时跳过注释/script/style 的 tokenizer |
| `mask_opaque_html(source)` | 将不可解析区域替换为等长空白，便于安全扫描 |
| `extract_first_svg(html)` | 返回第一个完整顶层 `<svg>…</svg>` |
| `xml_safe_comments(svg)` | 重写注释内的 `--`；拒绝浏览器提前闭合的注释 |
| `xmlify_attributes` / `normalize_html_entities` | 让 HTML 属性/实体语法符合 XML |
| `ensure_xmlns` / `ensure_viewbox` / `set_root_id` | 根元素契约 |
| `slug_for(path)` | 去掉扩展名的 basename；驱动 root ID 与 defs 前缀 |
| `_CssRule` / `_css_rule_tree` / `_css_rule_nodes` | 轻量 CSS 规则树，含 at-rule 分组 |
| `_keyframe_names(rules, root_id)` | 重命名 keyframe，避免内联图表共享动画名 |
| `scope_selector` / `retarget_root_selector` / `body_inherited_css` | CSS 作用域化与继承带出 |
| `is_chrome_selector` | 丢弃页面 chrome（`html`、`body`、`h1`、`.card`、`.footer` 等） |
| `_diagram_css_leaf` / `_diagram_css_rules` / `diagram_css_from_html` | 构建被带出的 stylesheet |
| `namespace_inline_animations` | 重写内联样式和 stylesheet 中的 keyframe 绑定 |
| `merge_style_into_defs` | 在 `<title>`/`<desc>` 后插入一个 `<defs><style>` |
| `namespace_defs_ids` | 最长 ID 优先加前缀，重写 `url(#…)` 和 `href="#…"`，包括 CSS 转义 |
| `normalize_rgba_presentation_attrs` | 将安全 `rgba()` 绘制拆为颜色 + 不透明度，兼容严格 SVG 消费者 |
| `has_diagram_stylesheet` / `assert_export_gate` | 拒绝有 class 样式但无规则的 SVG |
| `export_svg_document(html, source_path, system_fonts=False)` | 公共转换 API |
| `main` | CLI：拒绝 gallery、缺失源、非有限 viewBox；默认写 `<stem>.svg` |

### 5.3 CLI

```bash
python3 export_svg.py <diagram.html> [<out.svg>] [--system-fonts]
```

---

## 6. `export_png.py` — HTML → PNG

**目的：** 用 headless Chromium 栅格化图表 HTML 中的第一个 SVG。

| 函数 | 职责 |
|---|---|
| `exact_size_scale(value)` | 校验有限数值且在 1–4 闭区间内 |
| `rasterize(src, out, scale=2, network_idle_timeout_ms=15000, fallback_settle_ms=4000)` | 打开 `file://…?motion=static`，等待 network idle（停滞时用 `window.stop()` 取消），等待 `document.fonts.ready`，断言所有 motion root 均为 `data-frame="static"`，把 SVG 宽高固定为 viewBox，释放裁剪祖先，最后 `omit_background=True` 截图 |
| `main` | 解析 `src`、`out`、可选 `scale` |

依赖：`playwright.sync_api`。安装：

```bash
python -m pip install playwright
python -m playwright install chromium
```

---

## 7. `self_check.py` — 安装后输出校验

**目的：** 仓库 Gate 的无依赖蒸馏子集，安装后的 Agent 可对任意生成文件运行。

```bash
python3 self_check.py <file> [--offline]
```

对每个文件输出 `OK` 或 `FAIL` 及具体发现；任一文件失败则退出码为 1。

### 7.1 Parser 与检查

| 符号 | 职责 |
|---|---|
| `DiagramParser(HTMLParser)` | 不执行文档，捕获 SVG、styles、scripts、resources、motion 属性和安全问题 |
| `SvgExtent` / `ViewBoxParser` | 解析 viewBox 和绘制范围 |
| `painted_points(tag, attrs)` | 对 rect/circle/ellipse/line/path/text/image 等给出保守边界 |
| `canonical_controller()` | 从 `assets/template-motion.html` 读取受审查 controller 并归一化换行 |
| `check_document_completeness` | 拒绝截断文件、mojibake、不平衡容器 markup、缺失闭合标签 |
| `check_css_references` | 拒绝远程 CSS `url()`、`@import`、不安全引用 |
| `check_svgs` | 无障碍 SVG 契约：`role="img"`、可解析 `aria-labelledby`、首子 `<title>`、`<desc>` |
| `check_viewbox` | 绘制内容保持在 viewBox 内；尺寸有限且为正 |
| `check_scripts` | 只允许唯一 canonical controller；拒绝事件处理器、`srcdoc`、可执行 URL |
| `check_motion` | 蒸馏级检查模式/控件/status/reduced-motion/print/无 JS 契约 |
| `verify(path, offline=False)` | 返回发现列表 |
| `main` | 对多个文件运行 CLI |

---

## 8. 仓库构建器与共享 helper

### 8.1 `axonometry.py` — 两类 axonometric 图表共用投影

```python
iso(x, y, z) = [x - y, (x + y) / 2 - z]
```

| 符号 | 职责 |
|---|---|
| `Proj(ox, oy)` / `.iso(x, y, z)` | 带平移的 2:1 dimetric 投影 |
| `Rect(x0, y0, x1, y1, r=0)` | 圆角矩形 footprint；`r=0` 为盒子，`r=w/2=d/2` 为圆柱 |
| `Rect.centers()` / `.inset(m, r=None)` / `.attr()` | 角中心、内缩 footprint、属性序列化 |
| `corner(theta)` | 四分之一圆弧属于哪个角 |
| `point(P, rect, theta, z, k=None)` | 投影后的轮廓点 |
| `walk(P, rect, a, b, z)` | 从角度 `a` 到 `b` 的 SVG path 命令；每个角是 2:1 椭圆的一个四分之一 |
| `SKINS` | light/dark 各面色调，全部引用 style-guide 角色 |

构建器 import 此模块；验证器故意重新实现投影，避免构建器 bug 与检查器共享。

### 8.2 `build-exploded-examples.py`

确定性构建 `example-exploded*.html`。

| 符号 | 职责 |
|---|---|
| `Part` | 声明的盒子、厚度、层级、面色调、轮廓 |
| `draw_part` | 从后到前绘制部件投影轮廓和可见面 |
| `housing_front` | 外壳墙体和前缘，绘制在内部部件之后 |
| `levels_of` / `explode` | 按层级分配 `z`；间距为 `max(gap_k × 顶面高度, 3 × 厚度)` |
| `Figure` / `build_svg` / `exploded_animated` | 图形组装与可选动效变体 |
| `Key`、`key_layout`、`mini`、`draw_key_grid` | 键帽/轴体细节 |
| 示例函数 | `phone`、`unboxing`、`ai_stack`、`keyboard`、`app_stack`、`vault_detail`、`tools_detail`、`skills_detail`、`harness_detail`、`interface_detail` |
| `render_all` / `main` | 写出全部示例；`--check` 校验已提交输出 |

### 8.3 `build-axonometric-plan-examples.py`

| 符号 | 职责 |
|---|---|
| `Box` | 声明的 footprint、高度、层级、色调、轮廓 |
| `behind(a, b)` / `bbox` | 深度比较和屏幕边界 |
| `depth_sort` | 对重叠盒子做从后到前排序 |
| `depth_sort_phased` | 保持每个建设阶段连续的从后到前排序 |
| `draw_box` | 一个 `<g data-box>`，轮廓路径在最前 |
| `tag` | 位于不透明背板上的水平标签，以其平面点为中心 |
| `Room` / `Plan` | office/campus/coffee-shop/warehouse 平面 |
| `build_svg` / `build_phased` | 静态与分阶段动效图形 |
| `office`、`campus`、`coffee_shop`、`warehouse` | 示例数据 |
| `render_all` / `main` | 写出示例；`--check` 校验已提交输出 |

### 8.4 `build-icons.py`

抓取并归一化图标集。生成文件绝不手改。

| 函数 | 职责 |
|---|---|
| `vendor_path(icon)` | 本地缓存路径 |
| `fetch_tabler`、`fetch_simple`、`fetch_logz`、`fetch_devicon`、`fetch_url` | 按来源抓取 |
| `normalize_tabler` | 提取内部几何，包上归一化属性 |
| `normalize_simple` | 提取 path 并强制 `fill=currentColor` |
| `normalize_url` | 直接抓取的通用 SVG 归一化 |
| `normalize_devicon` | 保留 viewBox，处理彩色 fill |
| `normalize_logz` | 解析内嵌 `<style>`/class 填充 |
| `build` / `main` | 写出 `assets/icons.html` 和 `references/primitive-icons.md` |

运行后生成的这两个文件必须通过 `git diff --exit-code`。

### 8.5 截图流水线

| 文件 | 关键函数 |
|---|---|
| `screenshot_catalog.py` | `canonical_slugs()` 从 SKILL.md 读取类型顺序；`source_path`、`screenshot_path`、`sha256`、`sha256_text`（LF 归一化）、`png_dimensions` |
| `render-canonical-screenshots.py` | `main()` 以 1440×1000、device scale 2、第一个 SVG、`document.fonts.ready` 渲染 45 张 canonical light 示例，并写 `docs/screenshots/manifest.json` |
| `verify-screenshot-freshness.py` | `main()` 比较 slug 顺序、renderer 描述、源 digest、PNG digest 和尺寸 |
| `build-readme-thumbs.py` | `target()` 选择 600px/q82 网格或 1400px/q90 宽图；`decoded_size`；生成 WebP 与 `thumbs/manifest.json`；`--check` 只校验不写入 |

### 8.6 打包与版本 helper

| 文件 | 关键函数 |
|---|---|
| `bump-plugin-version.py` | `read_manifest`、`parse_version`、`bump(root, part)`、`render_skill_metadata_version`：原子地自增四个清单和 SKILL.md 的 MAJOR.MINOR |
| `plugin_version_history.py` | `changed(base, head)`、`last-bump(sha)`：auto-bump workflow 使用的 git 版本历史 |
| `verify-plugin-package.py` | `verify_versions`、`verify_manifest_identity`、`verify_marketplaces`、`verify_native_skill_path`、`verify_qoder_commands`、`verify_skill_metadata_version`、`verify_package` |
| `build-openai-plugin-zip.py` | `index_entries`、`read_blobs`、`package_path`、`check_listing`、`check_assets`、`root_manifest`、`main` |
| `build-openai-plugin-zip.py --check` | 只校验 listing 限制和 staged ZIP 内容，不写文件 |

### 8.7 修复工具

`fix-mojibake.py` 替换常见 UTF-8 mojibake 序列。它是一次性修复 helper，不属于图表流程。

---

## 9. 值得了解的内置验证器

33 个验证器共享设计而非框架。多数用标准库（`html.parser` 或带文档注释的 regex）解析 HTML，从 `data-*` 属性读取声明值，重算不变量并报告具名发现。

| 验证器 | 关键函数 / 方法 |
|---|---|
| `verify-contrast.py` | `parse_color`、`parse_tokens`、`relative_luminance`、`contrast`、`composite`、`measure`、`check`；读取 `style-guide.md` 表格并按 skin 计算 WCAG 2.x 比值 |
| `verify-geometry.py` | `Rect`、`parse_rects`、`path_segments`（Bezier/arc 展平）、`translation`、`connectors`、`check_connectors`、`shared_run`、`stacked_connectors`、`shared_ports`、`check` |
| `verify-docs-sync.py` | 17 类漂移：description hook、gallery 可达性、README 文件树、reference 链接/锚点、command 路由、描述长度、Factory 安装面、打包支持路径、类型计数、High-Level 检查表、dark Line token、字号契约、字体 link 一致性、西里尔 fallback 顺序、尺寸预设一致性、ADR 0004 拆分路由、Loop station origin |
| `lint-skin.py` | `ResourceParser`、`AccessibleSvgParser`、`lint_accessible_svgs`、`allowed_colors`、`lint_text`、`canonical_controller_digest`、`load_baseline` |
| `lint-render.py` | `shipped_assets`、`resolver_rule`、`block_network`、`launch`、`shoot`、`compare_release`、`clipping_findings`、`check`、`self_test`，以及移动端/gallery/export fixture |
| `verify-waterfall.py` | 声明 `data-role`/`data-value`/`data-name`/`data-carry`；守恒、共享比例、carry、打印值、正负号处理 |
| `verify-treemap.py` | 按相对误差测量面积、标签适配、cell 值 |
| `verify-sankey.py` | 流量守恒与 ribbon 几何 |
| `verify-polar.py` | 定量 polar 编码与变体一致性 |
| `verify-heatmap.py` | 单调不透明度 ramp、完整 N×M 网格、≤1 个焦点 cell |
| `verify-marimekko.py` | 列宽份额、段份额、共享绘图高度、系列顺序、标签绑定 |
| `verify-slopegraph.py` | 共享比例与端点值 |
| `verify-streamgraph.py` | 层厚度、堆叠连续性、对称基线、边界曲线 |
| `verify-ridgeline.py` | 共享振幅、基线间距、bin、打印范围 |
| `verify-bubble.py` | 共享轴比例、`r = K·√size`、绘制顺序、accent 唯一性 |
| `verify-bump.py` | rank 网格顶点、排列 rank、端点标签位置 |
| `verify-beeswarm.py` | 共享数值比例、确定性避让、无叠印、accent/半径唯一性 |
| `verify-box-plot.py` | 五数顺序、共享比例、Tukey 栅栏、分类绑定 |
| `verify-skin-polarity.py` | 图例语气声明与实际 ramp 一致，覆盖所有 skin |
| `verify-architecture-delta.py` | 稳定 ID、快照成员、状态、签名、位置、端点、ledger 覆盖 |
| `verify-architecture-containment.py` | ID、父子、几何、虚线 band vs 实线 box 语法、PaaS 位置、peering |
| `verify-exploded.py` | 轮廓、等间距、层级、引线、标签列、动画抬升 |
| `verify-axonometric-plan.py` | footprint、plate 边界、共享地面、绘制顺序、标签位置 |
| `verify-block-registry.py` | 唯一 `data-block-id`、可解析父级、名称、无环 |
| `verify-service-blueprint.py` | visibility line、stage grid、层清单、焦点 cell |
| `verify-motion.py` | 静态优先 fallback、controller 身份、控件、预算、确定性 |
| `verify-semantic-motion.py` | 语义路由文档与动效示例结构 |
| `verify-doctor.py` | Doctor 契约：运行时检查、脚本存在、路由接线、路径错误 |
| `verify-*-import.py` | 提取器 vs fixtures、文档同步、对抗输入、上限、具名失败 |

完整 Gate 目录见 [05-verification-and-ci.md](05-verification-and-ci.md)。

---

## 10. 模块速查

| 模块 | 入口 | 产出 | 依赖 |
|---|---|---|---|
| `drawio_extract.py` | `main()` | Markdown digest / JSON IR | 标准库 |
| `mermaid_extract.py` | `main()` | Markdown digest / JSON IR | 标准库 |
| `excalidraw_extract.py` | `main()` | Markdown digest / JSON IR | 标准库 |
| `export_svg.py` | `main()` / `export_svg_document()` | 独立 `.svg` | 标准库 |
| `export_png.py` | `main()` / `rasterize()` | `.png` | Playwright |
| `self_check.py` | `main()` / `verify()` | `OK` / findings | 标准库 + 随附模板 |
| `axonometry.py` | library | 投影 path data | 标准库 |
| `build-exploded-examples.py` | `main()` | `example-exploded*.html` | `axonometry`、模板 |
| `build-axonometric-plan-examples.py` | `main()` | `example-axonometric-plan*.html` | `axonometry`、模板 |
| `build-icons.py` | `main()` | `icons.html`、`primitive-icons.md` | 网络 + vendor SVG |
| `build-readme-thumbs.py` | `main()` | WebP 缩略图 + manifest | Pillow |
| `render-canonical-screenshots.py` | `main()` | 45 张 PNG + manifest | Playwright、`screenshot_catalog` |
| `bump-plugin-version.py` | `main()` | 清单 + SKILL 版本自增 | 标准库 |
| `verify-*.py` | `main()` | findings，退出 0/1/2 | 标准库（部分需要 Playwright） |
