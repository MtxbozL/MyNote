
## 错误代码：

```SQL
# Write your MySQL query statement below

SELECT
    r.contest_id,
    ROUND(
        IFNULL(
			COUNT(r.user_id) / 3 * 100), 0), 2) AS percentage
FROM
    Users u  
LEFT JOIN
    Register r
ON
    u.user_id = r.user_id
WHERE
    r.contest_id <> BETWEEN 208 AND 210
ORDER BY
    percentage DESC
ORDER BY
    r.contest_id ASC;
```


## 解析：

### 一、你的代码 3 类致命错误（按优先级排序）

#### 1. 语法错误（直接导致执行报错）

- **括号不匹配**：`ROUND(IFNULL(COUNT(r.user_id)/3*100),0),2)` 中`ROUND`函数的括号完全错位，正确格式是`ROUND(要四舍五入的数, 小数位数)`。
- **重复 ORDER BY**：写了两次`ORDER BY`，SQL 不允许，应合并为`ORDER BY 字段1 排序, 字段2 排序`。
- **非法 WHERE 条件**：`r.contest_id <> BETWEEN 208 AND 210` 语法错误，`BETWEEN`本身是范围判断，不能和`<>`组合使用，且该条件完全多余（题目要求返回所有赛事）。

#### 2. 核心逻辑错误（结果完全错误）

**分母硬编码为 3**：这是最严重的问题。总用户数不是固定值 3，必须从`Users`表动态统计，否则换一个测试用例就会完全失败。

#### 3. 冗余且错误的表连接

你写的`Users LEFT JOIN Register`完全多余且方向错误：

- 不需要关联`Users`表：`Register`表的`user_id`是外键，保证所有用户都有效，注册人数直接从`Register`表统计即可。
- 左连接方向错误：会导致**没有用户注册的赛事**被过滤掉，不符合题目 “返回所有赛事注册率” 的要求。

---

### 二、完整解题思路（三步法）

这是一道**分组统计 + 子查询求比例**的经典题，核心公式：

> **赛事注册率 = (该赛事注册用户数 ÷ 系统总用户数) × 100**

| 步骤  | 目标                     | 用到的语法                                    |
| :-- | :--------------------- | :--------------------------------------- |
| 1   | 计算系统总用户数（所有赛事共用分母）     | 标量子查询 `(SELECT COUNT(*) FROM Users)`     |
| 2   | 按赛事 ID 分组，统计每个赛事的注册用户数 | `GROUP BY contest_id` + `COUNT(user_id)` |
| 3   | 计算注册率并排序               | `ROUND()` + 双字段排序（先降序后升序）                |

---

### 三、标准正确代码（最简最优，面试首选）

```sql
SELECT
    contest_id,
    -- 核心：注册人数 / 总用户数 * 100，保留两位小数
    ROUND(COUNT(user_id) * 100 / (SELECT COUNT(*) FROM Users), 2) AS percentage
FROM Register
GROUP BY contest_id
-- 先按注册率降序，再按赛事ID升序
ORDER BY percentage DESC, contest_id ASC;
```

---

### 四、逐行语法详解（为什么要这么写）

#### 1. 为什么用标量子查询`(SELECT COUNT(*) FROM Users)`

- 这是 **动态获取总用户数** 的唯一正确方式，无论总用户数是多少，代码都能通用。
- 标量子查询只会执行一次，效率很高，不会影响性能。

#### 2. 为什么不需要关联`Users`表

- `Register`表的`user_id`是外键，引用自`Users`表的主键，保证了所有注册用户都是系统有效用户。
- 直接从`Register`表按`contest_id`分组统计`COUNT(user_id)`，就能得到每个赛事的准确注册人数。

#### 3. 为什么用`COUNT(user_id)`而不是`COUNT(*)`

- 两者结果完全相同（因为`user_id`非空），但`COUNT(user_id)`语义更清晰，明确表示统计 “注册用户的数量”。

#### 4. 为什么双字段排序

- 题目明确要求：**注册率高的排在前面**，如果注册率相同，**赛事 ID 小的排在前面**。
- 这是 SQL 排序的标准写法，`ORDER BY A DESC, B ASC` 表示先按 A 降序，A 相同再按 B 升序。

---

### 五、进阶：处理无注册用户的赛事（严谨版）

如果题目要求包含**没有任何人注册的赛事**（注册率为 0），则需要先获取所有赛事 ID，再左连接统计：

```sql
SELECT
    c.contest_id,
    ROUND(IFNULL(COUNT(r.user_id) * 100 / (SELECT COUNT(*) FROM Users), 0), 2) AS percentage
FROM (SELECT DISTINCT contest_id FROM Register) c
LEFT JOIN Register r ON c.contest_id = r.contest_id
GROUP BY c.contest_id
ORDER BY percentage DESC, c.contest_id ASC;
```

- 用`SELECT DISTINCT contest_id FROM Register`获取所有赛事 ID
- 用`LEFT JOIN`保留无注册用户的赛事
- 用`IFNULL(..., 0)`将无注册用户的赛事注册率从`NULL`转为 0

---

### 六、易错点总结

1. 永远不要硬编码分母或分子，必须用子查询动态统计。
2. 比例计算注意顺序：`COUNT * 100 / 总数量` 比 `COUNT / 总数量 * 100` 精度更高（避免整数除法截断）。
3. 双字段排序不要写两次`ORDER BY`，用逗号分隔即可。