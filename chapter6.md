# Chapter 6：自动化抓取与缓存：从 WDQS 到本地数据管道

在前面的章节中，你已经掌握了 SPARQL 查询的编写技巧。现在，我们面临的问题是如何将“一次性查询”转变为“生产级数据流水线”。

如果你只是想下载几百条数据，点击 WDQS 网页上的 Download 按钮就够了。但对于**构造对话数据**而言，我们通常面临以下挑战：
1. **数据量大**：需要采样数万甚至数十万个实体。
2. **查询复杂**：单一查询容易超时（Timeout）。
3. **网络不稳定**：WDQS 是公共资源，会有速率限制（Rate Limit）和偶尔的宕机。
4. **迭代频繁**：你修改了 Prompt 模板，需要重新跑数据，如果每次都重新请求 API，效率极低且不环保。

本章将指导你使用 Python 构建一个健壮的（Robust）、支持断点续传和本地缓存的数据抓取系统。

---

## 1. 学习目标

- **掌握 WDQS API 协议**：HTTP 方法选择、User-Agent 规范、JSON 响应解析。
- **构建健壮的请求器**：实现指数退避（Exponential Backoff）重试机制，优雅处理 429/50x 错误。
- **实施高效缓存策略**：实现“Query Hash”缓存，将网络 IO 转化为本地磁盘 IO。
- **解决大规模数据获取**：放弃低效的 OFFSET 分页，掌握基于 ID 或属性的“切片（Slicing）”技巧。
- **数据持久化**：使用 JSONL 格式构建流式数据处理管道。

---

## 2. 核心架构：数据管道 (The Pipeline)

一个成熟的对话数据获取系统通常包含四个模块。不要在一个脚本里写完所有逻辑，应该尽量解耦。

```ascii
[ 配置层: Topic Definitions ]
       | (生成 SPARQL)
       v
+-------------+      1. Check      +------------------+
|             | -----------------> |                  |
|  Fetcher    |      2. Return     |   Local Cache    |
| (Py Script) | <----------------- | (disk/sqlite)    |
|             |      (If Hit)      |                  |
+-------------+                    +------------------+
       |
       | 3. HTTP Request (If Miss)
       | (with Retry & Backoff)
       v
+-------------+                    +------------------+
|             | 4. JSON Response   |                  |
|   WDQS      | -----------------> |  Raw Data Dump   |
| (Internet)  | 5. Write Cache     |  (.jsonl files)  |
+-------------+                    +------------------+
                                           |
                                           | 6. Stream Read
                                           v
                                   +------------------+
                                   |                  |
                                   |   Normalizer     |
                                   | (Clean & Format) |
                                   |                  |
                                   +------------------+
```

### 2.1 必须遵守的规则：User-Agent

Wikidata 对匿名爬虫非常敏感。如果你的 Python 脚本不带 Header，默认 User-Agent 通常是 `python-requests/x.x.x`，这会被直接封禁。

**Rule of Thumb**：
> 永远在 Header 中附带你的项目名称和联系方式（邮箱）。

```python
HEADERS = {
    'User-Agent': 'MyDialogueBot/1.0 (bot_admin@example.com) based on Wikidata-Toolkit',
    'Accept': 'application/sparql-results+json'
}
```

### 2.2 GET vs POST

WDQS 支持 GET 和 POST。
- **GET**：将查询拼接到 URL 参数中。限制：URL 长度通常不能超过 2KB~4KB。
- **POST**：将查询放在 Body 中。限制：几乎没有长度限制。

**最佳实践**：为了避免“URI Too Long”错误，统一封装一个使用 **POST** 的请求函数。

---

## 3. 核心技术详解

### 3.1 缓存策略：以查询为键 (Query-as-Key)

在开发调试阶段，你可能会运行脚本 50 次。如果没有缓存，你会对 Wikidata 发起 50 次相同的请求，既慢又可能被封 IP。

我们推荐**文件哈希缓存**：

