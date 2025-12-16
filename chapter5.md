# Chapter 5：把事实变成对话：多轮结构与标注方案

## 1. 开篇：跨越“图”与“话”的鸿沟

在 Chapter 3 和 4 中，我们学会了如何从 Wikidata 的海洋中捕捞高质量的“三元组（Triples）”。
比如我们抓到了：`[李白 (Q7078)] --P569(出生日期)--> [701年]`。

但是，**三元组不是对话**。
- 三元组是静态的、结构化的、高压缩的信息。
- 对话是动态的、流式的、包含冗余和上下文指代的过程。

本章的核心任务是构建一个**“对话模拟器”（Dialogue Simulator）**。你需要设计一套逻辑，让两个虚拟角色（User 和 Agent）在知识图谱上“游走”，并将游走的路径转化为自然语言。

**本章学习目标**：
1.  **对话动作建模**：掌握 8 种核心对话动作（从简单的属性查询到复杂的集合比较）。
2.  **图游走算法**：学习基于“实体栈（Entity Stack）”的上下文管理机制，实现深度的多跳对话。
3.  **黄金标准 Schema**：设计工业级的对话数据格式（JSONL），包含完整的意图（Intent）、槽位（Slot）和事实溯源（Grounding）。
4.  **多样性注入**：通过控制“信息密度”和“角色人设”，避免生成的对话千篇一律。

---

## 2. 核心论述：对话即图上的游走 (Dialogue as Graph Traversal)

想象知识图谱是一张巨大的地图，对话就是在这张地图上的旅行路线。

### 2.1 对话状态模型

一个多轮对话系统本质上维护着一个**状态（State）**。在基于 Wikidata 的生成中，状态主要由**当前焦点实体（Current Focus Entity）**决定。

```text
图谱结构：
[Q:诺兰] --(P:导演)--> [Q:星际穿越] --(P:配乐)--> [Q:汉斯·季默]
   |                       |
(P:出生地)              (P:主演)
   |                       |
[Q:伦敦]               [Q:马修·麦康纳]

对话流向（游走路径）：
Round 1: 聊诺兰 (Focus: 诺兰) -> 问出生地
Round 2: 聊他的作品 (Focus: 诺兰 -> Edge -> 星际穿越) -> 焦点转移到 [星际穿越]
Round 3: 聊这部电影的演员 (Focus: 星际穿越) -> 提到 [马修]
Round 4: 回溯 (Pop Stack) -> 聊这部电影的配乐 (Focus: 星际穿越 -> Edge -> 汉斯·季默)
```

为了实现自动化生成，我们需要模拟这种“焦点的转移”。

### 2.2 八大对话动作集 (The Extended Dialogue Action Set)

为了生成能够训练 LLM 处理复杂任务的数据，我们需要定义丰富的动作空间：

| 动作类型 (Intent) | 逻辑描述 | 示例 | 难度 |
| :--- | :--- | :--- | :--- |
| **1. Fact Retrieval** | `(S, P, ?)` <br> 查询单一事实 | User: "李白是哪年生的？"<br>Agent: "701年。" | ⭐ |
| **2. Boolean Verification** | `(S, P, O) -> True/False` <br> 验证事实真伪 | User: "李白是清朝人吗？"<br>Agent: "不，他是唐朝人。" | ⭐⭐ |
| **3. Entity Pivot (Drill-down)** | `Last_Obj` 变为 `New_Subj` <br> 深入话题 | User: "他写过什么诗？"<br>Agent: "《静夜思》。"<br>User: "**这首诗**表达了什么？" | ⭐⭐ |
| **4. Contextual Follow-up** | 省略主语 <br> `(Current_Subj, New_P, ?)` | User: "那**他**死在哪里了？"<br>Agent: "安徽当涂。" | ⭐⭐ |
| **5. Listing & Counting** | `Count(P)` 或 `List(P)` <br> 集合操作 | User: "列举三个欧盟成员国。"<br>Agent: "法国、德国、意大利..." | ⭐⭐⭐ |
| **6. Comparison** | `Compare(S1.P, S2.P)` <br> 属性对比 | User: "珠峰和K2谁更高？"<br>Agent: "珠峰更高，海拔8848米，而K2是8611米。" | ⭐⭐⭐ |
| **7. Temporal Constraint** | `Query(P) where time in range` <br> 时间切片 | User: "2010年的时候苹果CEO是谁？"<br>Agent: "当时是乔布斯（直到2011年）。" | ⭐⭐⭐⭐ |
| **8. Disambiguation** | `Clarify(Label)` <br> 歧义消除 | User: "讲讲李娜。"<br>Agent: "你想听网球冠军李娜，还是歌手李娜？" | ⭐⭐⭐⭐ |

