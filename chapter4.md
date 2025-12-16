# Chapter 4：主题生成：从知识图谱采样“多样化主题池”

在构建对话数据（尤其是用于训练或评测 LLM 的数据）时，最致命的问题往往不是“数据量不够”，而是**“数据分布偏差”**。

如果你只是随机爬取 Wiki，你可能会得到 80% 的欧美流行文化（Head），15% 的二战历史，以及 5% 的其他内容。这种数据训练出的模型，在回答“非洲有哪些著名诗人”或“唐朝的税收制度”时会显得捉襟见肘。

本章将带你构建一个工业级的**主题采样器（Topic Sampler）**。我们将超越简单的 `SELECT *`，学习如何设计**分层采样（Stratified Sampling）**策略，在热度、地域、时间、领域等多个维度上实现数据的精确控制。

## 4.1 本章学习目标

1.  **构建主题分类树**：利用 Wikidata 的层级结构（Ontology），将业务需求映射为精确的查询路径。
2.  **热度分层（Head/Tail）**：量化实体“冷热度”，确保数据集既懂常识（Head）又有深度（Tail）。
3.  **多维均衡采样**：通过 SPARQL 动态参数，强制平衡地域（亚非拉 vs 欧美）、时间（古代 vs 现代）和性别/类型分布。
4.  **清洗与去重**：识别并剔除消歧义页、列表页、同质化数据及敏感内容。
5.  **工程化采样方案**：学习如何利用哈希随机（Hash Randomization）解决大数据集采样超时的问题。

---

## 4.2 核心概念：主题空间的设计

在写代码之前，我们需要定义“主题空间”。在 Wikidata 中，这通常意味着找到正确的 **根节点（Root QID）** 和 **属性路径（Property Path）**。

### 4.2.1 领域映射逻辑

不要直接平铺式地抓取。建议维护一个类似 YAML 的配置文件，定义你的采样目标：

```yaml
# topic_schema.yaml 示例
domains:
  - name: "世界文学"
    root_qid: "Q8261" # Novel (小说)
    # 策略：不仅要小说，还要文学作品(Q7725634)的子类
    path_strategy: "wdt:P31/wdt:P279*" 
    sampling_weight: 0.3 # 占总数据集 30%
    
  - name: "知名企业"
    root_qid: "Q4830453" # Business Enterprise
    path_strategy: "wdt:P31/wdt:P279*"
    sampling_weight: 0.2
```

### 4.2.2 关键路径：`P31` vs `P279`

理解这两个属性的区别是采样的基础：

*   **P31 (instance of / 是...的实例)**：实体 $\rightarrow$ 类别。例如：`《三体》 P31 小说`。
*   **P279 (subclass of / 是...的子类)**：类别 $\rightarrow$ 父类别。例如：`科幻小说 P279 小说`。

> **Rule of Thumb #1: 永远使用传递路径查询**
> 
> *   ❌ **新手写法**：`?item wdt:P31 wd:Q11424` (仅查找直接被标记为“电影”的项)。
> *   ✅ **专家写法**：`?item wdt:P31/wdt:P279* wd:Q11424` (查找是“电影”或“电影的任何子类”的项)。
>
> 理由：Wikidata 的分类非常细致。很多条目不会直接挂在“根节点”下，而是挂在叶子节点（如“定格动画电影”）下。

---

## 4.3 采样策略 A：头部与长尾 (Head vs. Tail)

LLM 需要兼顾“通用性”和“知识广度”。我们可以利用 **Sitelinks (维基百科链接数)** 作为实体热度的代理指标。

### 4.3.1 定义热度桶 (Buckets)

| 桶 (Bucket) | Sitelinks 数量 | 特点 | 采样目的 |
| :--- | :--- | :--- | :--- |
| **Head (头部)** | > 30 | 国际知名（如：爱因斯坦、巴黎） | 测试模型的基础常识跟随能力 |
| **Mid (腰部)** | 5 ~ 30 | 区域知名（如：某省会城市的市长、某知名游戏角色） | 测试特定领域的知识深度 |
| **Tail (长尾)** | 1 ~ 4 | 极冷门（如：18世纪的捷克诗人、某小行星） | 减少幻觉，测试知识边界 |

