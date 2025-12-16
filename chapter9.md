# Chapter 9：多语言与中文缺失处理

在构建中文对话数据集时，Wikidata 的多语言特性既是最大的优势，也是最大的陷阱。理想情况下，我们希望每个实体都有完美的 `zh-cn`（大陆简体）标签。但现实是，数据分布极不均匀：头部实体中文资料详尽，长尾实体往往只有英文、德文甚至只有 QID。

如果处理不当，你的对话数据可能会出现以下“车祸现场”：
- **混合语言怪胎**：“Q12345 位于 Finland。”（ID泄漏 + 英文地名）
- **繁简混乱**：“这个程式的设计非常好。”（在大陆语境下使用了台湾术语）
- **生硬翻译**：“我是土耳其。”（将国家 Turkey 误译为火鸡）

本章将构建一套防御性的**多语言回退流水线（Language Fallback Pipeline）**，确保生成的对话在中文语境下自然、流畅且准确。

## 1. Wikidata 的语言代码迷宫

首先，你需要理解 Wikidata 混乱的语言标签系统。中文并不是只有一个 `zh` 代码。

| 语言代码 | 含义 | 典型特征 | 处理策略 |
| :--- | :--- | :--- | :--- |
| `zh` | 中文（通用） | 可能是简繁混杂，早期数据常标记为此 | 需要检测脚本并转码 |
| `zh-cn` | 大陆简体 | 我们最想要的目标 | **直接使用** |
| `zh-hans` | 简体中文（通用） | 与 `zh-cn` 极度接近 | **直接使用** |
| `zh-tw` | 台湾正体 | 繁体字，且包含地区词（如软体/软件） | **需 OpenCC 转换** |
| `zh-hk` | 香港繁体 | 繁体字，包含香港地区词 | **需 OpenCC 转换** |
| `zh-hant` | 繁体中文（通用） | 繁体字 | **需 OpenCC 转换** |
| `zh-sg` / `zh-my` | 新马简体 | 简体字，但在某些词汇上可能不同 | 视作简体使用 |

**核心原则**：你不能只请求 `zh`。你必须制定一个**优先级队列**。

---

## 2. 策略一：SPARQL 层面的精准获取

许多教程建议使用 `SERVICE wikibase:label` 魔法服务，这对于快速查看数据很有用，但**对于工程化数据生产是危险的**。因为你无法知道返回的字符串到底是原本的中文，还是系统自动回退的英文。

### 推荐做法：显式获取 + 语言标记

我们在查询时，应该把所有可能的中文变体以及英文都查出来，并带上语言标记，把决策逻辑留在 Python 端处理。

```sparql
SELECT ?item ?label_zh_cn ?label_zh_tw ?label_en ?sitelink_title WHERE {
  # 1. 你的核心查询逻辑
  ?item wdt:P31 wd:Q5. # 比如查人物

  # 2. 显式获取不同语言的 Label (利用 OPTIONAL)
  OPTIONAL { ?item rdfs:label ?label_zh_cn. FILTER(LANG(?label_zh_cn) = "zh-cn") }
  OPTIONAL { ?item rdfs:label ?label_zh_tw. FILTER(LANG(?label_zh_tw) = "zh-tw") }
  OPTIONAL { ?item rdfs:label ?label_en.    FILTER(LANG(?label_en) = "en") }

  # 3. 获取中文维基百科的标题 (Sitelink) - 这是关键备胎！
  OPTIONAL {
    ?sitelink schema:about ?item;
              schema:isPartOf <https://zh.wikipedia.org/>;
              schema:name ?sitelink_title.
  }
}
LIMIT 10
```

**为什么这样做？**
这给了你最大的灵活性。你可以在本地代码中决定：“如果 `zh-cn` 有，用它；如果没，看 `sitelink`；再没，看 `zh-tw` 转码；最后才考虑 `en`。”

---

## 3. 策略二：利用 Sitelinks（维基百科标题）

这是处理中文缺失的**“核武器”**。
很多时候，Wikidata 志愿者忘记填写 `label`，但维基百科上已经有成熟的中文条目了。

- **Entity**: `Q12345`
- **Label (zh)**: (Missing)
- **Sitelink (zhwiki)**: `绝命毒师`

**处理 Sitelink 的痛点：消歧义后缀**
维基百科标题为了唯一性，常带有括号。
- 原始标题：`李白 (诗人)`、`苹果 (消歧义)`、`Python (编程语言)`
- **清洗规则**：
  在生成对话前，必须使用正则表达式去除末尾的括号内容。
  
