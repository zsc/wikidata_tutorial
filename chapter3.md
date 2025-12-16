# Chapter 3：WDQS 入门：用 SPARQL 把知识“查出来”

## 1. 开篇段落

在构建对话系统数据流时，我们的核心痛点通常是：“我需要 10,000 条关于‘法国作家’的数据，每条都要有出生地、代表作和获奖记录，最好还得有中文名。”

手动检索是不可能的。我们需要一种能够批量、精准、结构化提取知识的工具。
**Wikidata Query Service (WDQS)** 就是这个强大的接口，而 **SPARQL** 则是我们要掌握的“咒语”。

不要被“编程语言”四个字吓倒。对于我们的用途（造数据），SPARQL 更像是在做**完形填空**：你画出一个数据的“形状”，Wikidata 就会把所有符合这个形状的数据填充给你。

**本章学习目标**：
1.  **工具掌握**：熟悉 WDQS 界面、辅助工具及数据导出格式（JSON/CSV）。
2.  **核心语法**：精通三元组匹配、`wdt:` 前缀的含义、以及中文标签服务。
3.  **关键逻辑**：掌握 `FILTER`（筛选）、`OPTIONAL`（可选匹配，对长尾数据极重要）、`MINUS`（排除）。
4.  **实战能力**：编写能够直接用于生产环境的采样查询语句。

---

## 2. 文字论述

