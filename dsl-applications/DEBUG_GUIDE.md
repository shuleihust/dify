# Dify DSL 调试手册（AI 内部参考）

## 🔴 最关键的坑（必看）

### Assigner节点必须用version 2格式

**❌ 错误写法（会导致conversation_id not found）**:
```yaml
- data:
    assigned_variable_selector: [conversation, stage]
    input_variable_selector: [code_node, output]
    type: assigner
```

**✅ 正确写法（必须这样写）**:
```yaml
- data:
    items:
    - input_type: variable
      operation: over-write
      value: [code_node, output]
      variable_selector: [conversation, stage]
      write_mode: over-write
    type: assigner
    version: '2'
  height: 86
```

**关键**: 所有assigner节点都必须有`version: '2'`和`items`数组结构，否则conversation变量无法更新！

---

## 快速参考

### DSL 导入（正确方法）

```python
import requests

with open('app.yml', 'r', encoding='utf-8') as f:
    yaml_content = f.read()

url = 'http://localhost:8080/console/api/apps/imports'

headers = {
    'Content-Type': 'application/json',
    'x-csrf-token': csrf_token  # 小写，从用户提供
}

cookies = {
    'access_token': access_token,
    'csrf_token': csrf_token
}

payload = {
    'mode': 'yaml-content',
    'yaml_content': yaml_content
}

response = requests.post(url, headers=headers, cookies=cookies, json=payload, timeout=30)
```

**关键点**：
- 必须同时提供 `headers` 和 `cookies`
- `x-csrf-token` 必须小写
- `access_token` 和 `csrf_token` 从用户的浏览器 cookie 中获取

---

## DSL 文件规范

### 必需字段

```yaml
app:
  description: '...'
  icon: 📚
  icon_background: '#FFF3E0'
  mode: advanced-chat  # 或 workflow
  name: 应用名称
  use_icon_as_answer_icon: false

dependencies: []
kind: app
version: 0.4.0  # 固定值，Dify 1.9.2

workflow:
  conversation_variables: []
  environment_variables: []
  features: {}
  graph:
    edges: []
    nodes: []
```

### 节点布局

```yaml
viewport:
  x: 0
  y: 0
  zoom: 0.8

nodes:
  - id: start_node
    position: {x: 80, y: 300}
  - id: middle_node
    position: {x: 380, y: 280}  # 横向间距 ~300
  - id: end_node
    position: {x: 680, y: 280}
```

### 变量引用格式

```yaml
prompt_template:
  - role: system
    text: |
      系统变量: {{#sys.query#}}
      会话变量: {{#conversation.stage#}}
      节点输出: {{#node_id.text#}}
```

### 会话变量定义

**⚠️ 重要：`id` 必须使用 UUID 格式**

```yaml
conversation_variables:
  - id: ab77671b-7e2c-40ce-82cd-5ddedf0bd308  # ✅ UUID 格式
    name: var_string
    value: ''          # string 用 ''
    value_type: string
    
  - id: a29ce65f-e4a8-4dcb-bb99-96c688df6264  # ✅ UUID 格式
    name: var_number
    value: 0           # number 用数字
    value_type: number
```

**❌ 错误示例**：
```yaml
conversation_variables:
  - id: stage        # ❌ 不能用普通字符串
    name: stage
```

**错误信息**：
```
sqlalchemy.exc.DataError: invalid input syntax for type uuid: "stage"
```

**生成 UUID**：
```python
import uuid
str(uuid.uuid4())  # 如 'ab77671b-7e2c-40ce-82cd-5ddedf0bd308'
```

---

## 模型配置

```yaml
model:
  name: deepseek-chat
  provider: deepseek
  mode: chat
  completion_params:
    temperature: 0.85
```

**常用提供商**：
- `deepseek` - DeepSeek
- `tongyi` - Qwen（通义千问）

---

## 节点类型

### LLM 节点（advanced-chat）

```yaml
- data:
    memory:
      query_prompt_template: '{{#sys.query#}}'
      window:
        enabled: true
        size: 10
    model:
      name: deepseek-chat
      provider: deepseek
      completion_params:
        temperature: 0.85
      mode: chat
    prompt_template:
      - role: system
        text: "..."
    type: llm
  id: llm_node
```

### Answer 节点

```yaml
- data:
    answer: '{{#llm_node.text#}}'
    type: answer
  id: answer_node
```

---

## 边连接

```yaml
edges:
  - id: start-to-llm
    source: start_node
    sourceHandle: source
    target: llm_node
    targetHandle: target
    type: custom
```

**检查**：source 和 target 必须是存在的节点 ID

---

## 导入后检查

成功响应：
```json
{
  "id": "导入ID",
  "status": "completed",  // 或 completed-with-warnings
  "app_id": "应用ID"
}
```

---

## 文件组织

```
dsl-applications/
└── app-name/
    ├── app-name.yml    # DSL 配置
    └── README.md       # 使用说明
```

---

## 检查清单

