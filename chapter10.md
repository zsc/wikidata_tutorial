# Chapter 10：许可与合规：Wikidata CC0 与对话数据发布注意事项

## 1. 开篇段落

在前面的章节中，我们已经掌握了如何挖掘数据并生成自然的中文对话。现在，假设你手里已经拥有了一个包含 10 万条多轮对话的 JSONL 文件，准备上传到 Hugging Face，或者交付给企业客户用于训练垂直领域的 LLM。

在按下“发布”键之前，必须通过最后一关：**许可（Licensing）与合规（Compliance）**。

很多开发者误以为“Wikidata 是开源的”就意味着“毫无风险”。实际上，Wikidata 的 **CC0 (Creative Commons Zero)** 协议虽然放弃了版权，但在数据合成（Data Synthesis）场景下，你仍然面临**隐私权（GDPR）**、**人格权**、**协议传染（Viral Licensing）**以及**数据溯源**的挑战。本章将从法律边界到工程实践，教你如何构建一个既合规又专业的对话数据集。

## 2. 文字论述

### 2.1 深入理解 CC0：你拥有什么，不拥有什么

Wikidata 的核心数据采用 **CC0 1.0 Universal**。这是一种“放弃版权”的声明，意味着数据进入了**公有领域（Public Domain）**。

| 维度 | Wikidata CC0 允许你做的 | CC0 **不**覆盖（你需要小心的） |
| :--- | :--- | :--- |
| **复制与分发** | 可以自由复制整个数据库 | - |
| **修改** | 可以修改数据、格式、结构 | - |
| **商业用途** | 可以用于构建闭源商业软件 | - |
| **署名** | 法律上**不强制**署名（但强烈建议） | - |
| **隐私/人格权** | - | **活人数据**：受 GDPR 等隐私法保护。不能滥用名人肖像或隐私信息。 |
| **商标/专利** | - | **品牌名称**：可以用文字提到“Nike”，但不能通过生成数据假冒 Nike 官方客服。 |
| **外部引用** | - | **引用的描述文本**：如果某些描述来自 Wikipedia，则不再是 CC0。 |

### 2.2 致命陷阱：协议传染 (License Contamination)

这是构建对话数据最容易踩的雷区。Wikidata 本身是 CC0，但 Wikidata 经常引用 **Wikipedia（维基百科）** 的内容。

*   **Wikidata (数据)** = **CC0** (随便用)
*   **Wikipedia (文本)** = **CC-BY-SA 3.0/4.0** (署名-相同方式共享)

**场景复现**：
你在构造对话时，觉得只有三元组（实体-属性-值）太干巴，于是你读取了实体的 `schema:description`（描述）或者直接通过 API 抓取了对应 Wikipedia 页面的首段摘要（Abstract），拼接到 LLM 的 Prompt 中润色。

**后果**：
一旦你的数据集混入了 CC-BY-SA 的文本，根据“SA (Share-Alike)”条款，你的整个衍生数据集（Derived Dataset）可能被迫要求**必须开源**，且必须沿用 CC-BY-SA 协议。这对商业闭源模型是致命的。

```ascii
+---------------------+      +------------------------+
|   Wikidata Core     |      |   Wikipedia Snippets   |
| (Triples, Aliases)  |      | (Abstracts, Articles)  |
|       [CC0]         |      |     [CC-BY-SA]         |
+----------+----------+      +-----------+------------+
           |                             |
           v                             v
    +------+-----------------------------+------+
    |          你的对话生成流水线               |
    | (Data Synthesis Pipeline)                 |
    +--------------------+----------------------+
                         |
                         v
          +-----------------------------+
          |      最终对话数据集          |
          |  [?] 协议状态：被污染        |
          |  可能被迫变为 CC-BY-SA       |
          +-----------------------------+
```

**Rule-of-Thumb**：
> 商业数据集中，**严禁**直接混入 Wikipedia 的长文本摘要。仅使用 Wikidata 的 Label（标签）、Alias（别名）和 Property Value（属性值）。如果必须使用长文本，请使用 GPT 基于 CC0 的三元组重新生成（Rephrase），而不是直接复制粘贴。

### 2.3 工程实践：数据溯源 (Provenance) 设计

为了合规和调试，每条生成的对话都必须“有据可查”。不要只存对话文本，要存**元数据**。

建议的 JSONL 结构：

```json
{
  "id": "conv_1024",
  "dialogue": [
    {"role": "user", "content": "爱因斯坦是哪一年拿的诺贝尔奖？"},
    {"role": "assistant", "content": "他在1921年获得了诺贝尔物理学奖。"}
  ],
  "provenance": {
    "source_dataset": "Wikidata",
    "license": "CC0 1.0",
    "snapshot_date": "2023-11-20", 
    "entities": [
      {
        "qid": "Q937", 
        "label": "Albert Einstein",
        "veracity_check": "high"
      }
    ],
    "facts": [
      {
        "property_id": "P166",
        "property_label": "award received",
        "value_id": "Q38104",
        "value_label": "Nobel Prize in Physics",
        "qualifiers": {"P585": "1921"}
      }
    ]
  },
  "safety": {
    "is_living_person": false,
    "sensitive_topics": []
  }
}
```