### 4.3.2 对应的 SPARQL 写法

我们通常不一次性查完，而是分批查询。

```sparql
# 查询：腰部实体 (Mid-tier) 的科幻电影
SELECT ?item ?itemLabel ?sitelinks WHERE {
  ?item wdt:P31/wdt:P279* wd:Q24925. # 科幻电影
  ?item wikibase:sitelinks ?sitelinks.
  
  # 核心逻辑：锁定腰部区间
  FILTER (?sitelinks >= 5 && ?sitelinks <= 30)
  
  SERVICE wikibase:label { bd:serviceParam wikibase:language "zh,en". }
}
# 随机取样将在后续处理，这里先 LIMIT 较大值
LIMIT 1000
```

---

## 4.4 采样策略 B：维度均衡 (Debiasing)

Wikidata 存在显著的 **西方中心主义 (Western Bias)**。如果你不加控制地采样“城市”，前 100 个结果可能 80% 在欧洲和北美。

我们需要在 SPARQL 层面强制进行 **分层 (Stratification)**。

### 4.4.1 地域均衡 (Geographical Balancing)

利用 `P30` (Continent) 或 `P17` (Country) 进行控制。

**糟糕的策略**：随机查 1000 个城市。
**优秀的策略**：分别查 200 个亚洲城市、200 个非洲城市...

```sparql
# 模板查询：获取特定大洲的作家
SELECT ?item ?itemLabel ?countryLabel WHERE {
  ?item wdt:P31 wd:Q5;            # 人类
        wdt:P106 wd:Q36180.       # 职业：作家
  
  # 关键链条：人 -> 国籍 -> 所属大洲
  ?item wdt:P27 ?country.
  ?country wdt:P30 ?continent.
  
  # 过滤器：在这里填入特定大洲的 QID
  # 亚洲: Q48, 非洲: Q15, 欧洲: Q46, 南美: Q18
  FILTER (?continent = wd:Q15) 
  
  SERVICE wikibase:label { bd:serviceParam wikibase:language "zh,en". }
}
LIMIT 100
```
*在 Python 脚本中，轮询 Q48, Q15, Q46... 拼接结果。*

### 4.4.2 时间均衡 (Temporal Balancing)

避免数据集中在 21 世纪。使用 `P569` (出生日期) 或 `P577` (出版日期/发生日期)。

> **Rule of Thumb #2: 并不是所有实体都有精确日期**
> 
> 不要强制 `FILTER (?date = "1990-01-01"^^xsd:dateTime)`。
> 最好使用 **范围查询 (Range Query)**，并允许一定的缺失值（如果不强求时间属性）。

```sparql
# 筛选 19 世纪 (1801-1900) 的画作
FILTER (?inception >= "1801-01-01"^^xsd:dateTime && ?inception <= "1900-12-31"^^xsd:dateTime)
```

---

## 4.5 采样策略 C：解决“随机查询”超时问题

### 4.5.1 `ORDER BY RAND()` 的陷阱

对包含数百万实体的类（如 `wd:Q5` 人类）执行 `ORDER BY RAND()` 会导致 WDQS 数据库全表扫描并排序，极大概率 **Timeout**。

### 4.5.2 解决方案：MD5 哈希采样法

利用实体的 QID 或 Label 生成哈希值，筛选特定哈希结尾的实体。这是在大数据集中实现“确定性随机”的最佳实践。