- [ ] version = 0.4.0
- [ ] 所有节点 ID 唯一
- [ ] edges 引用的节点都存在
- [ ] **会话变量 `id` 使用 UUID 格式（不能用普通字符串）**
- [ ] 变量 value 和 value_type 匹配
- [ ] 提示词变量引用格式正确 `{{#...#}}`
- [ ] 模型名称和提供商正确

---

---

## 已解决的问题记录

### 问题1: LLM提示词中包含提示语句导致重复输出

**现象**：
回复节点输出前，LLM已经输出了"✅ 世界观已完成！输入'继续'生成故事大纲。"

**原因**：
LLM节点的提示词末尾包含了提示语句，LLM会生成这些提示，然后answer节点又加了一次提示。

**解决方案**：
移除所有LLM提示词末尾的"最后提示"语句，只在answer节点中显示提示。

```yaml
# ❌ 错误
prompt_template:
  - role: system
    text: "...要求：逻辑自洽\n\n最后提示：\"✅ 完成！\""

# ✅ 正确
prompt_template:
  - role: system
    text: "...要求：逻辑自洽"
```

---

### 问题2: 输入"继续"后重复进入相同阶段

**现象**：
完成世界观生成后，输入"继续"，又重新进入世界观生成阶段。

**原因**：
`conversation.stage` 变量没有被更新，IF节点的条件判断一直匹配init阶段。

**解决方案**：
在每个阶段完成后，使用assigner节点更新stage变量：

```yaml
# 正确的节点流程
worldview_llm 
  -> code_worldview (返回 worldview + next_stage: "outline")
  -> assigner_worldview (保存 conversation.worldview)
  -> assigner_stage1 (更新 conversation.stage = "outline")
  -> answer_worldview

# Edges
- worldview_llm -> code_worldview
- code_worldview -> assigner_worldview
- assigner_worldview -> assigner_stage1
- assigner_stage1 -> answer_worldview
```

**Code节点示例**：
```python
def main(worldview_text: str) -> dict:
    return {
        "worldview": worldview_text,
        "next_stage": "outline"  # 下一个阶段的名称
    }
```

---

### 问题3: IF节点条件与code节点输出不匹配

**现象**：
修复了stage更新后，输入"继续"仍然进入错误的分支。

**原因**：
IF节点的case条件值与code节点返回的next_stage值不匹配。

```yaml
# ❌ 错误配置
# Code节点返回
next_stage: "outline"  # 世界观完成后

# 但IF节点判断
case-worldview:  # 查找 stage == "worldview"
  conditions:
    - value: worldview  # ❌ 匹配不上 "outline"
```

**解决方案**：
确保IF节点的case条件值与code节点返回的next_stage值一致：

```yaml
# ✅ 正确配置
# 流程设计
init → worldview_llm → stage="outline"
outline → outline_llm → stage="characters"
characters → characters_llm → stage="ready"
ready → summary_llm → 生成章节

# IF节点的case
- case-init: stage == "init" → worldview_llm
- case-outline: stage == "outline" → outline_llm
- case-characters: stage == "characters" → characters_llm
- case-ready: stage == "ready" → summary_llm

# Code节点的输出
worldview完成 → next_stage: "outline"
outline完成 → next_stage: "characters"
characters完成 → next_stage: "ready"
```

---

### 问题4: LLM无法正确处理"继续"等简单指令

**现象**：
修复了stage更新和IF条件匹配后，输入"继续"时，LLM生成的内容不符合预期或提示用户需要更多信息。

**原因**：
当用户输入"继续"时，`{{#sys.query#}}`的值就是"继续"。但LLM提示词没有说明如何处理这种简单指令，导致LLM不知道该如何响应。

**解决方案**：
在LLM提示词中添加指令处理说明：

```yaml
# ❌ 错误 - 没有说明如何处理"继续"
prompt_template:
  - role: system
    text: |
      **用户指令**：{{#sys.query#}}
      
      请基于世界观创建故事大纲...

# ✅ 正确 - 明确说明如何处理简单指令
prompt_template:
  - role: system
    text: |
      **用户指令**：{{#sys.query#}}
      
      现在请基于世界观创建故事大纲。如果用户只是输入"继续"、"下一步"等简单指令，
      请直接按照标准流程生成；如果用户提供了具体要求，请结合这些要求生成。
      
      请创建大纲...
```

**适用节点**：
- outline_llm（大纲生成）
- characters_llm（角色生成）
- 其他需要理解用户意图的LLM节点
  
---

### 问题5: 节点位置重叠导致界面显示异常

**现象**：
在Dify工作流编辑器中，部分节点看不到或重叠显示，无法清晰看到完整的工作流结构。

**原因**：
多个节点设置了相同的`position`坐标，导致节点在界面上叠加显示。

**案例**：
```yaml
# ❌ 错误 - outline_llm和code_outline位置重叠
- id: outline_llm
  position:
    x: 680
    y: 240

- id: code_outline
  position:
    x: 680  # ❌ 与outline_llm相同
    y: 240
```

**解决方案**：
为每个节点设置不同的x坐标，保持合理的横向间距（建议300px）：