1. 输入 SPARQL 查询字符串。
2. 计算 MD5 哈希值（例如 `a1b2c3d4...`）。
3. 检查 `cache/a1b2c3d4.json` 是否存在。
   - **Hit**: 直接读取文件内容。
   - **Miss**: 发起 HTTP 请求，成功后将结果写入该文件。

这种方法的优点是**幂等性**（Idempotency）：只要查询语句没变，结果就是稳定的。

### 3.2 错误处理与指数退避

网络错误主要分为两类：
1. **硬错误 (400 Bad Request)**：你的 SPARQL 语法错了。重试 100 次也没用。策略：**抛出异常，人工修复**。
2. **软错误 (429 Too Many Requests / 5xx Server Error)**：服务器忙。策略：**等待后重试**。

**指数退避 (Exponential Backoff)** 算法：
- 第 1 次失败：等待 1 秒
- 第 2 次失败：等待 2 秒
- 第 3 次失败：等待 4 秒
- ...
- 超过最大次数：放弃并报错。

### 3.3 大规模数据的切片策略 (Slicing)

这是本章最高级的技巧。

#### ❌ 反模式：使用 OFFSET 分页
```sparql
# 极不推荐
SELECT * WHERE { ... } LIMIT 1000 OFFSET 10000
```
随着 OFFSET 增大，数据库必须扫描并丢弃前面的记录，查询会变得极其缓慢直至超时（Timeout）。

#### ✅ 最佳实践：基于 ID 或属性切片

**方法 A：按时间/数值切片**
适用于有明确时间属性的数据（如出生日期）。
- Query 1: 1900-1910
- Query 2: 1910-1920
- ...

**方法 B：按 QID 范围切片 (万能法)**
由于每个实体都有唯一的数值 ID（Q123 中的 123），我们可以利用 SPARQL 的字符串处理或数值转换来分片。

虽然 SPARQL 处理字符串 ID 较慢，但更高效的是结合 `wd:Q*` 实际上是 IRI 的特性，或者直接依赖**外部逻辑生成多条查询**。

例如，我们要查所有“人类”：
- 不要在 SPARQL 里写 `LIMIT 1000000`.
- 而是写 10 个查询，每个查询增加约束：
  - `Query 1`: ... 且 `?item` 位于“亚洲”
  - `Query 2`: ... 且 `?item` 位于“欧洲”
  - ...

---

## 4. 实战代码构建

我们将构建一个名为 `WikidataFetcher` 的类。

### 4.1 基础结构与缓存

```python
import os
import hashlib
import json
import time
import requests

class WikidataFetcher:
    def __init__(self, cache_dir="cache"):
        self.endpoint = "https://query.wikidata.org/sparql"
        self.headers = {
            'User-Agent': 'DialogueSynthBot/0.1 (me@mysite.com)',
        }
        self.cache_dir = cache_dir
        if not os.path.exists(cache_dir):
            os.makedirs(cache_dir)

    def _get_cache_path(self, query):
        # 计算 MD5
        query_hash = hashlib.md5(query.encode('utf-8')).hexdigest()
        return os.path.join(self.cache_dir, f"{query_hash}.json")
    
    def fetch(self, query, force_refresh=False):
        cache_path = self._get_cache_path(query)
        
        # 1. 尝试读取缓存
        if not force_refresh and os.path.exists(cache_path):
            print(f"Loading from cache: {cache_path}")
            with open(cache_path, 'r', encoding='utf-8') as f:
                return json.load(f)
        
        # 2. 缓存未命中，发起请求
        print("Fetching from WDQS...")
        data = self._make_request(query)
        
        # 3. 写入缓存
        with open(cache_path, 'w', encoding='utf-8') as f:
            json.dump(data, f, ensure_ascii=False)
            
        return data

    def _make_request(self, query):
        # 将在下一节实现具体的请求逻辑
        pass
```

### 4.2 实现重试逻辑