```sparql
# 高级技巧：随机采样 1/100 的人类数据，且不使用 RAND()
SELECT ?item ?itemLabel WHERE {
  ?item wdt:P31 wd:Q5. 
  
  # 将 QID 转换为字符串 ID (去掉 URL 前缀)
  BIND(STR(?item) AS ?itemStr)
  # 生成 MD5 哈希
  BIND(MD5(?itemStr) AS ?hash)
  
  # 筛选：只取哈希值以 "a1" 结尾的实体 (概率约 1/256)
  FILTER(REGEX(?hash, "a1$")) 
  
  SERVICE wikibase:label { bd:serviceParam wikibase:language "zh,en". }
}
LIMIT 100
```
*这种方法极快，且结果可复现。*

---

## 4.6 清洗与安全过滤

在数据进入对话生成环节前，必须进行清洗。

### 4.6.1 结构性噪音清洗

Wikidata 中有很多不是“实体”的条目，需要剔除。

```sparql
# 在查询中加入 MINUS 块
MINUS {
  ?item wdt:P31 wd:Q4167410.  # 排除：维基百科消歧义页
}
MINUS {
  ?item wdt:P31 wd:Q13406463. # 排除：维基百科列表条目 (List of...)
}
MINUS {
  ?item wdt:P31 wd:Q11266439. # 排除：维基百科模板 (Template:...)
}
```

### 4.6.2 敏感内容过滤 (NSFW/Safety)

对于对话模型，通常需要过滤色情、暴力或仇恨内容。我们可以维护一个**黑名单类列表**。

```sparql
# 排除特定类的实例及其子类
MINUS { ?item wdt:P31/wdt:P279* wd:Q12744. } # 色情片
MINUS { ?item wdt:P31/wdt:P279* wd:Q35245. } # 犯罪行为
```

---

## 4.7 典型流程图

构建主题池的完整流水线如下：

```ascii
[配置: topic_schema.yaml]
(定义领域, 权重, 过滤规则)
       |
       v
[查询生成器 (Python)] <-----> [黑名单 QID 池]
(拼接 SPARQL: 增加地域/时间/Sitelinks 约束)
       |
       v
[WDQS API 执行器] ----> (使用 MD5 哈希采样防超时)
       |
       v
[原始结果 (Raw JSON)]
       |
       v
[后处理与验证]
1. 去重 (QID Deduplication)
2. Label 检查 (必须有中文名? 或允许回退英文?)
3. 描述检查 (Description 是否包含 "disambiguation"?)
       |
       v
[最终主题池 (Topic Pool)] ===> 进入下一章：对话生成
```

---

## 4.8 本章小结

*   **路径**：使用 `wdt:P31/wdt:P279*` 抓取完整的主题树，不要只抓叶子。
*   **分层**：通过 `wikibase:sitelinks` 区分 Head/Mid/Tail，按比例混合以保证难度多样性。
*   **均衡**：不要把所有鸡蛋放在一个篮子里。在 Python 端循环查询不同的 `Continent` 和 `Time Period` 来打破数据偏见。
*   **性能**：避免在大类上使用 `ORDER BY RAND()`，改用 `MD5` 哈希后缀筛选。
*   **清洁**：显式排除 `Q4167410` (消歧义页) 和敏感类别。

---

## 4.9 练习题

### 基础题 (50%)

<details>
<summary><b>习题 4.1：基础的长尾筛选（点击展开）</b></summary>

**题目**：查找 **50 个** 类型为“电子游戏”（Q7889），且 Wikipedia 链接数在 **2 到 5 之间**（冷门但非孤立）的游戏。

**Hint**：结合 `wikibase:sitelinks` 和 `FILTER` 的逻辑与 (`&&`)。

<details>
<summary><b>参考答案</b></summary>

```sparql
SELECT ?game ?gameLabel ?sitelinks WHERE {
  ?game wdt:P31/wdt:P279* wd:Q7889. # 使用路径查询包含子类
  ?game wikibase:sitelinks ?sitelinks.
  
  FILTER(?sitelinks >= 2 && ?sitelinks <= 5)
  
  SERVICE wikibase:label { bd:serviceParam wikibase:language "zh,en". }
}
LIMIT 50
```
</details>
</details>

<details>
<summary><b>习题 4.2：时间切片采样（点击展开）</b></summary>