```yaml
# ✅ 正确 - 节点水平排列，间距300px
- id: outline_llm
  position:
    x: 680
    y: 240

- id: code_outline
  position:
    x: 980  # 680 + 300
    y: 240

- id: assigner_outline
  position:
    x: 1280  # 980 + 300
    y: 240

- id: assigner_stage2
  position:
    x: 1580  # 1280 + 300
    y: 240
```

**布局建议**：
- **横向间距**：300px（节点之间清晰可辨）
- **纵向间距**：140-160px（不同流程分支）
- **起始位置**：x: 80（start节点）, x: 380（IF节点）, x: 680（第一个LLM节点）

---

### 问题6: Assigner节点格式错误导致conversation_id not found 🔴 **最关键的坑**

**严重程度**: 🔴 CRITICAL - 会导致整个workflow无法正常工作

**现象**：
```
Node assigner_stage1 failed to run
VariableOperatorNodeError: conversation_id not found
```

- stage变量无法更新，输入"继续"后一直停留在相同阶段
- 所有conversation变量都无法更新
- workflow看起来正常运行但变量状态始终不变

**根本原因**：
使用了错误的assigner节点格式。在Dify 1.9.2的advanced-chat模式下，assigner节点**必须**使用`version: '2'`和`items`数组结构。

**错误格式**：
```yaml
# ❌ 错误 - 旧版本格式
- data:
    assigned_variable_selector:
    - conversation
    - stage
    input_variable_selector:
    - code_worldview
    - next_stage
    output_type: string
    type: assigner
    write_mode: over-write
```

**正确格式**：
```yaml
# ✅ 正确 - version 2格式
- data:
    desc: 更新阶段为outline
    items:
    - input_type: variable
      operation: over-write
      value:
      - code_worldview
      - next_stage
      variable_selector:
      - conversation
      - stage
      write_mode: over-write
    type: assigner
    version: '2'
  height: 86  # 注意高度也要更新为86
```

**必需字段**：
- `version: '2'` - 必须指定版本2
- `items` - 数组结构，包含赋值操作
- `input_type: variable` - 输入类型
- `operation: over-write` - 操作类型
- `value` - 来源变量的选择器（数组）
- `variable_selector` - 目标变量的选择器（数组）
- `write_mode: over-write` - 写入模式

**适用场景**：
- advanced-chat模式下更新conversation变量
- workflow模式下更新变量
- 所有需要持久化变量的场景

---

### 问题7: Answer节点在变量递增后显示导致章节号错误

**现象**：
```
初稿显示："第1章完成"  ✅ 正确
润色显示："第2章完成"  ❌ 错误（应该也是第1章）
```

**原因**：
节点执行顺序错误，在显示最终结果之前就递增了章节号。

**错误流程**：
```
polish_llm（润色第1章）
  ↓
code_chapter（递增：current_chapter = 1 → 2）
  ↓
assigner_chapter（更新：current_chapter = 2）
  ↓
answer_final（显示"第{{current_chapter}}章" → "第2章"）← 错误！
```

**正确流程**：
```
polish_llm（润色第1章）
  ↓
answer_final（显示"第{{current_chapter}}章" → "第1章"）← 正确！
  ↓
code_chapter（递增：current_chapter = 1 → 2）
  ↓
assigner_chapter（更新：current_chapter = 2）
```

**修复方法**：
调整edges连接顺序，确保answer节点在变量更新节点之前执行：

```yaml
# ❌ 错误的边连接
- id: polish-to-code
  source: polish_llm
  target: code_chapter
- id: code-to-assigner
  source: code_chapter
  target: assigner_chapter
- id: assigner-to-answer
  source: assigner_chapter
  target: answer_final

# ✅ 正确的边连接
- id: polish-to-answer
  source: polish_llm
  target: answer_final        # 先显示结果
- id: answer-to-code
  source: answer_final
  target: code_chapter        # 再递增变量
- id: code-to-assigner
  source: code_chapter
  target: assigner_chapter
```

**关键原则**：
- 如果answer节点需要显示某个变量的"当前值"
- 确保该answer节点在变量更新节点**之前**执行
- 边的连接顺序决定了节点的执行顺序

---

### 问题8: 章节号从0开始而不是1

**现象**：
```
用户输入"生成第1章" → 系统显示"第0章"  ❌
第二次"继续" → 显示"第1章"
第三次"继续" → 显示"第2章"
```

**原因**：
conversation_variables 中 `current_chapter` 的初始值设置为 `0`。

**错误配置**：
```yaml
conversation_variables:
  - description: 当前章节编号
    id: a29ce65f-e4a8-4dcb-bb99-96c688df6264
    name: current_chapter
    value: 0  # ❌ 错误：从0开始
    value_type: number
```

**正确配置**：
```yaml
conversation_variables:
  - description: 当前章节编号
    id: a29ce65f-e4a8-4dcb-bb99-96c688df6264
    name: current_chapter
    value: 1  # ✅ 正确：从1开始
    value_type: number
```

**说明**：
- 章节编号通常从第1章开始，而不是第0章
- 初始值应该根据业务逻辑设置合理的起始值
- 数字类型的 conversation_variables 初始值要符合用户预期

---

**更新**: 2025-11-08
