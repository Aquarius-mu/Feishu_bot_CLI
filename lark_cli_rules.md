# lark-cli 使用规则（避坑指南）

处理任何飞书相关任务前，**必须先读取本文件**，再开始工作。

---

## 1. 常见命令参数正确写法

### 1.0 im +messages-send @mention 语法

**两种方式都支持真实 @，语法不同：**

#### 方式A（推荐）：卡片（interactive）+ schema 2.0 + div/lark_md
- `@mention` 用 `<at id="ou_xxx">名字</at>`（注意是 `id=` 不是 `user_id=`）
- 结构：`schema:2.0` + `body.elements` + `{tag:div, text:{tag:lark_md}}`

```bash
CONTENT=$(python3 -c "
import json
card = {
    'schema': '2.0',
    'config': {'wide_screen_mode': True},
    'header': {
        'title': {'tag': 'plain_text', 'content': '【{version} {环境}】已发好 🎉'},
        'template': 'wathet'
    },
    'body': {
        'elements': [
            {
                'tag': 'div',
                'text': {
                    'tag': 'lark_md',
                    'content': '<at id=\"ou_王同乐\">王同乐(Atlas)</at> <at id=\"ou_张超\">张超(Scott)</at>\n请知悉～'
                }
            },
            {'tag': 'hr'},
            {
                'tag': 'div',
                'text': {
                    'tag': 'lark_md',
                    'content': '<at id=\"ou_xxx\">负责人1</at> <at id=\"ou_yyy\">负责人2</at>\n麻烦配置活动，辛苦了！'
                }
            }
        ]
    }
}
print(json.dumps(card))
")
lark-cli im +messages-send \
  --chat-id oc_xxx \
  --msg-type interactive \
  --content "$CONTENT" \
  --as bot
```

#### 方式B：文本（text）格式 —— 用 `<at user_id="ou_xxx">`
```bash
CONTENT=$(python3 -c "
import json
print(json.dumps({'text': '<at user_id=\"ou_xxx\">名字</at> 消息内容'}))
")

lark-cli im +messages-send \
  --chat-id oc_xxx \
  --msg-type text \
  --content "\$CONTENT" \
  --as bot
```

**关键区别**：
- 卡片 `lark_md` 用 `<at id="ou_xxx">` ✅，用 `<at user_id="...">` 无效 ❌
- 文本消息用 `<at user_id="ou_xxx">` ✅
- `--text` 传 `@姓名` 是纯文本，无@效果 ❌
- `--content` 不能直接传非JSON字符串 ❌

### 1.1 im +chat-search
```bash
# 正确：关键词必须用 --query
lark-cli im +chat-search --query "MGame654.0" --as user

# 错误：不能用位置参数
lark-cli im +chat-search "MGame654.0"  # ❌
```

### 1.2 contact +search-user
```bash
# 正确：关键词用 --query，--as 是有效 flag
lark-cli contact +search-user --query "王同乐" --as user

# 错误：--keyword 不存在
lark-cli contact +search-user --keyword "王同乐"  # ❌
```

### 1.3 im chat.members get（获取群成员列表）
```bash
# 正确：必须用 --params 传入 path 参数
lark-cli im chat.members get --params '{"chat_id":"oc_xxx"}' --as user --page-all

# 错误：不能用 --chat-id flag
lark-cli im chat.members get --chat-id oc_xxx  # ❌
```

### 1.4 wiki spaces get_node（获取知识库节点信息）
```bash
# 正确
lark-cli wiki spaces get_node --params '{"token":"IU9ywLNvEi5WLDkEehAc0alwn5b"}' --as user

# 错误：wiki nodes get 不存在
lark-cli wiki nodes get IU9ywLNvEi5WLDkEehAc0alwn5b  # ❌
```

### 1.5 sheets +export 文件路径
```bash
# 正确：必须用相对路径，不能用 /tmp/ 等绝对路径
lark-cli sheets +export --spreadsheet-token xxx --file-extension xlsx --output-path ./output.xlsx

# 错误：绝对路径被拒绝
lark-cli sheets +export --output-path /tmp/file.xlsx  # ❌
```