```python
import re

def clean_sitelink(title):
    if not title: return None
    # 去除末尾的括号及其内容，如 "Name (Type)" -> "Name"
    # 注意：这就处理了 99% 的情况，但要小心 "Alien (film)" 这种变成 "Alien" 后是否会有歧义
    return re.sub(r"\s*\(.*?\)$", "", title).strip()
```

---

## 4. 策略三：繁简与地区词转换 (OpenCC)

如果只有 `zh-tw` 标签（例如 "光年" vs "光年" 没区别，但 "程式" vs "程序" 区别很大），我们需要转换。

简单的字符映射（Character Mapping）是不够的，我们需要**词汇转换（Phrase Mapping）**。

### 工具推荐：OpenCC
不要自己写 replace，请使用 OpenCC（Python版 `opencc-python-reimplemented`）。

| 配置文件 | 功能 | 适用场景 |
| :--- | :--- | :--- |
| `t2s.json` | 繁体字符 -> 简体字符 | 仅字形转换，不改词汇。适合人名、古籍。 |
| `tw2sp.json` | 台湾正体 -> 大陆简体（含词汇） | **推荐默认使用**。会将“软体”转为“软件”，“计程车”转为“出租车”。 |

**Python 实现：**
```python
from opencc import OpenCC

# 初始化转换器 (台湾正体 -> 大陆简体)
cc = OpenCC('tw2sp')

def fallback_strategy(row):
    # 1. 优先 zh-cn
    if row.get('label_zh_cn'):
        return row['label_zh_cn']
    
    # 2. 其次 Sitelink (通常质量很高，且大部分是繁体/简体自动适应)
    if row.get('sitelink_title'):
        # 维基百科标题可能是繁体，建议也过一遍 OpenCC 以防万一
        return cc.convert(clean_sitelink(row['sitelink_title']))
    
    # 3. 再次 zh-tw (进行词汇转换)
    if row.get('label_zh_tw'):
        return cc.convert(row['label_zh_tw'])
        
    return None # 依然没有中文
```

---

## 5. 策略四：中英混排的边界（Code-Switching）

当所有中文手段都失效，只剩下英文 Label 时，我们该怎么办？
这取决于**实体类型**与**对话场景**。我们需要建立一个“可接受度”评分。

### 可接受保留英文的场景 (Acceptable)
1.  **软件/编程/技术栈**：`Python`, `TensorFlow`, `Linux`, `API`。
    *   *自然度*：“如何安装 TensorFlow？”（完美）
2.  **特定的品牌/型号**：`iPhone 15`, `PlayStation 5`。
    *   *自然度*：“我想买一台 PlayStation。”（自然）
3.  **学术/专有名词（无标准译名）**：某些只有拉丁学名的生物，或最新的论文概念。
4.  **著名的英文缩写**：`NASA`, `FIFA`, `WHO`。
    *   *技巧*：优先查找实体的 Alias 中全大写的词。

### 必须丢弃或强力翻译的场景 (Unacceptable)
1.  **自然地名**：`Smallville`。
    *   *怪异度*：“我想去 Smallville 旅游。”（像机翻）
2.  **历史人物/普通人名**：`John Smith`。
    *   *怪异度*：“你知道 John Smith 是谁吗？”（除非是英语教学场景，否则很怪）
3.  **普通名词/概念**：`Chair`, `Freedom`。
    *   *怪异度*：“这张 Chair 很舒服。”（只有特定ABC群体这样说话）

**工程决策**：
在数据采样的 `Where` 子句中，你可以通过过滤 `P31`（实例）来决定是否允许英文回退。例如，如果 `P31` 是 `wd:Q6256`（国家），则强制要求必须有中文；如果是 `wd:Q7397`（软件），则允许英文。

---

## 6. 进阶：利用 LLM 进行“上下文感知”补全

这是现代数据工程的最后一道防线。如果你必须提高数据覆盖率，可以使用 LLM 离线批量翻译。

**关键点：不要只扔一个单词给 LLM 翻译。**
Wikidata 里的 `Label` 脱离上下文会有歧义。
- 例子：`Turkey`
- 如果不给上下文，LLM 可能翻译成“火鸡”或“土耳其”。

**Prompt 模板**：
```text
Context: The entity has ID {qid}. 
Type: {type_label} (e.g., Country, Animal, or Food).
Description: {english_description}.
English Label: {english_label}.

Task: Provide the most standard Simplified Chinese name for this entity. 
If it is a person's name, use standard transliteration.
Output only the Chinese name.
```

---

## 7. 本章小结

