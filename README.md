---
name: feishu-lark-cli
version: 1.0.0
description: "飞书 lark-cli 操作完整指南：消息/文档/表格/群组/用户/日历/任务，内嵌所有关键规则和典型流程。"
metadata:
  requires:
    bins: ["lark-cli", "jq", "python3"]
  cliHelp: "lark-cli --help"
---

# feishu-lark-cli

## 0. 使用前必读规则

### lark-cli 路径
```bash
LARK=/home/jiadongsun/nodejs/node22.15/bin/lark-cli
# 或直接用 lark-cli（已在 PATH）
```

### 身份选择
| 场景 | 身份 |
|------|------|
| 发消息到群/私聊 | `--as bot`（+messages-send **必须** bot） |
| 搜索群、搜索用户、读文档 | `--as user` |
| 获取群成员、读日历、创建任务 | `--as user` |
| 回复消息 | `--as bot` |

### 不熟悉命令时，先查 schema
```bash
lark-cli schema im.messages.create
lark-cli schema contact.users.get
```

### path 参数用 --params
```bash
# 正确
lark-cli im chat.members get --params '{"chat_id":"oc_xxx"}' --as user --page-all
# 错误 ❌
lark-cli im chat.members get --chat-id oc_xxx
```

---

## 1. @提及格式（关键，不同消息类型不同语法）

| 消息类型 | @语法 | 示例 |
|---------|-------|------|
| 卡片 `interactive` lark_md | `<at id="ou_xxx">姓名</at>` | `<at id="ou_abc">张三</at>` |
| 文本 `text` | `<at user_id="ou_xxx">姓名</at>` | `<at user_id="ou_abc">张三</at>` |
| `--text "姓名"` | 纯文本无@效果 ❌ | — |

### @全体
```python
# 卡片 lark_md
"<at id=\"all\">所有人</at>"
# 文本
'{"text": "<at user_id=\"all\">所有人</at> 内容"}'
```

---

## 2. 消息操作

### 发文本消息
```bash
CONTENT=$(python3 -c "import json; print(json.dumps({'text': '消息内容'}))")
lark-cli im +messages-send \
  --chat-id oc_xxx \
  --msg-type text \
  --content "$CONTENT" \
  --as bot
```

### 发卡片消息（interactive）
```bash
CARD=$(python3 -c "
import json
card = {
  'schema': '2.0',
  'header': {'title': {'tag': 'plain_text', 'content': '标题'}, 'template': 'blue'},
  'config': {'streaming_mode': False},
  'body': {'elements': [
    {'tag': 'markdown', 'content': '内容'}
  ]}
}
print(json.dumps(card))
")
lark-cli im +messages-send \
  --chat-id oc_xxx \
  --msg-type interactive \
  --content "$CARD" \
  --as bot
```

### 发卡片并@指定人
```bash
CARD=$(python3 -c "
import json
card = {
  'schema': '2.0',
  'header': {'title': {'tag': 'plain_text', 'content': '【通知】'}, 'template': 'green'},
  'body': {'elements': [
    {'tag': 'markdown', 'content': '<at id=\"ou_aaa\">张三</at> <at id=\"ou_bbb\">李四</at>\n请查收～'}
  ]}
}
print(json.dumps(card))
")
lark-cli im +messages-send --chat-id oc_xxx --msg-type interactive --content "$CARD" --as bot
```

### 回复消息
```bash
CONTENT=$(python3 -c "import json; print(json.dumps({'text': '回复内容'}))")
lark-cli im +messages-reply \
  --message-id om_xxx \
  --msg-type text \
  --content "$CONTENT" \
  --as bot
```

### 发私聊（按 open_id）
```bash
lark-cli im +messages-send \
  --user-id ou_xxx \
  --msg-type text \
  --content '{"text":"私聊内容"}' \
  --as bot
```

---

## 3. 文档/知识库操作

### 搜索知识库
```bash
lark-cli wiki +search --query "关键词" --as user
```

### 获取知识库节点信息（解析 obj_token）
```bash
# wiki URL: /wiki/{node_token}
lark-cli wiki spaces get_node \
  --params '{"token":"IU9ywLNvEi5WLDkEehAc0alwn5b"}' \
  --as user
# 返回 obj_token（即 spreadsheet_token 或 docx_token）
```

### 读飞书文档内容
```bash
lark-cli doc +content --document-token xxx --as user
```

---

## 4. 表格/多维表格操作

### 读 Spreadsheet sheet 列表
```bash
lark-cli sheets +info --spreadsheet-token xxx --as user
```

### 读 Spreadsheet 单元格
```bash
lark-cli sheets +read \
  --spreadsheet-token xxx \
  --sheet-id yyy \
  --range "A1:Z100" \
  --as user
```

### ⚠️ bitable 嵌入 spreadsheet 时的完整流程（三步，不可跳）

**Step 1**：获取 metainfo（找 blockToken）
```bash
lark-cli api GET /open-apis/sheets/v2/spreadsheets/{spreadsheetToken}/metainfo --as user
# 返回各 sheet 的 blockInfo.blockToken，格式：{base_token}_{table_id}
```

**Step 2**：解析 blockToken
```
BvXLbhpxPaybGNsfCTucV6VfnDb_tblshNr3dE1oWqU0
→ base_token = BvXLbhpxPaybGNsfCTucV6VfnDb
→ table_id   = tblshNr3dE1oWqU0
```