---

## 2. 嵌入式 Bitable（多维表格块）访问流程

**核心规则**：当 `sheets +info` 返回 `resource_type: "bitable"` 的页签时，**不能**直接用 spreadsheet token 作为 base_token。必须走以下流程：

### 完整流程（3步）

**Step 1**：通过 metainfo API 获取 blockToken
```bash
lark-cli api GET /open-apis/sheets/v2/spreadsheets/{spreadsheetToken}/metainfo --as user
```
返回结果中找到目标 sheet（按 sheetId 或 title 匹配），取其 `blockInfo.blockToken`。

**Step 2**：解析 blockToken
```
blockToken 格式：{base_token}_{table_id}
例：BvXLbhpxPaybGNsfCTucV6VfnDb_tblshNr3dE1oWqU0
→ base_token = BvXLbhpxPaybGNsfCTucV6VfnDb
→ table_id   = tblshNr3dE1oWqU0
```

**Step 3**：用 base_token + table_id 读取数据
```bash
lark-cli base +record-list --base-token BvXLbhpxPaybGNsfCTucV6VfnDb \
  --table-id tblshNr3dE1oWqU0 --limit 100 --as user
```

### 错误路径（不要走）
- ❌ `lark-cli base +table-list --base-token {spreadsheet_token}` → 报 91402 NOTEXIST
- ❌ `lark-cli sheets +read --sheet-id {bitable_sheet_id}` → 报 sheetId not found
- ❌ `lark-cli base +record-list --base-token {sheet_id}` → 报 91402 NOTEXIST

---

## 3. Token 类型辨别速查

| 输入来源 | Token 用途 | 正确处理 |
|---------|-----------|---------|
| URL `/wiki/{token}` | wiki node token | `wiki spaces get_node --params '{"token":"..."}'` 取 `obj_token` |
| URL `/sheets/{token}` | spreadsheet token | 直接用于 `sheets` 命令；嵌入 bitable 须走 metainfo |
| URL `/base/{token}` | base_token | 直接用于 `base` 命令 |
| metainfo blockToken | `{base_token}_{table_id}` | 按 `_` 分割，前半段是 base_token，后半段（tbl 开头）是 table_id |

---

## 4. record-list 返回结构

`+record-list` 返回的数据结构：
```json
{
  "data": {
    "field_id_list": ["fldXxx", "fldYyy", ...],  // 字段顺序
    "fields": ["活动名", "负责人", ...],            // 字段名顺序（同上）
    "data": [[val0, val1, ...], ...],              // 每行按 field_id_list 顺序
    "has_more": false
  }
}
```

**注意**：`--field-id` 参数只能传**单个**字段 ID，不能传逗号分隔的多个。需要多字段时，不传 `--field-id`，拿全量再在本地按索引提取。

---

## 5. 执行效率原则

1. **先查缓存**：执行前先读 `~/feishu_bot/cache.json`，已有的群 ID、人员 open_id、bitable token 无需重复查询。缓存包含：
   - 版本发版群 chat_id（按版本号索引）
   - 王同乐、张超的 open_id
   - MLA配置确认表的 spreadsheet_token、wiki_token
   - 已解析过的各版本 bitable base_token + table_id
2. **并行查询**：群搜索、人员搜索、文档搜索互相独立，应一次性并行发出。
3. **schema 优先**：不确定命令参数时，先 `lark-cli schema <resource>.<method>` 查清楚，不要盲试。
4. **path 参数用 --params**：`location: "path"` 的参数一律通过 `--params '{"key":"val"}'` 传入，不要猜 flag 名。
5. **版本号映射**：版本号如 `654.0` 对应表格页签标题 `654`（取整数部分）。
6. **查完更新缓存**：每次解析到新的版本群 chat_id 或 bitable token，执行完后写入 `~/feishu_bot/cache.json`，供后续会话复用。
7. **发版通知只发一条消息**：bot 通过 `lark-cli im +messages-send --chat-id <群id>` 发通知，所有 @人（王同乐、张超、活动负责人）合并在同一条消息里，不拆成多条。读 bitable 只需提取负责人去重后的 open_id 列表，**不需要整理每人对应哪些活动**，消息中也不列活动明细，只 @ 人即可。
   消息固定格式（方案C）：
   ```
   【{version} {环境}】已发好 🎉
   @王同乐(Atlas) @张超(Scott) 请知悉～

   @活动负责人1 @活动负责人2 ...
   麻烦配置活动，辛苦了！
   ```