1.  **回退链条（Fallback Chain）**：`zh-cn` > `Sitelink (Cleaned)` > `zh-tw (OpenCC)` > `zh` > `LLM Translation` > `English (特定领域)` > `DROP`。
2.  **Sitelink 价值连城**：它是被低估的高质量中文语料源，记得清洗括号。
3.  **OpenCC 必不可少**：使用 `tw2sp` 配置进行“词汇级”简繁转换。
4.  **类型决定策略**：根据实体的 P31 类型（人/地/物/事）决定是否接受英文 Label。

---

## 8. 练习题

### 基础题

1.  **OpenCC 实践**：
    安装 `opencc` (Python)，尝试转换字符串 `“他在使用软体看影片”`（台湾惯用语）。使用 `t2s.json` 和 `tw2sp.json` 分别转换，观察并记录结果的区别。

2.  **Sitelink 清洗正则**：
    编写一个 Python 函数，能够正确处理以下 Wikipedia 标题：
    - `Java (编程语言)` -> `Java`
    - `速度与激情 (电影)` -> `速度与激情`
    - `(Do Not Clean Me)` -> `(Do Not Clean Me)` (如果整个标题就在括号里，可能是特殊艺术作品，怎么处理？)

<details>
<summary>参考答案</summary>

1.  **OpenCC 实践**
    - `t2s.json`: "他在使用软体看影片" -> "他在使用软体看影片" (仅转了字，词汇没变，大陆读起来依然别扭)。
    - `tw2sp.json`: "他在使用软体看影片" -> "他在使用软件看视频" (词汇也转换了，符合大陆习惯)。

2.  **Sitelink 清洗正则**
    ```python
    import re
    def clean_title(text):
        # 匹配末尾的括号，且括号前允许有空格
        pattern = r"\s*\([^)]+\)$"
        # 只有当去除括号后还有内容时才去除（防止整个标题就是括号的情况）
        cleaned = re.sub(pattern, "", text)
        return cleaned if cleaned.strip() else text
    
    print(clean_title("(Do Not Clean Me)")) # 输出 (Do Not Clean Me)
    print(clean_title("Java (编程语言)"))   # 输出 Java
    ```
</details>

### 挑战题

3.  **构建完整的采样器类**：
    设计一个 Python 类 `EntityResolver`，输入一个包含 Wikidata 原始 SPARQL 结果（含多语言字段）的字典，输出最终用于对话的中文名称。
    要求：
    - 集成 OpenCC。
    - 集成 Sitelink 清洗。
    - 包含一个 `allow_english_types` 的白名单（如 Q7397 软件），如果在白名单内且无中文，返回英文；否则返回 None。

<details>
<summary>提示 (Hint)</summary>
你的类初始化时应该加载 OpenCC 模型。`resolve` 方法应该接收 row 和 type_qid。逻辑中包含：
`name = row['zh_cn']` -> `if not name: name = clean(row['sitelink'])` -> `if not name: name = cc.convert(row['zh_tw'])` -> `if not name and type_qid in allow_list: name = row['en']`。
</details>

---

## 9. 常见陷阱与错误 (Gotchas)

### 1. 错误的“通用中文”
Wikidata 早期的 `zh` 标签质量参差不齐。有时候 `zh` 标签里填的是繁体，甚至是日文汉字（Kanji）。
*   **调试技巧**：如果取到了 `zh` 标签，建议依然过一遍 `OpenCC` 的 `t2s`，确保统一为简体。

### 2. 别名 (Alias) 的陷阱
你可能发现某个实体没有中文 Label，但有中文 Alias。
*   **例子**：Label: None; Alias: `肯伊·威斯特`。
*   **Gotcha**：Wikidata 的 Alias 可能包含非常口语化甚至错误的叫法（黑粉的蔑称）。使用 Alias 替代 Label 时要小心，最好只在多轮对话的“同义改写”环节使用，不要在第一轮介绍时作为正式名称使用。

### 3. 多值 Sitelinks
极少数情况下，一个 Item 可能链接到多个中文维基项目（如 zhwiki, zh-yue-wiki 粤语, zh-min-nan-wiki 闽南语）。
*   **Gotcha**：SPARQL 查询时如果只写 `schema:inLanguage "zh"`，可能会导致笛卡尔积（一行变多行）。
*   **修复**：明确指定 `schema:isPartOf <https://zh.wikipedia.org/>`。

### 4. 描述 (Description) 的机翻腔
有些中文描述是 bot 批量刷进去的。
*   **特征**：`xx是xx` (缺少主语)，或者语序倒装。
*   **建议**：如果描述看起来很短或者包含典型的机翻错误，不如根据 `P31` (类型) 和 `P106` (职业) 使用模版自己生成描述（详见第7章）。