---

## 3. 工程化实现：从数据到 Schema

生成数据的价值在于其**结构化标注**。如果只是生成文本，LLM 无法学习“检索”和“引用”。我们需要定义一个包含**思维链（Chain of Thought）**和**溯源（Provenance）**的格式。

### 3.1 推荐的数据结构 (JSONL)

这是一个生产级的 Schema 设计，包含了训练对话模型所需的所有元数据。

```json
{
  "conversation_id": "syn_wiki_bio_001",
  "domain": "biography",
  "seed_entity": {"qid": "Q7078", "label_zh": "李白"},
  "turns": [
    {
      "turn_id": 0,
      "role": "user",
      "text": "李白这辈子写了多少诗啊？",
      "intent": "count_property",
      "slots": {"entity": "李白", "property": "作品数量"}
    },
    {
      "turn_id": 1,
      "role": "assistant",
      "text": "李白一生创作了大量诗歌，现存确实的数量大约有1000多首。",
      "grounding": {
        "source": "wikidata",
        "triples": [
          {"s": "Q7078", "p": "P123", "o": "1000+", "reference_url": "..."} 
          // 注：此处仅为示例，Wikidata可能通过P800列举作品，需聚合计算
        ]
      }
    },
    {
      "turn_id": 2,
      "role": "user",
      "text": "那他最有名的代表作是什么？",
      "intent": "fact_retrieval_ranking",
      "context_dependency": "resolved_to:Q7078" // 标记指代消解
    },
    {
      "turn_id": 3,
      "role": "assistant",
      "text": "他最著名的作品包括《静夜思》、《望庐山瀑布》和《将进酒》等。",
      "grounding": {
        "triples": [
           {"s": "Q7078", "p": "P800", "o": "Q12345", "label": "静夜思"},
           {"s": "Q7078", "p": "P800", "o": "Q67890", "label": "将进酒"}
        ]
      },
      "api_call_simulation": "wiki_query(Q7078, P800, limit=3)" // 模拟工具调用
    },
    {
      "turn_id": 4,
      "role": "user",
      "text": "《静夜思》是什么时候写的？",
      "intent": "entity_pivot",
      "focus_shift": "Q7078 -> Q12345" // 焦点转移
    },
    {
      "turn_id": 5,
      "role": "assistant",
      "text": "关于《静夜思》具体的创作年份有争议，一般认为是他在726年创作的。",
      "grounding": {
         "triples": [
            {"s": "Q12345", "p": "P571", "o": "726", "qualifier_p": "P1480", "qualifier_o": "maybe"}
         ]
      }
    }
  ]
}
```

### 3.2 关键字段详解
- **`grounding` (溯源)**：这是最重要的字段。它告诉模型“这句话不是瞎编的，是来自这里”。这对减少幻觉至关重要。
- **`focus_shift` (焦点转移)**：记录对话焦点的变化路径，有助于训练模型的主题跟踪能力。
- **`qualifier` (限定符)**：如上例中的 `P1480` (sourcing circumstances) -> `maybe`，捕捉知识的不确定性。

---

## 4. 自动化生成策略：状态机算法

如何编写脚本自动生成上述数据？这里提供一个 **Rule-of-Thumb 算法**。

### 算法流程：
1.  **初始化**：
    *   `EntityStack = [Seed_Entity]`
    *   `History = []`
    *   `Current_Focus = Seed_Entity`
