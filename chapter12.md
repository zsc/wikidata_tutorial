# Chapter 12：查询模板库与速查表 (Appendices)

## 1. 本章概览

在构建对话数据流水线时，SPARQL 查询是获取原材料的第一步。很多时候，你不需要精通 SPARQL 的每一个语法细节，只需要拥有一个**高质量的代码片段库**（Snippets）。

本章不仅仅是简单的语法参考，而是针对**对话生成场景**整理的“武器库”。我们重点解决了以下工程难题：
- **查全率**：如何找到描述一个实体的所有关键属性？
- **多样性**：如何避免只抓取到“头部热门数据”？
- **精准度**：如何利用限定符（Qualifiers）生成“在2008年获得奥斯卡奖”这样精确的句子，而不是笼统的“获得过奥斯卡”。

**使用建议**：
建议将本章提到的 SPARQL 模板保存为 `.rq` 文件，或整理进你的 Python `QueryBuilder` 类中。

---

## 2. 基础配置与前缀 (Standard Prefixes)

在 Python 脚本（如 `SPARQLWrapper`）中发送请求时，必须包含这些前缀。

```sparql
PREFIX wd: <http://www.wikidata.org/entity/>          # 实体 (Item), e.g., wd:Q5
PREFIX wdt: <http://www.wikidata.org/prop/direct/>    # 真值属性 (Truthy), e.g., wdt:P31
PREFIX wikibase: <http://wikiba.se/ontology#>         # 维基基础本体
PREFIX p: <http://www.wikidata.org/prop/>             # 声明节点 (Statement Node)
PREFIX ps: <http://www.wikidata.org/prop/statement/>  # 声明的具体值
PREFIX pq: <http://www.wikidata.org/prop/qualifier/>  # 限定符 (Qualifier)
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>  # 标签用
PREFIX bd: <http://www.bigdata.com/rdf#>              # BigData 扩展 (Label Service)
PREFIX schema: <http://schema.org/>                   # 用于描述/别名
PREFIX skos: <http://www.w3.org/2004/02/skos/core#>   # 用于别名 (altLabel)
```

---

## 3. ID 速查表 (The "Big Map")

在生成对话时，你需要根据 PID 的数据类型来决定生成的句式（例如：时间类型用“发生在...”，地点类型用“位于...”）。

### 3.1 核心元数据 (Core Metadata)
*用于过滤数据质量和基本类型。*

| 属性/类型 | ID | 说明 | 对话生成用途 |
| :--- | :--- | :--- | :--- |
| **Instance of** | `P31` | 实例归属 | 核心过滤器，决定话题是“人”还是“山”。 |
| **Subclass of** | `P279` | 子类归属 | 用于扩充话题，如 `P279* wd:Q11424` (所有类型的电影)。 |
| **Image** | `P18` | 图像 | 多模态对话素材。 |
| **Sitelinks** | `wikibase:sitelinks`| 维基链接数 | **极其重要**，用于衡量“热度”，过滤冷门噪声。 |
| **Disambiguation**| `Q4167410` | 消歧页 | **黑名单**，必须在 `MINUS { ?item wdt:P31 wd:Q4167410 }` 中排除。 |

### 3.2 人物与生物 (People & Biographies)
*用于生成传记类、八卦类、关系类对话。*

| 属性名 | ID | 数据类型 | 典型值示例 |
| :--- | :--- | :--- | :--- |
| **性别** | `P21` | Item | 男性/女性/非二元 |
| **出生日期** | `P569` | Time | 1980-01-01 |
| **死亡日期** | `P570` | Time | 2023-10-27 |
| **出生地** | `P19` | Item | 纽约 |
| **国籍** | `P27` | Item | 中国 |
| **职业** | `P106` | Item | 政治家、演员 |
| **配偶** | `P26` | Item | (另一个人物QID) |
| **父亲/母亲** | `P22`/`P25` | Item | (父母QID) |
| **获知奖项** | `P166` | Item | 诺贝尔物理学奖 |
| **学历/母校** | `P69` | Item | 哈佛大学 |

### 3.3 地理与行政区 (Geography & Places)
*用于生成旅游助手、地理问答对话。*