**Step 3**：读 bitable 记录
```bash
lark-cli base +record-list \
  --base-token BvXLbhpxPaybGNsfCTucV6VfnDb \
  --table-id tblshNr3dE1oWqU0 \
  --limit 100 \
  --as user
```

**禁止走的错误路径：**
```bash
lark-cli base +table-list --base-token {spreadsheet_token}  # ❌ 91402 NOTEXIST
lark-cli sheets +read --sheet-id {bitable_sheet_id}         # ❌ sheetId not found
lark-cli base +record-list --base-token {sheet_id}          # ❌ 91402 NOTEXIST
```

### record-list 返回结构
```json
{
  "data": {
    "fields": ["字段名1", "字段名2"],
    "data": [["值1", "值2"], ["值1", "值2"]],
    "has_more": false
  }
}
```
多字段时不传 `--field-id`（它只支持单个字段），拿全量本地提取。

### 写 bitable 记录
```bash
lark-cli base records create \
  --params '{"app_token":"xxx","table_id":"tblyyy"}' \
  --data '{"fields":{"字段名":"值"}}' \
  --as user
```

---

## 5. 群组操作

### 搜索群（查 chat_id）
```bash
lark-cli im +chat-search --query "群名关键词" --as user
```

### 获取群详情
```bash
lark-cli im +chats-get --chat-id oc_xxx --as bot
```

### 获取群成员列表（open_id + 姓名）
```bash
lark-cli im chat.members get \
  --params '{"chat_id":"oc_xxx"}' \
  --as user \
  --page-all
```

---

## 6. 用户操作

### 按姓名/邮箱/手机搜 open_id
```bash
lark-cli contact +search-user --query "张三" --as user
```

### 查用户详情（姓名/部门/邮箱）
```bash
lark-cli contact +get-user \
  --user-id ou_xxx \
  --user-id-type open_id \
  --as user \
  --jq '.data.user | {name,en_name,email,department_ids}'
```

---

## 7. 日历操作

### 查今日日程
```bash
lark-cli calendar +agenda --as user
```

### 创建日程
```bash
lark-cli calendar +event-create \
  --summary "会议名称" \
  --start "2026-04-17T10:00:00+08:00" \
  --end   "2026-04-17T11:00:00+08:00" \
  --as user
```

---

## 8. 任务操作

### 查我的任务
```bash
lark-cli task +get-my-tasks --as user
```

### 创建任务
```bash
lark-cli task +create \
  --summary "任务标题" \
  --due "2026-04-20T18:00:00+08:00" \
  --as user
```

---

## 9. 错误处理

返回 `code != 0` 时读 `msg` 字段告知用户，**不静默忽略**：
```bash
result=$(lark-cli im +messages-send ... 2>&1)
code=$(echo "$result" | jq -r '.code // 0')
if [[ "$code" != "0" ]]; then
  msg=$(echo "$result" | jq -r '.msg // "未知错误"')
  echo "Error $code: $msg"
fi
```

常见错误码：
| code | 含义 |
|------|------|
| 91402 | bitable/base token 不存在，检查是否用了错误的 token |
| 99991663 | access token 过期，重新 `lark-cli auth login` |
| 230002 | 没有操作权限，检查 --as bot/user |

---

## 10. 典型完整流程：知识库文档 → bitable → 发群通知

```bash
# Step 1: 搜知识库，找到 node_token
lark-cli wiki +search --query "MLA配置确认表" --as user | jq '.data.items[0]'

# Step 2: 解析 node_token → obj_token（spreadsheet_token）
NODE_TOKEN="IU9ywLNvEi5WLDkEehAc0alwn5b"
OBJ_TOKEN=$(lark-cli wiki spaces get_node \
  --params "{\"token\":\"$NODE_TOKEN\"}" --as user \
  --jq '.data.node.obj_token')

# Step 3: 看 sheet 列表，找嵌入的 bitable
lark-cli sheets +info --spreadsheet-token "$OBJ_TOKEN" --as user

# Step 4: 获取 metainfo → 提取 blockToken
METAINFO=$(lark-cli api GET /open-apis/sheets/v2/spreadsheets/${OBJ_TOKEN}/metainfo --as user)
BLOCK_TOKEN=$(echo "$METAINFO" | jq -r '.data.sheets[] | select(.title=="目标sheet名") | .blockInfo.blockToken')

# Step 5: 解析 base_token + table_id
BASE_TOKEN=${BLOCK_TOKEN%_*}
TABLE_ID=${BLOCK_TOKEN#*_}

# Step 6: 读 bitable 记录
lark-cli base +record-list \
  --base-token "$BASE_TOKEN" \
  --table-id "$TABLE_ID" \
  --limit 200 --as user

# Step 7: 整理负责人 open_id，构建卡片发群
CARD=$(python3 -c "
import json, sys
card = {
  'schema': '2.0',
  'header': {'title': {'tag': 'plain_text', 'content': '通知'}, 'template': 'green'},
  'body': {'elements': [{'tag': 'markdown', 'content': '<at id=\"ou_aaa\">张三</at> 请处理～'}]}
}
print(json.dumps(card))
")
lark-cli im +messages-send --chat-id oc_xxx --msg-type interactive --content "$CARD" --as bot
```