```python
    def _make_request(self, query):
        max_retries = 5
        for attempt in range(max_retries):
            try:
                # 使用 POST 方法
                response = requests.post(
                    self.endpoint, 
                    data={'query': query, 'format': 'json'}, 
                    headers=self.headers,
                    timeout=60 # 设置超时很重要
                )
                
                # 400 错误直接抛出，不重试
                if response.status_code == 400:
                    raise ValueError(f"SPARQL Syntax Error: {response.text}")

                # 429 或 5xx 错误进行重试
                if response.status_code in [429, 500, 502, 503, 504]:
                    sleep_time = 2 ** attempt # 1, 2, 4, 8...
                    print(f"Error {response.status_code}. Retrying in {sleep_time}s...")
                    time.sleep(sleep_time)
                    continue
                
                response.raise_for_status() # 其他错误抛出异常
                
                return response.json()
                
            except requests.exceptions.RequestException as e:
                print(f"Network error: {e}")
                if attempt == max_retries - 1:
                    raise
                time.sleep(2 ** attempt)
        
        raise Exception("Max retries exceeded")
```

---

## 5. 本章小结

1.  **工程化第一**：不要依赖浏览器，要依赖代码。
2.  **缓存是必选项**：`md5(query)` 是实现缓存最简单有效的方法，能节省 90% 的调试时间。
3.  **拥抱 JSONL**：在处理列表型数据时，JSONL（每行一个 JSON）比 JSON Array 更节省内存，比 CSV 表达能力更强。
4.  **切片优于分页**：用业务逻辑（时间、地点、ID段）拆分查询，避免使用深度的 OFFSET。
5.  **礼貌爬取**：设置 User-Agent，遇到 429 请等待。

---

## 6. 练习题

### 基础题

**练习 6.1：JSON 结果提取**
Wikidata 返回的 JSON 结构比较深。编写一个辅助函数 `simplify_results(data)`，将原始的 SPARQL JSON 响应转化为简单的字典列表。
输入示例：
```json
{"head": {...}, "results": {"bindings": [{"item": {"type": "uri", "value": "http://.../Q1"}, "itemLabel": {"value": "Cat"}}]}}
```
期望输出：
```python
[{"item": "Q1", "itemLabel": "Cat"}]
```
*提示：需要处理 value 中的 URL，提取最后的 QID。*

<details>
<summary>点击查看答案思路</summary>

```python
def simplify_results(raw_data):
    simplified = []
    bindings = raw_data.get('results', {}).get('bindings', [])
    for row in bindings:
        new_row = {}
        for key, value_obj in row.items():
            val = value_obj['value']
            # 如果是wikidata实体URL，提取QID
            if 'entity/Q' in val:
                val = val.split('/')[-1]
            new_row[key] = val
        simplified.append(new_row)
    return simplified
```
</details>

**练习 6.2：JSONL 写入器**
编写一个函数 `save_to_jsonl(data_list, filename)`。
要求：
1. 如果文件不存在，创建并写入。
2. 如果文件存在，**追加**写入。
3. 确保中文不被转义（显示为汉字而不是 `\uXXXX`）。

<details>
<summary>点击查看答案思路</summary>

```python
def save_to_jsonl(data_list, filename):
    # 使用 'a' 模式进行追加
    with open(filename, 'a', encoding='utf-8') as f:
        for item in data_list:
            # ensure_ascii=False 保证中文正常显示
            f.write(json.dumps(item, ensure_ascii=False) + '\n')
```
</details>

---

### 挑战题

**练习 6.3：自动切分查询生成器 (Challenge)**
假设你需要查询 1900 年到 2020 年每一部电影的名称。单一查询会超时。
编写一个 Python 生成器 `query_generator()`。
它接受一个基本的 SPARQL 模板（包含 `{start}` 和 `{end}` 占位符），起始年份和结束年份，以及步长（比如 5 年）。
它通过 `yield` 返回填充好时间的 SPARQL 字符串。

*SPARQL 时间过滤提示*：`FILTER(?date >= "{start}-01-01"^^xsd:dateTime && ?date < "{end}-01-01"^^xsd:dateTime)`

<details>
<summary>点击查看答案思路</summary>