| 属性名 | ID | 数据类型 | 典型值示例 |
| :--- | :--- | :--- | :--- |
| **所属国家** | `P17` | Item | 必须校验的属性，避免“巴黎在中国”的错误生成。 |
| **坐标** | `P625` | GlobeCoord | Point(121.4 31.2) |
| **人口** | `P1082` | Quantity | 24,000,000 |
| **面积** | `P2046` | Quantity | 6000 km² |
| **下辖地区** | `P150` | Item | (包含的行政区) |
| **毗邻** | `P47` | Item | (接壤的国家/省份) |
| **最高点** | `P610` | Item | 珠穆朗玛峰 |

### 3.4 文化与作品 (Arts, Media & Works)
*用于生成推荐系统、影评、书籍讨论对话。*

| 属性名 | ID | 数据类型 | 典型值示例 |
| :--- | :--- | :--- | :--- |
| **创作者** | `P170` | Item | (艺术家) |
| **作者** | `P50` | Item | (作家) |
| **导演** | `P57` | Item | (导演) |
| **演员成员** | `P161` | Item | (演员列表) |
| **出版日期** | `P577` | Time | 2000-01-01 |
| **类型(Genre)**| `P136` | Item | 科幻、流行音乐 |
| **片长** | `P2047` | Quantity | 120 分钟 |
| **制作商** | `P272` | Item | 华纳兄弟 |

---

## 4. 场景化查询模板库 (Scenario-Based Templates)

这里将查询按“对话功能”分类。你可以直接复制使用。

### 场景 A：生成“实体百科介绍” (Profile Generation)
**目标**：获取关于一个实体的全方位信息，用于生成“介绍一下 X”的回复。
**策略**：一次性拉取多个属性，使用 `OPTIONAL` 防止因某个属性缺失导致整条数据被丢弃。

```sparql
SELECT ?item ?itemLabel ?desc ?birthDate ?birthPlaceLabel ?jobLabel WHERE {
  # 1. 定义范围：中国作家，且按热度排序前 100
  ?item wdt:P31 wd:Q5;
        wdt:P106 wd:Q36180;
        wdt:P27 wd:Q148.
  ?item wikibase:sitelinks ?sitelinks.

  # 2. 可选属性：即使没有这些属性，也返回实体
  OPTIONAL { ?item wdt:P569 ?birthDate. }
  OPTIONAL { ?item wdt:P19 ?birthPlace. }
  OPTIONAL { ?item wdt:P106 ?job. }
  
  # 3. 获取描述和别名
  OPTIONAL { ?item schema:description ?desc. FILTER(LANG(?desc) = "zh") }

  # 4. 标签服务
  SERVICE wikibase:label { 
    bd:serviceParam wikibase:language "zh,en". 
    ?item rdfs:label ?itemLabel.
    ?birthPlace rdfs:label ?birthPlaceLabel.
    ?job rdfs:label ?jobLabel.
  }
}
ORDER BY DESC(?sitelinks)
LIMIT 100
```

### 场景 B：生成“列表与推荐” (List & Recommendation)
**目标**：生成类似“列举几部张艺谋导演的电影”的数据。
**关键点**：从“导演”实体反向查询“电影”。

```sparql
SELECT ?directorLabel ?movieLabel ?year WHERE {
  # 绑定导演：张艺谋 (Q55430)
  BIND(wd:Q55430 AS ?director)

  # 反向查询：找电影，其 P57 (导演) 是张艺谋
  ?movie wdt:P31 wd:Q11424;
         wdt:P57 ?director;
         wdt:P577 ?pubDate.
         
  # 提取年份
  BIND(YEAR(?pubDate) AS ?year)

  SERVICE wikibase:label { bd:serviceParam wikibase:language "zh,en". }
}
ORDER BY DESC(?year) # 按年份倒序
LIMIT 20
```

### 场景 C：生成“复杂关系与限定符” (Qualifiers & Context)
**目标**：生成包含时间状语的复杂句，如“奥巴马从2009年到2017年担任美国总统”。
**关键点**：使用 `p:` 进入声明节点，使用 `pq:` 提取时间。