### 2.4 数据卡 (Data Card) 规范

发布数据集时，必须附带 `DATA_CARD.md`。这不仅是 Hugging Face 的要求，也是行业规范。针对 Wikidata 衍生数据，重点填写以下部分：

#### 1. Dataset Description (数据集描述)
*   **Source**: Wikidata Query Service.
*   **Creation Method**: SPARQL sampling + Template/LLM Phrasing.
*   **Languages**: Chinese (zh-cn).

#### 2. Licensing Information (许可信息)
*   **Base License**: CC0 1.0 Universal.
*   **Disclaimer**: "While the underlying data is CC0, the natural language phrasing is machine-generated. Users should verify facts against original QIDs."

#### 3. Personal and Sensitive Information (个人敏感信息)
*   **Statement**: "Contains public figures extracted from Wikidata."
*   **GDPR Compliance**: 说明你是否过滤了非公众人物（例如：过滤掉了虽然有 QID 但维基百科链接数 < 3 的普通人）。

#### 4. Bias and Limitations (偏见与局限)
*   **Representation Bias**: 承认 Wikidata 本身存在的欧美中心主义（Eurocentrism）或性别偏差。
*   **Hallucinations**: 警告自然语言生成部分可能产生非事实性的连接词。

---

## 3. 本章小结

*   **CC0 是基石**：Wikidata 的三元组、标签、别名是安全的 CC0 资产，可商用。
*   **隔离 CC-BY-SA**：严防死守 Wikipedia 的摘要文本混入，除非你打算开源整个项目。
*   **元数据护体**：工程上必须保留 `QID`、`PID` 和快照时间，这是应对法律质疑和数据除错的最佳手段。
*   **隐私红线**：对于“在世人物”，应实施更严格的采样过滤策略（如仅限高知名度公众人物）。
*   **透明度**：通过详细的 Data Card 告知用户数据的来源、处理方法和潜在偏见。

---

## 4. 练习题

### 基础理解题

**练习 1：协议边界判断**
> 你正在为一个旅游 App 开发问答机器人。你从 Wikidata 下载了所有“旅游景点”的坐标（P625）和名称（Label），并从 Wikimedia Commons 下载了该景点对应的图片（P18）。
> **问题**：你能在不检查每张图片协议的情况下，直接在 App 里展示这些图片并收费吗？

<details>
<summary>点击展开答案</summary>

**答案：** **绝对不能。**
**提示：** 区分“引用图片的链接”和“图片文件本身”。
**解析：**
1.  Wikidata 中存储的“图片文件名”或“URL”这段文本是 CC0 的。
2.  但 URL 指向的**图片文件本身**存储在 Wikimedia Commons，其协议各不相同（从 CC0 到 CC-BY-SA 4.0 都有）。
3.  很多图片要求署名（Attribution），有些甚至禁止修改。如果你不检查每张图的 metadata 就商用，极大概率侵权。
</details>

**练习 2：敏感数据过滤**
> 你要生成一个关于“家庭关系”的对话集。在编写 SPARQL 查询时，为了遵守 GDPR 并降低隐私风险，你应该添加什么过滤条件？请至少列出两条。

<details>
<summary>点击展开答案</summary>

**答案：**
1.  **排除在世人物**：过滤掉没有 `P570`（死亡日期）的实体，或者计算年龄排除未成年人。
2.  **关注度过滤**：过滤掉 `wikibase:sitelinks`（维基百科页面链接数）较低的实体。例如 `FILTER(?sitelinks > 5)`。这样可以确保只包含“公众人物”，排除只被少量记录的普通人。
</details>

**练习 3：溯源字段设计**
> 为什么在保存对话数据时，建议保存 `snapshot_date`（快照日期）？

<details>
<summary>点击展开答案</summary>

**答案：**
Wikidata 是动态更新的。
1.  **事实变更**：某国总统今天换届了，如果你生成的数据说的是旧总统，有了快照日期，你就可以解释“这是基于 2023 年的数据生成的”，而不是模型出错了。
2.  **ID 复用/合并**：极少数情况下 QID 会被重定向或合并，快照日期有助于历史回溯。
</details>

### 进阶挑战题

**练习 4：商业化清洗场景**
> 某初创公司抓取了 Wikidata 所有数据，用 LLM 改写成了一本电子书《世界百科全书》并以 9.9 元出售。
> 1. 如果他们只用了三元组（QID/PID），合法吗？
> 2. 如果他们在 Prompt 里放入了 Wikidata 的 `description` 字段（其中部分由机器从 Wikipedia 自动导入），且没有人工审核，风险在哪里？

<details>
<summary>点击展开答案</summary>