**题目**：查找 **10 位** 出生于 **1980年到1990年之间** 的 **网球运动员**（Q10833314）。

**Hint**：注意 `xsd:dateTime` 比较格式。

<details>
<summary><b>参考答案</b></summary>

```sparql
SELECT ?player ?playerLabel ?birthDate WHERE {
  ?player wdt:P31 wd:Q5;                # 是人类
          wdt:P106 wd:Q10833314;        # 职业是网球运动员
          wdt:P569 ?birthDate.          # 出生日期
          
  FILTER(?birthDate >= "1980-01-01"^^xsd:dateTime && 
         ?birthDate <= "1990-12-31"^^xsd:dateTime)
         
  SERVICE wikibase:label { bd:serviceParam wikibase:language "zh,en". }
}
LIMIT 10
```
</details>
</details>

<details>
<summary><b>习题 4.3：排除消歧义页（点击展开）</b></summary>

**题目**：查找 Label 包含 "Phoenix" 的所有条目，但**必须排除**消歧义页面（Q4167410）。

**Hint**：使用 `MINUS` 子句。

<details>
<summary><b>参考答案</b></summary>

```sparql
SELECT ?item ?itemLabel ?desc WHERE {
  ?item rdfs:label ?label.
  FILTER(CONTAINS(?label, "Phoenix"))
  
  # 排除消歧义页
  MINUS { ?item wdt:P31 wd:Q4167410. }
  
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en". }
}
LIMIT 20
```
</details>
</details>

---

### 挑战题 (50%)

<details>
<summary><b>习题 4.4：设计地域均衡的建筑查询（点击展开）</b></summary>

**题目**：请编写一个查询，寻找位于 **南美洲 (Q18)** 的 **摩天大楼 (Q11303)**。
**要求**：
1. 必须有中文标签。
2. 必须有图片（P18）。
3. 这是一个从属性链（Property Chain）推导位置的练习。

**Hint**：摩天大楼 -> P131 (行政区划) -> ... -> P17 (国家) -> P30 (洲)。路径可能不直接，推荐直接用 `wdt:P17/wdt:P30` 链式查询。

<details>
<summary><b>参考答案</b></summary>

```sparql
SELECT ?building ?buildingLabel ?countryLabel ?image WHERE {
  ?building wdt:P31/wdt:P279* wd:Q11303;  # 是摩天大楼或其子类
            wdt:P18 ?image.               # 有图片
  
  # 地理位置约束链：建筑 -> 国家 -> 洲
  ?building wdt:P17 ?country.
  ?country  wdt:P30 wd:Q18.               # Q18 = 南美洲
  
  # 强制要求有中文标签 (通过过滤 label 的语言 tag)
  ?building rdfs:label ?buildingLabel.
  FILTER(LANG(?buildingLabel) = "zh")
  
  SERVICE wikibase:label { bd:serviceParam wikibase:language "zh". }
}
LIMIT 20
```
</details>
</details>

<details>
<summary><b>习题 4.5：实现 MD5 哈希采样（点击展开）</b></summary>

**题目**：不使用 `ORDER BY RAND()`，请从数以万计的 **小行星 (Q3863)** 中随机抽取大约 10 个样本。假设哈希空间分布均匀。

**Hint**：`BIND(MD5(STR(?item)) AS ?hash)`。正则匹配哈希的最后几位。

<details>
<summary><b>参考答案</b></summary>

```sparql
SELECT ?asteroid ?asteroidLabel ?hash WHERE {
  ?asteroid wdt:P31 wd:Q3863.
  
  BIND(MD5(STR(?asteroid)) AS ?hash)
  
  # 16进制哈希，每位有16种可能。
  # 匹配最后3位为 "abc"，概率为 1/(16^3) = 1/4096
  # 如果小行星有 50万个，这会返回约 120 个。调整位数以控制数量。
  FILTER(REGEX(?hash, "abc$")) 
  
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en". }
}
LIMIT 10
```
</details>
</details>

