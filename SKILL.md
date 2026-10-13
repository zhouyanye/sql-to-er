---
name: er-diagram
description: Generate classic Chen-style ER diagrams from SQL DDL, pasted CREATE TABLE, or natural-language data model requests. Trigger on ER/ERD/entity-relationship/schema diagram/SQL转ER图, or when user asks to design tables and draw an ER diagram. Outputs only what the user asked for (Chen PNG by default; Mermaid/Word/GoJS only on request). After PNG is generated, MUST deliver it as an image attachment (not just a file path in text). SQL parse via yanleaf.com API with local fallback.
---

# ER Diagram (SQL → ER)

经典 Chen 风格 ER 图 +（按需）Word 三线表 / Mermaid / GoJS。

## 触发条件

在以下情况使用：

- 用户提供 `.sql` / 粘贴 `CREATE TABLE`，并要求出图或设计文档
- 用户明确要求：ER 图 / ERD / SQL转ER / 数据库设计文档
- 用户用自然语言描述业务并要求「设计表结构并画 ER 图」（由**当前模型**设计 list-of-tables，**不要**调用 yanleaf 的 AI 接口）

不要在仅讨论 SQL 语法、改某一个字段时自动出图。

## 产出什么（按需，不要默认全出）

**只生成用户要的东西。** 未点名的格式不要跑、不要塞给用户。

| 用户说法 | 生成 |
|----------|------|
| 画 ER 图 / 生成 ERD / 看看图 | **仅 PNG**（`--format png`） |
| 要 Mermaid / 要代码块图 | **仅 Mermaid**（`--format mermaid`） |
| 数据库设计文档 / 三线表 / Word / docx | **仅 Word**（`--format word`） |
| GoJS / 给前端用的 JSON | **仅 GoJS**（`--format gojs`） |
| 全部 / 图和文档都要 / 导出全套 | `--format all` |

默认（只说「画个 ER 图」）：**PNG only**。不要默认 `--format all`。

## 必须：把 PNG 当图片发出去（OpenClaw / 渠道）

生成 PNG 后，**禁止只回复文件路径文字**。用户要看到的是图片，不是 `已生成 /path/to/erd.png`。

### 推荐写法（Agent 最终回复）

简短说明 + **单独一行**媒体标记（路径必须真实存在、可读）：

```text
已生成经典 Chen 风格 ER 图（表 user，1 张实体）。
MEDIA:./erd.png
```

或工作区内相对路径 / 绝对路径均可，例如：

```text
MEDIA:/root/.openclaw/workspace/erd_user.png
```

规则：

1. `MEDIA:` 必须单独成行（前后可有说明文字，但该行不要夹其它内容）。
2. 优先把 PNG 写到**当前工作区**（如 workspace 根目录 `./erd.png`），再引用相对路径，兼容沙箱。
3. 若平台提供结构化附件字段，也可使用：`media` / `mediaUrl` / `path` / `filePath` 指向同一 PNG。
4. 不要只说「图片在 xxx」而不带 `MEDIA:` 或等价附件字段——渠道不会自动把路径变成图片。

### CLI 自测（Gateway 所在机器）

```bash
openclaw message send --target <chat_id> --media /path/to/erd.png --message "ER 图"
```

见：https://docs.openclaw.ai/zh-CN/nodes/images  
FAQ：Skill 生成了图但没发出去 → 出站附件须用 `media` / `mediaUrl` / `path` / `filePath` 或回复中的 `MEDIA:` 行。

## SQL 路径

```bash
# 只要图（最常见）— 输出到工作区便于 MEDIA 引用
python scripts/sql_to_erd.py schema.sql --out ./erd --format png

# 只要 Word
python scripts/sql_to_erd.py schema.sql --out ./design --format word

# 用户明确要全套
python scripts/sql_to_erd.py schema.sql --out ./erd --format all
```

解析：优先 `https://yanleaf.com/api/tools/er-diagram/parse`，失败本地 `sql_parser.py`。  
引擎：`--engine auto`（有 `dot` 用 Graphviz，否则 matplotlib）。

## 自然语言路径

1. 按规范设计 list-of-tables（snake_case、主键、外键 `xxx_id`、审计字段、3NF）。
2. 写入 JSON 后只渲染用户要的格式：

```bash
python scripts/render_chen.py ./tables.json ./erd.png yanleaf.com
```

回复中同样必须带 `MEDIA:./erd.png`（或等价附件字段）。只要 Word 则用 `export_word.py`，不要顺手再出 PNG。

## 关系与精简

- 外键所在表 = **N**，被引用表 = **1**（不要画反）
- 字段很多或用户说「只要关系」：保留 PK + FK + 少量关键字段（`--max-attrs`）
- 用户说重画：按新要求再生成，仍只出其需要的格式，PNG 仍用 `MEDIA:` 发出

## 依赖（Python 3.9+）

```bash
pip install sqlparse pyyaml python-docx pillow matplotlib graphviz
```

Graphviz 系统包可选；没有也能出 PNG。中文需系统字体或 `assets/` 内字体。

## 脚本

| 文件 | 作用 |
|------|------|
| `sql_to_erd.py` | SQL 入口（用 `--format` 控制产物） |
| `render_chen.py` | matplotlib Chen PNG |
| `api_client.py` | yanleaf parse API |
| `sql_parser.py` | 本地解析回退 |
| `generate_erd.py` | Mermaid / GoJS / Graphviz DOT |
| `export_word.py` | Word 三线表 |
| `watermark.py` | Graphviz PNG 水印 |

在线工具：https://yanleaf.com/tools/sql-to-er