### 2.1 WDQS 界面与工作流
访问 [query.wikidata.org](https://query.wikidata.org/)。

界面主要由三部分组成：
1.  **Query Editor**（代码区）：编写 SPARQL 的地方。支持 `Ctrl+Space` 自动补全（输入 "Cat" 会自动提示 `wd:Q146`）。
2.  **Query Helper**（左侧边栏）：如果不写代码，可以通过图形化界面点选属性（适合初学者探索属性 ID）。
3.  **Result View**（结果区）：展示查询表格。**关键功能**：右上角的 `Download` 按钮，支持导出 `JSON`（适合编程处理）和 `CSV`（适合人工查看）。

### 2.2 SPARQL 的本质：图谱匹配 (Graph Matching)

Wikidata 是一个**知识图谱 (Knowledge Graph)**。它的本质是一张巨大的网，节点是**实体 (Item)**，边是**属性 (Property)**。

SPARQL 查询本质上是在定义一个**子图结构**。你定义结构，数据库寻找匹配。

#### ASCII 图解：从思维到代码

假设我们要找：“所有出生在伦敦的摇滚乐手”。
这涉及两个事实：
1.  X 是一个摇滚乐手（职业 -> 摇滚乐手）。
2.  X 出生在伦敦（出生地 -> 伦敦）。

```text
      [变量 ?musician]
          |      \
          |       \  (关系2: wdt:P19 出生地)
          |        \
(关系1:    |         > [实体: wd:Q84 伦敦]
 wdt:P106 |
 职业)    |
          v
    [实体: wd:Q639669 摇滚乐手]
```

**代码映射**：
```sparql
SELECT ?musician WHERE {
  ?musician wdt:P106 wd:Q639669 .  # 边 1
  ?musician wdt:P19  wd:Q84 .      # 边 2
}
```

### 2.3 关键前缀：`wd:` vs `wdt:`

这是初学者最容易混淆的地方，但在造数据时必须分清。

*   **`wd:Qxxx` (Wikidata Entity)**: 代表一个具体的对象（如“鲁迅”、“北京”）。出现在三元组的**宾语**位置。
*   **`wdt:Pxxx` (Wikidata Truthy Property)**: 代表**直接真值**属性。
    *   **为什么叫“直接”？** Wikidata 的数据结构很深，一个事实可能包含“引用来源”、“生效时间”等。但在大多数对话生成场景中，我们只需要事实本身（例如：鲁迅的配偶是许广平），不需要知道这个事实是哪本书引用的。
    *   **Rule of Thumb**: 95% 的情况下，在 `WHERE` 子句的中间位置（谓语），你都应该用 `wdt:`。

### 2.4 “Label Service”：让数据说人话

SPARQL 原生返回的是 ID (`Q123`)。为了生成中文对话，我们必须把 ID 转换成文本。

**魔法代码块**（建议设置为输入法短语自动输入）：
```sparql
SERVICE wikibase:label { bd:serviceParam wikibase:language "[AUTO_LANGUAGE],zh,en". }
```
*   **解析**：这行代码告诉 Wikidata：“请尝试给我中文标签（zh），如果没有中文，就给我英文（en），如果都没有，就给我 ID。”
*   **用法**：只要在 SELECT 里加上 `?变量名Label`（驼峰式，必须以 Label 结尾），这一列就会自动出现。

### 2.5 处理“不完整”数据：`OPTIONAL`

在真实对话中，我们经常遇到这样的情况：有些作家有“笔名”，有些没有。
*   如果使用标准写法：`?writer wdt:P742 ?pseudonym.`
*   **后果**：数据库会**丢弃**所有没有笔名的作家！这会导致样本偏差（剩下的全是信息极度丰富的大名人）。

**解决方案**：使用 `OPTIONAL { ... }`。
```sparql
SELECT ?writer ?writerLabel ?pseudonym WHERE {
  ?writer wdt:P106 wd:Q36180 .        # 必须是作家
  OPTIONAL { ?writer wdt:P742 ?pseudonym . } # 如果有笔名就拿，没有就留空，不要丢弃行
  SERVICE wikibase:label { ... }
}
```
**数据启示**：使用 `OPTIONAL` 可以保证数据采样的**多样性**，避免只采样到“完美数据”。在后续处理 CSV/JSON 时，记得检查字段是否为空。

---

## 3. 本章小结

1.  **查询骨架**：
    ```sparql
    SELECT ?var ?varLabel WHERE {
      ?var wdt:属性ID wd:实体ID .
      SERVICE wikibase:label { bd:serviceParam wikibase:language "zh,en". }
    } LIMIT 100
    ```
2.  **逻辑连接**：
    *   **AND**: 写在同一 `WHERE` 块里的多行语句默认是“与”关系。
    *   **OR**: 使用 `UNION`（本章暂不展开，通常用多条查询替代）。
    *   **NOT**: 使用 `MINUS { ... }` 或 `FILTER NOT EXISTS { ... }`。
3.  **数据质量控制**：
    *   `OPTIONAL`: 保留缺失属性的数据。
    *   `FILTER`: 对数值、时间、字符串进行筛选。
    *   `DISTINCT`: 去重（防止一个人因为有两个职业出现两次）。

---

## 4. 练习题

> **做题建议**：请在 WDQS 编辑器中实际运行代码。观察结果数量和列的内容。

### 基础题 (50%)：熟悉语法与 Label

#### 练习 3.1：寻找“猫科动物” (Simple Lookup)
**场景**：我们要生成关于动物的闲聊数据。
**目标**：查询所有是“家猫 (Q146)”的**品种**（breeds）。
**提示**：
1. 实体：家猫 (Q146)。
2. 属性：这些品种是家猫的“子类 (subclass of, P279)”。
3. 记得加中文 Label。

<details>
<summary><b>点击查看答案</b></summary>

```sparql
SELECT ?breed ?breedLabel WHERE {
  ?breed wdt:P279 wd:Q146 .
  SERVICE wikibase:label { bd:serviceParam wikibase:language "zh,en". }
}
LIMIT 50
```
</details>

#### 练习 3.2：带时间限制的查询 (FILTER)
**场景**：生成历史题材对话，需要特定年代的人物。
**目标**：查找出生在 **1900年1月1日之后** 的物理学家。
**提示**：
1. 物理学家 (Q169470)，职业 (P106)。
2. 出生日期 (P569)。
3. 使用 `FILTER (?birthDate > "1900-01-01"^^xsd:dateTime)`。

<details>
<summary><b>点击查看答案</b></summary>

```sparql
SELECT ?physicist ?physicistLabel ?birthDate WHERE {
   ?physicist wdt:P106 wd:Q169470 ;   # 是物理学家
              wdt:P569 ?birthDate .   # 获取出生日期
   
   FILTER (?birthDate > "1900-01-01"^^xsd:dateTime) .

   SERVICE wikibase:label { bd:serviceParam wikibase:language "zh,en". }
}
LIMIT 20
```
</details>

#### 练习 3.3：谁娶了谁？(Bi-directional Relations)
**场景**：八卦/娱乐新闻对话数据。
**目标**：查找配偶 (P26) 也是“演员 (Q33999)”的人。
**提示**：
1. ?person 的配偶是 ?spouse。
2. ?spouse 的职业是 演员。
3. *进阶思考*：需要给 ?person 限制一个职业吗？（不限制就是全领域名人）。

<details>
<summary><b>点击查看答案</b></summary>

```sparql
SELECT ?person ?personLabel ?spouse ?spouseLabel WHERE {
  ?person wdt:P26 ?spouse .       # person 有个配偶叫 spouse
  ?spouse wdt:P106 wd:Q33999 .    # spouse 是个演员
  
  SERVICE wikibase:label { bd:serviceParam wikibase:language "zh,en". }
}
LIMIT 50
```
</details>

### 挑战题 (50%)：解决真实数据脏乱问题

#### 练习 3.4：可选信息采集 (OPTIONAL)
**场景**：生成人物卡片。有些信息（如推特账号）可能不存在，但我们希望尽可能收集，而不是因为没有推特就把这个人的所有信息都扔掉。
**目标**：查询“美国总统”，必须返回名字，**如果有**推特账号 (P2002) 则返回，没有则留空。
**提示**：
1. 美国总统：职务 (P39) -> 美国总统 (Q11696)。
2. 推特账号：P2002。
3. 将推特查询放入 `OPTIONAL { ... }`。

<details>
<summary><b>点击查看答案</b></summary>

```sparql
SELECT ?president ?presidentLabel ?twitter WHERE {
  ?president wdt:P39 wd:Q11696 .
  
  OPTIONAL { ?president wdt:P2002 ?twitter . }
  
  SERVICE wikibase:label { bd:serviceParam wikibase:language "zh,en". }
}
```
*观察结果：你会发现像华盛顿、林肯这样的总统也有数据行，只是 ?twitter 列是空的。如果不加 OPTIONAL，他们会被直接过滤掉。*
</details>

#### 练习 3.5：排除特定数据 (MINUS)
**场景**：我们需要生成“非虚构”类的对话数据，需要排除神话传说或虚构角色。
**目标**：查找“著名的熊”，但排除“虚构的熊”。
**提示**：
1. 熊 (Q1175)。
2. 虚构角色 (Q95074)。
3. 使用 `MINUS { ?item wdt:P31 wd:Q95074 }`。

<details>
<summary><b>点击查看答案</b></summary>

```sparql
SELECT ?bear ?bearLabel WHERE {
  ?bear wdt:P31 wd:Q1175 .
  
  MINUS { ?bear wdt:P31 wd:Q95074 . } # 排除虚构角色(如维尼熊)
  
  SERVICE wikibase:label { bd:serviceParam wikibase:language "zh,en". }
}
LIMIT 20
```
*注：如果不加 MINUS，你会查到“帕丁顿熊”。加了之后，你应该主要看到动物园里的名熊（如克努特）。*
</details>

#### 练习 3.6：统计与聚合 (COUNT & GROUP BY)
**场景**：在开始大规模爬取前，我们需要评估数据量。比如，我想知道哪个国家的“博物馆”数据最丰富？
**目标**：统计每个国家 (P17) 拥有的博物馆 (Q33506) 数量，按数量降序排列。
**提示**：
1. 查找所有博物馆及其国家。
2. 使用 `GROUP BY ?country ?countryLabel`。
3. 使用 `COUNT(?museum) as ?count`。
4. `ORDER BY DESC(?count)`。

<details>
<summary><b>点击查看答案</b></summary>

```sparql
SELECT ?country ?countryLabel (COUNT(?museum) AS ?count) WHERE {
  ?museum wdt:P31 wd:Q33506 ;
          wdt:P17 ?country .
  
  SERVICE wikibase:label { bd:serviceParam wikibase:language "zh,en". }
}
GROUP BY ?country ?countryLabel
ORDER BY DESC(?count)
LIMIT 20
```
*数据洞察：这个查询能帮你决定后续采样策略。如果某个国家数据太少，你可能需要在 prompt 中做特殊处理。*
</details>

---

## 5. 常见陷阱与错误 (Gotchas)

#### 1. 笛卡尔积爆炸 (Cartesian Product Explosion)
这是最危险的错误。
*   **场景**：你想查某人的“所有职业”和“所有孩子”。
    ```sparql
    ?p wdt:P106 ?job .
    ?p wdt:P40 ?child .
    ```
*   **后果**：如果某人有 3 个职业，4 个孩子，结果会返回 $3 \times 4 = 12$ 行！数据会产生冗余组合。
*   **解决**：
    *   方法一：分两次查询（先查职业，再查孩子）。
    *   方法二：使用 `GROUP_CONCAT` 将多个值合并成一个字符串（如 "Singer, Actor"）。

#### 2. Label Service 导致的超时
虽然 Label Service 很方便，但它非常消耗性能。如果你查询的数据量极大（>10万行），启用 Label Service 可能会导致 `Timeout`。
*   **技巧**：在探索阶段使用 Label Service。在编写自动化爬虫脚本（Python）时，建议**不使用 Label Service**，只取 QID，然后通过本地词表或单独的 API 批量获取 Label，这样速度最快且最稳定。

#### 3. 语言回退的“坑”
`wikibase:language "zh,en"` 的意思是“先找简体中文，没有就找英文”。
*   **陷阱**：有时候你拿到了英文 Label，但你的对话模板是中文的。这会生成“著名的作家 William Shakespeare 出生于...”这样的混合语料。
*   **处理**：在后处理阶段（Python），需要检测 Label 里的字符是否包含中文字符，如果全是英文，需要决定是丢弃、翻译还是保留。

#### 4. `LIMIT` 的位置
`LIMIT` 必须放在查询的最后。
*   ❌ `LIMIT 10 SELECT ...`
*   ✅ `SELECT ... LIMIT 10`

---
[< Chapter 2：数据模型](chapter2.md) | [Chapter 4：主题生成 >](chapter4.md)