<details>
<summary><b>习题 4.6：开放设计题——多轮对话的数据完备性检查（点击展开）</b></summary>

**题目**：假设你要生成一个关于“书籍推荐”的多轮对话，机器人需要知道书籍的：1. 作者 2. 出版日期 3. 类型 4. 主角。
请写一个查询，找出满足**所有**这4个条件（即属性都不为空）的书籍，并限制只查找 **中文标签存在** 的书籍。

**思考**：这样的数据在 Wikidata 中多吗？如果太少，在数据生成阶段该怎么做？（降级策略）

<details>
<summary><b>参考答案</b></summary>

```sparql
SELECT ?book ?bookLabel ?authorLabel ?date ?genreLabel ?characterLabel WHERE {
  ?book wdt:P31/wdt:P279* wd:Q571;  # 书籍
        wdt:P50 ?author;            # 作者
        wdt:P577 ?date;             # 出版日期
        wdt:P136 ?genre;            # 类型
        wdt:P674 ?character.        # 登场人物 (这个属性通常很稀疏！)
        
  # 强制中文标签存在
  ?book rdfs:label ?bookLabel.
  FILTER(LANG(?bookLabel) = "zh")
  
  SERVICE wikibase:label { bd:serviceParam wikibase:language "zh". }
}
LIMIT 50
```

**思考解答**：
同时满足这4个属性（特别是“登场人物 P674”）且有中文名的书籍，在 Wikidata 中属于**极其稀疏**的数据（可能只有哈利波特、红楼梦等头部书籍满足）。
**策略建议**：
在构造数据流水线时，不要用 `AND` 强约束所有属性。应该用 `OPTIONAL` 查询这些属性。如果某个属性缺失，生成的对话模板就变成：“抱歉，我不太清楚这本书的主角是谁，但我知道它的作者是...” —— **这也增加了对话数据的真实感（机器人不应该全知全能）。**
</details>
</details>

---

## 4.10 常见陷阱与错误 (Gotchas)

### 1. 递归查询的性能黑洞
**错误**：`?item wdt:P31* wd:Q5.` (查找人类及其所有子类实例)
**问题**：`*` 表示 0 次或多次。这会匹配到 `wd:Q5` 本身，甚至导致查询引擎遍历巨大的图谱路径。
**修正**：对于 `P31` (实例)，通常不需要递归，因为实体直连类。对于 `P279` (子类)，才需要 `*`。最稳妥写法：`?item wdt:P31/wdt:P279* wd:TargetClass`.

### 2. 忽略了多值属性 (Cardinality)
**场景**：查询电影的类型。
**问题**：一部电影可能是“科幻片”也是“动作片”。简单的 SELECT 会导致同一部电影出现多行数据（笛卡尔积）。
**修正**：在 Python 端处理聚合，或者在 SPARQL 中使用 `GROUP BY ?item` 和 `GROUP_CONCAT(?typeLabel; separator=", ")`。但在造数据初期，保留多行（explode）通常更方便后续处理。

### 3. Label Service 的“英文霸权”
**现象**：`bd:serviceParam wikibase:language "zh"`
**问题**：如果没有中文名，这一行数据就会展示为空白 label，或者直接被某些客户端丢弃。
**修正**：必须设置回退链：`"zh,zh-hans,zh-cn,en,fr,de"`. 优先展示中文，没有则展示英文，保证数据 ID 可见。

### 4. 混淆“国家”与“位置”
**问题**：筛选 `wdt:P17` (国家) = 中国。
**漏网之鱼**：很多历史人物（如李白）的国籍是“唐朝” (Q983898)，而不是“中华人民共和国” (Q148)。
**修正**：如果是查现代数据，用 P17；如果是查历史数据，需要包含历史政权 QID，或者使用 `P27` (国籍) 并接受多样化的值。

---

[< Chapter 3：WDQS 入门](chapter3.md) | [Chapter 5：把事实变成对话 >](chapter5.md)