2.  **循环生成 (Loop N Turns)**：
    *   **User 动作选择**：掷骰子决定下一步动作。
        *   30% 概率：继续问 `Current_Focus` 的其他属性（Breadth）。
        *   40% 概率：从 `Current_Focus` 的关系中选一个 `Object`，作为新的 `Current_Focus` 并压栈（Depth/Pivot）。
        *   20% 概率：回退，`Pop Stack`，回到上一个实体（Return）。
        *   10% 概率：比较/列表/复杂操作。
    *   **Query 构造**：根据选定的 `(Subject, Property)` 在本地缓存或 Wikidata 查找。
    *   **模板渲染**：
        *   User 话术：使用多样化模板（"X是什么？", "告诉我X的信息", "X呢？"）。
        *   Assistant 话术：根据三元组类型（时间、地点、人物）选择回答模板，并注入 Label 和 Description。
    *   **记录**：将 Turn 保存到 JSONL，更新 History。

---

## 5. 本章小结

1.  **不仅仅是问答**：高质量对话数据必须包含追问、指代、焦点转移和逻辑判断，而不仅仅是“Q: A是谁？ A: ...”。
2.  **状态管理**：使用“实体栈”来模拟人类对话中的思维跳跃和回溯。
3.  **结构化优先**：生成的最终产物应该是包含丰富元数据的 JSON，而不是纯文本。文本只是表象，结构才是灵魂。
4.  **捕捉细微差别**：利用 Wikidata 的 Qualifier（限定符）来生成关于时间、角色和不确定性的精准回答。

---

## 6. 练习题

### 基础题 (Fundamentals)

<details>
<summary><strong>练习 1：构建追问逻辑</strong></summary>

**场景**：当前焦点是“周杰伦 (Q211241)”。
**任务**：
1.  第一轮：询问他的配偶（P26）。
2.  第二轮：不出现“周杰伦”和“昆凌”的名字，询问结婚时间（P580）。
3.  请写出第二轮 User 的 3 种不同自然语言问法。

**提示**：利用代词“他们”、“这段婚姻”等。

<br>
<details>
<summary><strong>参考答案</strong></summary>

1.  **事实获取**：Bot 回复“是昆凌。”
2.  **User 追问模板**：
    *   “他们是什么时候结婚的？”
    *   “这段婚姻开始于哪一年？”
    *   “在一起多久了？”（虽然需要计算，但意图是询问开始时间）
</details>
</details>

<details>
<summary><strong>练习 2：多值处理 (List Handling)</strong></summary>

**场景**：查询“泰勒·斯威夫特 (Q26876)”的“乐器 (P1303)”。Wikidata 返回了：吉他、钢琴、班卓琴、尤克里里。
**任务**：编写一个处理逻辑，如果返回值超过 3 个，Bot 该如何回答？请写出伪代码逻辑和最终回复文本。

<br>
<details>
<summary><strong>参考答案</strong></summary>

**逻辑**：
```python
items = get_labels(P1303)
if len(items) > 3:
    display_items = items[:3]
    response = f"她会演奏多种乐器，包括{', '.join(display_items)}等。"
else:
    response = f"她会演奏{', '.join(items)}。"
```
**文本**：“她会演奏多种乐器，包括吉他、钢琴、班卓琴等。”
</details>
</details>

### 挑战题 (Advanced)

<details>
<summary><strong>练习 3：设计“反事实”纠错数据</strong></summary>

**目标**：生成用于训练模型**拒绝回答**或**纠正错误**的数据。
**任务**：利用 Wikidata，设计一个生成流程，制造“张冠李戴”的用户问题，并生成正确的纠错回答。
**例子**：User: "奥巴马是哪一年发明电灯的？" -> Bot: "奥巴马并没有发明电灯，电灯主要是爱迪生改良推广的。"

**提示**：需要选取两个不相关的实体，或者错误的属性值。

<br>
<details>
<summary><strong>参考答案</strong></summary>