```python
def query_generator(base_template, start_year, end_year, step=5):
    current = start_year
    while current < end_year:
        next_val = min(current + step, end_year)
        # 构造参数字典
        params = {
            "start": current,
            "end": next_val
        }
        # 填充模板
        yield base_template.format(**params)
        current = next_val

# 使用示例
template = """
SELECT ?film ?filmLabel WHERE {{
  ?film wdt:P31 wd:Q11424 ; wdt:P577 ?date .
  FILTER(?date >= "{start}-01-01"^^xsd:dateTime && ?date < "{end}-01-01"^^xsd:dateTime)
  SERVICE wikibase:label {{ bd:serviceParam wikibase:language "zh,en". }}
}}
"""
for q in query_generator(template, 1900, 2020, 10):
    print(q) # 打印切分后的查询，随后可送入 Fetcher
```
</details>

**练习 6.4：审计日志 (Audit Log)**
修改你的 Fetcher 类。每次发生网络请求（不是 Cache Hit）时，向一个名为 `audit.log` 的文件中写入一行日志。
格式：`[时间] [状态码] [耗时ms] [查询哈希] [返回记录数]`
这对于监控你的数据管道健康状况非常重要。

<details>
<summary>点击查看答案思路</summary>

在 `_make_request` 方法内部记录开始时间 `start = time.time()`，请求结束后记录 `end = time.time()`。
使用 Python 的 `logging` 模块或简单的文件追加写入。
```python
duration = (time.time() - start) * 1000
record_count = len(data.get('results', {}).get('bindings', []))
log_line = f"[{time.ctime()}] [{response.status_code}] [{duration:.0f}ms] [{hash}] [{record_count}]\n"
with open("audit.log", "a") as f:
    f.write(log_line)
```
</details>

---

## 7. 常见陷阱与错误 (Gotchas)

### 🔴 陷阱 1：`wdt:` vs `p:` 的混淆导致数据膨胀
当你不需要限定符（Qualifier）或引用（Reference）时，**千万不要**查询 `p:Pxxx`。
- `wdt:Pxxx`：直接指向值（Truthy value），通常每个属性只有 1-2 个值。
- `p:Pxxx`：指向声明节点（Statement node）。如果你查了这个节点，SPARQL 结果集会发生笛卡尔积爆炸。
**Rule**: 除非你要造包含“时间/地点/来源”等细节的复杂对话，否则只用 `wdt:`。

### 🔴 陷阱 2：Label Service 的隐形超时
`SERVICE wikibase:label { ... }` 是个黑盒，有时候会极慢。
**调试技巧**：如果查询超时，尝试去掉 Label Service，只查 QID。如果速度变快，说明瓶颈在 Label 服务。此时可以在 Python 端后续批量查询 Label，而不是在复杂 SPARQL 中做 Join。

### 🔴 陷阱 3：Python 字典无序 (但在 3.7+ 已改善)
虽然 Python 3.7+ 字典保持插入顺序，但 JSON 标准本身是无序的。
不要依赖 `keys()` 的顺序来对齐 CSV 列头，始终显式指定列名列表。

### 🔴 陷阱 4：忽略数据类型的转换
SPARQL 返回的时间通常是 `2023-01-01T00:00:00Z` 格式的字符串。
在生成对话时，直接把这个字符串填进模板会很怪（“他在 1990-01-01T00:00:00Z 出生”）。
你需要在**规范化阶段（Normalizer）**编写函数，将其转换为“1990年1月1日”或“1990年”。

---

## 下一章预告

现在你的硬盘里可能已经躺着 100 个 `.jsonl` 文件，包含 50,000 个实体的原始数据。但它们还是这种格式：
`{"entity": "Q42", "birth_date": "1952-03-11"}`。

如何把它变成：“*道格拉斯·亚当斯出生于1952年，他是《银河系漫游指南》的作者。*”？
甚至变成多轮对话：“*Q: 道格拉斯是哪国人？ A: 英国。*”？

下一章 **[Chapter 7：中文自然语言生成](chapter7.md)**，我们将构建核心的**模板引擎**与**自然语言润色器**。