**答案：**
1.  **合法**。三元组是 CC0，改写后的文本由 LLM 生成，版权归属（视当地法律）通常属于生成者或公有领域，但不侵犯 Wikidata 版权。
2.  **风险极高**。Wikidata 的 `description` 字段虽然大部分是短语，但有时会包含从 Wikipedia 导入的长句。如果这些描述构成了“独创性表达”且属于 CC-BY-SA，那么电子书可能违反了 Share-Alike 条款（即电子书必须免费/开源）。虽然概率较低（因为 description 通常很短），但在法律合规审查中这是一个瑕疵。
</details>

**练习 5：编写审计脚本**
> 编写一个伪代码或 Python 函数逻辑，用于检查你的输出数据集是否意外包含了“受保护的品牌词”作为误导性实体。
> *背景：Wikidata 中 Q1（宇宙）到 Q999999 都有。Q201445 是 "Nike"（公司）。*

<details>
<summary>点击展开答案</summary>

**提示：** 关注 P31 (instance of) 和 P1454 (legal form)。

**参考逻辑：**
```python
def safety_check(entity_data):
    # 1. 检查是否是商业组织
    # P31 (是) -> Q4830453 (商业企业) 或 Q6881511 (企业)
    is_business = check_property(entity_data, "P31", ["Q4830453", "Q6881511"])
    
    # 2. 检查意图
    # 如果生成的对话是 "我是 Nike 的客服，请问有什么帮您？" -> 违规（冒充）
    # 如果生成的对话是 "Nike 是一家成立在俄勒冈州的公司" -> 合规（陈述事实）
    
    if is_business:
        print(f"Warning: Entity {entity_data['qid']} is a business.")
        print("Ensure dialogue templates are strictly 3rd-person descriptive.")
        print("Do NOT use 1st-person 'I am...' templates for this entity.")
```
**解析：** CC0 允许你谈论商标持有人，但不允许你**冒充**商标持有人或在商业活动中引起混淆。
</details>

**练习 6：Data Card 的伦理声明**
> 你的数据集包含关于“药物（Q12140）”及其“副作用（P1909）”的对话。
> 请为 `DATA_CARD.md` 写一段“免责声明（Disclaimer）”和“预期用途（Intended Use）”。

<details>
<summary>点击展开答案</summary>

**参考答案：**
*   **Disclaimer**: "This dataset contains medical information derived from Wikidata. It is for **informational and NLP research purposes only** and does NOT constitute medical advice. Wikidata is a community-edited database and may contain errors. Do not use this model for clinical decision-making."
*   **Intended Use**: "Evaluating the ability of LLMs to structure biomedical knowledge graphs; Generating synthetic training data for entity extraction."
*   **Out-of-Scope Use**: "Building medical chatbots for patients without human-in-the-loop verification."
</details>

---

## 5. 常见陷阱与错误 (Gotchas)

### 🔴 陷阱 1：幽灵引用 (Ghost References)
**现象**：SPARQL 查询时未过滤 `wikibase:rank`。
**问题**：Wikidata 保留了历史数据。例如，某城市的人口数据可能有 1950年、1980年、2020年三条。如果你随机采样，可能会生成“北京的人口是 200 万”（这是 1949 年的数据）。
**对策**：总是优先选择 `PreferredRank`，或者在查询时按 `P585` (point in time) 倒序排列取最新值。

### 🔴 陷阱 2：外部 ID 的反向抓取
**现象**：Wikidata 包含大量 External IDs（如 IMDb ID, Twitter ID）。
**问题**：虽然 ID 本身是 CC0，但开发者写脚本去遍历这些 ID 爬取 IMDb 或 Twitter 的内容，并将这些内容混入数据集。
**后果**：这直接违反了 IMDb/Twitter 的 ToS（服务条款），法律风险远高于 Wikidata 本身。
**对策**：仅使用 Wikidata 内的数据。如果需要外部数据，必须单独处理其合规性，不要以为有了 Wikidata ID 就是“尚方宝剑”。

### 🔴 陷阱 3：Label 的语言回退误导
**现象**：使用 `SERVICE wikibase:label { bd:serviceParam wikibase:language "zh,en". }`
**问题**：如果中文 Label 缺失，WDQS 会自动回退到英文。如果你的生成模板是中文的，最后会生成中英夹杂的句子：“这座建筑位于 New York City。”
**后果**：虽然合规，但数据质量低劣，后续清洗困难。
**对策**：在代码层面明确检测 Label 的语言属性。如果获取到的是英文 Label，要么调用翻译 API（需注意翻译 API 的许可），要么丢弃该条数据。

### 🔴 陷阱 4：忽视 "人格权" (Right of Publicity)
**现象**：生成了大量虚拟对话，模拟某位已故名人（如玛丽莲·梦露）推荐某款现代商品。
**问题**：虽然照片可能不再受版权保护，但许多州/国家保护名人的“形象权”或“人格权”，禁止未经授权的商业代言关联。
**对策**：在数据生成 Prompt 中添加负面约束（Negative Constraints），禁止生成名人推荐商品类的内容，仅限于传记式问答。