**生成策略**：
1.  **选取实体 A** (如：奥巴马 Q76)。
2.  **选取属性 P** (如：发明者 P61)。
3.  **选取实体 B** (如：电灯泡 Q64995)，确保 B 的 P 是 C (爱迪生)，且 C != A。
4.  **构造 User 话术**：将 A 强行作为 B 的 P 的主语。“[A] 发明了 [B] 吗？” 或 “[A] 是哪年发明 [B] 的？”
5.  **构造 Bot 话术**：
    *   Step 1: 否定 (False)。
    *   Step 2: 检索 B 的真实 P 值 (C)。
    *   Step 3: 生成“不是 A，其实是 C”。
</details>
</details>

<details>
<summary><strong>练习 4：时间限定符的推理</strong></summary>

**场景**：实体“德国 (Q183)”，属性“货币 (P38)”。
Wikidata 数据：
1.  德国马克 (Q16068) (end time: 2001-12-31)
2.  欧元 (Q4916) (start time: 2002-01-01)
**任务**：设计一个对话，User 询问“1990年德国用什么钱？”，Bot 必须根据限定符推理出正确答案。请写出 reasoning 过程。

<br>
<details>
<summary><strong>参考答案</strong></summary>

**Reasoning Process**:
1.  **Extract Constraint**: User query time $T_q = 1990$.
2.  **Retrieve Candidates**: Get all values for `Germany -> Currency`. Candidates: [Mark, Euro].
3.  **Filter**:
    *   Check Mark: No start time (assume valid from past), `end_time = 2001`. Since $1990 < 2001$, **Match**.
    *   Check Euro: `start_time = 2002`. Since $1990 < 2002$, **Mismatch**.
4.  **Generate**: "1990年时，德国使用的货币是德国马克（Deutsche Mark）。"
</details>
</details>

---

## 7. 常见陷阱与错误 (Gotchas)

### 1. "幻觉式"模板 (Template Hallucinations)
*   **现象**：模板写死为 `"{Name} 的{Prop}是 {Value}"`。
*   **问题**：当 Wikidata 数据缺失时，可能会生成 `"{Name} 的配偶是 None"` 或者空字符串。
*   **对策**：在填槽（Slot Filling）之前，**必须**检查 Value 是否存在。如果不存在，应该切换到“我不知道”或“数据缺失”的模板，或者直接放弃生成这一轮。

### 2. 忽略性别代词
*   **现象**：Bot 对女性实体称呼“他”，或对男性称呼“她”。
*   **原因**：中文生成时默认使用了“他”。
*   **对策**：查询实体时，顺便查询 `P21 (性别)`。
    *   Q6581097 (male) -> "他"
    *   Q6581072 (female) -> "她"
    *   其他/未知 -> "它" 或直呼其名。

### 3. 递归死循环
*   **现象**：Bot 和 User 陷入了无限循环：“A的父亲是B”，“B的儿子是A”，“A的父亲是B”...
*   **原因**：图游走算法没有记录 `Visited_Nodes`（已访问节点）。
*   **对策**：维护一个全局 `Visited Set`。在选择下一跳实体时，排除掉最近 3 轮内已经聊过的实体。

### 4. 值的单位与格式化
*   **现象**：User: "太阳有多重？" Bot: "1.989E30 公斤。"
*   **问题**：直接输出了科学计数法或原始数据，不符合人类直觉。
*   **对策**：
    *   对于大数字：编写 formatter，转为“1989亿亿亿”或保持科学计数法但解释。
    *   对于日期：将 `+1893-12-26T00:00:00Z` 转为 `1893年12月26日`。

### 5. 关系的方向性错误
*   **现象**：User: "谁是周杰伦的父亲？" Bot: "海瑟薇·罗密欧。" (这是他女儿)
*   **原因**：没有分清属性的方向。比如 `P40 (子女)` 是从父母指向子女。如果 User 问父亲，你需要反向查询，或者查询 `P22 (父亲)`。
*   **Rule of Thumb**：在写查询模板时，务必在 WDQS 上先验证属性的方向性。Wikidata 的属性名通常暗示了方向（如 "Child of" vs "Father"），但有时会有反向属性。