8. **发版环境识别**：用户说"xxx版本发好了"时，按以下规则解析环境类型，通知内容中体现对应环境：
   - `{version} 发好了` → 通用发版
   - `{version} 9999发好了` → 9999服
   - `{version} 测服发好了` → 测试服
   - `{version} 正式服发好了` → 正式服
   - 无论哪种环境，都需要 @活动负责人配置对应活动。

---

## 6. Python 构建卡片 JSON 规则

### 6.1 禁止用 `$(python3 -c "...")`

卡片内容含反引号（如代码块 ` ``` `）、特殊字符时，bash 会把反引号解释为命令替换，导致语法报错：

```
/bin/bash: command substitution: syntax error near unexpected token `"Hello"'
```

**正确做法**：用 heredoc 或写入临时文件：

```bash
# 方式A：heredoc（推荐）
python3 - <<'PYEOF'
import json
card = { ... }
print(json.dumps(card, ensure_ascii=False))
PYEOF
```

```bash
# 方式B：写文件，再 cat 读取
python3 /path/to/build_card.py > /tmp/card.json
CARD=$(cat /tmp/card.json)
lark-cli im +messages-send --chat-id oc_xxx --msg-type interactive --content "$CARD" --as bot
```

### 6.2 服务器 Python 版本是 3.6，不支持 `capture_output`

`subprocess.run()` 的 `capture_output=True` 是 Python 3.7+ 才有的参数，3.6 会报：

```
TypeError: __init__() got an unexpected keyword argument 'capture_output'
```

**正确写法**（3.6 兼容）：

```python
# 错误（3.7+）
result = subprocess.run(cmd, capture_output=True, text=True)

# 正确（3.6 兼容）
result = subprocess.run(cmd, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
output = result.stdout.decode()
```

> 实际调用 lark-cli 时，直接在 bash 里执行更简单，无需从 Python 调用 subprocess。

### 6.4 collapsible_panel header `background_color` 颜色枚举规则

支持带层级后缀的颜色枚举（`blue-100`、`green-100` 等），浅色推荐 `-50`～`-200`。
**`grey` 也可用**；不支持 RGBA 值；不支持无后缀的基础色名（`"blue"` 单用会报错 11310）。

```json
"header": {
  "title": {"tag": "plain_text", "content": "标题"},
  "background_color": "blue-100"
}
```

> :WARNING: 踩坑：颜色枚举报错 11310 时，先检查变量是否赋值正确（color/content 是否对调），再怀疑颜色值本身。

---

### 6.5 header icon token 必须用已验证的名称

`header.icon.token` 填了不存在的名字（如 `rocket-outlined`）时，图标静默不显示，不报错，但标题内容也会消失。

**安全做法**：标题里直接用 Unicode emoji（`🚀 标题`），效果等同且不依赖 token 名称。

**div 的 `icon` 字段同样不可靠**，`contacts-outlined` / `calendar-outlined` 等 token 实测也不显示。

统一规则：**所有图标都用 Unicode emoji 写在 lark_md content 里**，不依赖 `icon.token`。

已验证 header icon 可用的 token（仅供参考，其余一律用 emoji 替代）：`warning-outlined` · `arrow-down-outlined`

---

### 6.3 schema 2.0 不支持 `note` tag

使用 `note` 会报错：`cards of schema V2 no longer support this capability; unsupported tag note`（错误码 200861）

替代方案：

```json
{"tag": "div", "text": {"tag": "lark_md", "content": "<font color=grey>备注文字</font>"}}
```