```sparql
SELECT ?personLabel ?positionLabel ?startTime ?endTime WHERE {
  # 某人拥有 P39 (担任职位) 的声明
  ?person p:P39 ?statement.
  
  # 声明的值是 美国总统 (Q11696)
  ?statement ps:P39 wd:Q11696;
             pq:P580 ?startTime.  # 限定符：开始时间
  
  OPTIONAL { ?statement pq:P582 ?endTime. } # 限定符：结束时间 (可能还在任)

  SERVICE wikibase:label { bd:serviceParam wikibase:language "zh,en". }
}
LIMIT 50
```

### 场景 D：生成“多跳推理与对比” (Multi-hop Reasoning)
**目标**：生成“A的父亲是谁？”或者“A和B谁更老？”的数据对。
**关键点**：在一个查询中引用两个实体或两层关系。

```sparql
# 查询：祖孙三代关系 (用于生成亲属关系推理题)
SELECT ?grandchildLabel ?childLabel ?grandparentLabel WHERE {
  ?grandchild wdt:P31 wd:Q5;
              wdt:P22 ?child.      # child 是 grandchild 的父亲
  ?child wdt:P31 wd:Q5;
         wdt:P22 ?grandparent.     # grandparent 是 child 的父亲
         
  SERVICE wikibase:label { bd:serviceParam wikibase:language "zh,en". }
}
LIMIT 50
```

---

## 5. 高级采样技巧 (Advanced Sampling Strategies)

为了让对话数据集具有**多样性**，避免偏见，请使用以下技巧。

### 5.1 随机采样的正确姿势 (MD5 Hash)
Wikidata 的 `RAND()` 经常超时。使用 ID 哈希法可以实现**确定性随机**（Deterministic Randomness），方便复现。

```sparql
# 随机抽取 1/100 的人类数据
SELECT ?item ?itemLabel WHERE {
  ?item wdt:P31 wd:Q5.
  
  # 将 QID 转换为字符串并哈希
  BIND(STR(?item) AS ?str_id)
  BIND(MD5(?str_id) AS ?hash)
  
  # 筛选哈希值以 "a5" 开头的实体 (16*16 = 256, 约 1/256 的采样率)
  FILTER(STRSTARTS(?hash, "a5")) 
  
  SERVICE wikibase:label { bd:serviceParam wikibase:language "zh". }
}
LIMIT 100
```

### 5.2 强制地域多样性 (Geo-Diversity)
避免数据集中全是欧美国家。使用 `VALUES` 轮询不同大洲或国家。

```sparql
SELECT ?countryLabel ?cityLabel WHERE {
  # 强制指定要采样的国家列表：中国、尼日利亚、巴西、法国
  VALUES ?country { wd:Q148 wd:Q1033 wd:Q155 wd:Q142 }
  
  ?city wdt:P31 wd:Q515;    # 是城市
        wdt:P17 ?country.   # 属于上述国家之一
        
  SERVICE wikibase:label { bd:serviceParam wikibase:language "zh". }
}
LIMIT 200
```

### 5.3 过滤“垃圾”数据 (Quality Filters)
在造数据前，先在 SPARQL 层过滤掉质量差的实体。

```sparql
FILTER NOT EXISTS { ?item wdt:P31 wd:Q4167410 } # 排除消歧页
FILTER NOT EXISTS { ?item wdt:P31 wd:Q13406463 } # 排除“维基媒体列表”
FILTER(BOUND(?itemLabel)) # 排除没有 Label 的
FILTER(!REGEX(STR(?itemLabel), "^Q[0-9]+$")) # 排除 Label 等于 QID 的
```

---

## 6. 常见错误与排障 (Troubleshooting & Gotchas)

### 错误 1：查询结果为空，但明明有数据
*   **原因 A**：你使用了 `wdt:Pxxx`，但该数据包含限定符，有时需要检查是否有特殊 rank（虽然少见，但 wdt 只指向 best rank）。
*   **原因 B**：**语言回退失败**。如果实体没有中文标签，且你只指定了 `"zh"`，Label Service 可能返回空行或 QID。
    *   *Fix*：始终使用 `"zh,en,fr"` 等多语言回退，或者在 SELECT 变量中显式包含 `?item` (QID)，以便调试。

