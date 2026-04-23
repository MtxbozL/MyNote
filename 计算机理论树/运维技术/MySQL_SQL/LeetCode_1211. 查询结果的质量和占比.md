## 错误代码：

```sql
SELECT DISTINCT
    query_name,
    ROUND(SUM(rating / position) / (SELECT COUNT(*) FROM Queries), 2) AS quality,
     AS poor_query_percentage
FROM
    Queries
GROUP BY
    query_name
```

## 核心疑惑点：

- 如果用 `WHERE rating < 3` 过滤，就只能统计劣质查询，无法同时统计全部查询的质量了。

## 正确解法：使用「条件聚合」（`IF()` + 聚合函数）

这是 SQL 中 **同时统计同一分组下不同条件数据** 的标准技巧，也是这道题的灵魂。它的原理是：

- 不使用 `WHERE` 过滤任何行，保留该分组下的所有数据
- 在聚合函数内部用 `IF(条件, 满足时的值, 不满足时的值)` 单独统计符合条件的行
- 这样就能在同一个 `GROUP BY` 分组里，同时得到 "全部数据的统计" 和 "部分数据的统计"

---

### 二、你的代码两个核心错误

1. **分母错误**：`(SELECT COUNT(*) FROM Queries)` 是统计 **全局所有查询的总条数** ，而题目要求的是**每个 query 自己的结果条数**作为分母。
2. **缺少劣质查询占比的计算**：不知道怎么在不影响质量计算的前提下统计 rating<3 的数量。
3. **多余的 DISTINCT**：`GROUP BY query_name` 已经保证了每个 query 只出现一次，`DISTINCT` 完全多余。

---

### 三、完整解题思路

这道题是 **条件聚合** 的经典入门题，核心是两个公式的实现：

| 指标               | 公式                                                 | 对应 SQL 语法                                    |
| :--------------- | :------------------------------------------------- | :------------------------------------------- |
| **质量 (quality)** | 该 query 所有结果的 `(rating/position)` 的平均值             | `AVG(rating / position)`                     |
| **劣质查询占比**       | (该 query 中 `rating < 3` 的结果数 ÷ 该 query 总结果数) × 100 | `SUM(IF(rating < 3, 1, 0)) / COUNT(*) * 100` |

|步骤|目标|用到的语法|
|:--|:--|:--|
|1|按 `query_name` 分组，每个 query 单独计算指标|`GROUP BY query_name`|
|2|计算质量：所有结果的 (rating/position) 平均值|`AVG()`|
|3|计算劣质占比：用条件聚合统计 rating<3 的行数，再除以总行数|`SUM(IF(条件,1,0))`|
|4|两个指标都保留两位小数|`ROUND(..., 2)`|

---

### 四、标准正确代码（面试首选）

```sql
SELECT
    query_name,
    -- 质量：所有结果的(rating/position)的平均值
    ROUND(AVG(rating / position), 2) AS quality,
    -- 劣质查询占比：(rating<3的数量 / 总数量) * 100
    ROUND(SUM(IF(rating < 3, 1, 0)) / COUNT(*) * 100, 2) AS poor_query_percentage
FROM Queries
GROUP BY query_name;
```

---

### 五、逐行语法详解（为什么要这么写）

#### 1. 为什么用 `AVG(rating / position)` 而不是 `SUM()/COUNT()`

- 两者完全等价：`AVG(x) = SUM(x) / COUNT(x)`
- `AVG()` 写法更简洁，语义更清晰，直接表达 "平均值" 的含义

#### 2. 为什么用 `SUM(IF(rating < 3, 1, 0))`

这就是 **条件聚合** 的核心用法：

- 对分组内的每一行，判断 `rating < 3` 是否成立
- 成立返回 1，不成立返回 0
- `SUM()` 把所有 1 加起来，结果就是该分组内满足条件的总行数
- ✅ 关键：它不会过滤掉任何行，不影响 `AVG(rating/position)` 的计算

#### 3. 为什么分母是 `COUNT(*)` 而不是全局子查询

- `COUNT(*)` 统计的是 **当前分组（该 query）的总行数**
- 每个 query 的分母都是自己的结果数，而不是所有 query 的总结果数
- 这是你代码中最容易犯的逻辑错误，也是面试高频踩坑点

---

### 六、用题目示例验证计算过程（Dog 查询）

Dog 查询有 3 条结果：

|rating|position|rating/position|rating < 3?|
|:--|:--|:--|:--|
|5|1|5.0|否 (0)|
|5|2|2.5|否 (0)|
|1|200|0.005|是 (1)|

计算：

- quality = (5.0 + 2.5 + 0.005) / 3 = 7.505 / 3 ≈ 2.50
- poor_query_percentage = (0 + 0 + 1) / 3 * 100 ≈ 33.33

和题目示例输出完全一致。

---

### 七、条件聚合的通用用法（面试必背）

当你需要在同一个查询里统计多个不同条件的指标时，永远用条件聚合，不要用多个子查询或连接。

通用模板：

```sql
SELECT
    分组字段,
    -- 统计满足条件A的数量
    SUM(IF(条件A, 1, 0)) AS count_a,
    -- 统计满足条件B的总和
    SUM(IF(条件B, 字段X, 0)) AS sum_b,
    -- 计算满足条件C的平均值
    AVG(IF(条件C, 字段Y, NULL)) AS avg_c,
    -- 计算满足条件D的占比
    ROUND(SUM(IF(条件D, 1, 0)) / COUNT(*) * 100, 2) AS percentage_d
FROM 表名
GROUP BY 分组字段;
```

---

### 八、易错点总结

1. 永远不要用 `WHERE` 过滤来统计部分数据，除非你只需要这部分数据。
2. 分母永远是当前分组的 `COUNT(*)`，不要用全局子查询。
3. 条件聚合中，`IF()` 的第三个参数是 0（统计数量时）或 NULL（统计平均值时）。