### 错误 2：`Query Timeout` (500/502)
*   **原因**：扫描范围太大。
    *   *Bad*: `?item wdt:P31 wd:Q5` (数千万人)。
    *   *Fix*: 必须加次级索引，如 `?item wdt:P31 wd:Q5; wdt:P27 wd:Q148` (中国用户)。
*   **技巧**：如果你必须跑全量，请使用 `OFFSET` 分页，或者使用上面的 **MD5 分片** 跑多次。

### 错误 3：多值爆炸 (Cartesian Product)
*   **现象**：一个人有 3 个职业，2 个国籍。查询返回了 3*2=6 行数据。
*   **后果**：生成对话时会重复：“他是演员，是中国人。他是作家，是中国人...”。
*   **Fix**：在 SPARQL 中使用 `GROUP_CONCAT` 聚合。

```sparql
# 将多个职业合并成一个字符串: "演员, 作家, 导演"
SELECT ?itemLabel (GROUP_CONCAT(?jobLabel; separator=", ") AS ?jobs) WHERE {
  ?item wdt:P31 wd:Q5;
        wdt:P106 ?job.
  ?job rdfs:label ?jobLabel. FILTER(LANG(?jobLabel) = "zh")
  
  SERVICE wikibase:label { bd:serviceParam wikibase:language "zh". }
}
GROUP BY ?itemLabel
```

---

## 7. 练习题 (Exercises)

### 基础题
**练习 1：化学元素周期表**
查询所有“化学元素”（P31: Q11344），列出它们的“符号”（P246）和“原子序数”（P1086）。
<details>
<summary>点击查看答案</summary>

```sparql
SELECT ?elementLabel ?symbol ?atomicNumber WHERE {
  ?element wdt:P31 wd:Q11344;
           wdt:P246 ?symbol;
           wdt:P1086 ?atomicNumber.
  SERVICE wikibase:label { bd:serviceParam wikibase:language "zh,en". }
}
ORDER BY ?atomicNumber
```
</details>

### 进阶题
**练习 2：查找“同时代”的人**
编写一个查询，找出所有和“爱因斯坦”（Q937）出生在同一年（P569），且也是“物理学家”（P106: Q169470）的人。
*提示：使用 `YEAR(?date)` 函数提取年份。*

<details>
<summary>点击查看答案</summary>

```sparql
SELECT ?scientistLabel ?birthDate WHERE {
  wd:Q937 wdt:P569 ?einsteinBirth.
  BIND(YEAR(?einsteinBirth) AS ?targetYear)
  
  ?scientist wdt:P106 wd:Q169470;
             wdt:P569 ?birthDate.
             
  FILTER(YEAR(?birthDate) = ?targetYear)
  FILTER(?scientist != wd:Q937) # 排除爱因斯坦自己
  
  SERVICE wikibase:label { bd:serviceParam wikibase:language "zh,en". }
}
```
</details>

**练习 3：数据清洗挑战**
查询“世界上最高的 10 座建筑”，但必须排除掉“规划中”或“未建成”的项目。
*提示：通常可以通过检查 `P580` (开始时间) 或 `P571` (成立/创建时间) 是否存在，或者检查是否有 `P582` (结束/拆除) 来做简单过滤。更严格的方法是检查 `P31` 的子类是否包含“未建成建筑”。这里尝试简单的存在性检查。*

<details>
<summary>点击查看答案</summary>

```sparql
SELECT ?building ?buildingLabel ?height WHERE {
  ?building wdt:P31/wdt:P279* wd:Q41176; # 建筑或其子类
            wdt:P2048 ?height.           # 高度
            
  # 必须有建成时间，作为“已建成”的一个弱代理
  FILTER EXISTS { ?building wdt:P571 ?date } 
  
  SERVICE wikibase:label { bd:serviceParam wikibase:language "zh,en". }
}
ORDER BY DESC(?height)
LIMIT 10
```
</details>

---

[回到目录](index.md) | [上一章：工程化实践](chapter11.md)
