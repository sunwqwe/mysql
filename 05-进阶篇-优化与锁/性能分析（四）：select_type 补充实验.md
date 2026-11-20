# 性能分析（四）：select_type 补充实验——DERIVED 与 UNION

> 接《性能分析（三）：EXPLAIN 详解》。三篇主文只实测了 PRIMARY + SUBQUERY，
> 本篇补齐 select_type 中另外两类：**UNION / UNION RESULT**，并实测了
> **FROM 子查询（DERIVED）**——结果与视频（MySQL）预期不同，
> 发现了 MariaDB 优化器的"派生表合并"行为，记录如下。
> 全部输出为 MariaDB 13 在 tb_user（50 万行）上的真实运行结果。

## 一、实验一：FROM 子查询（DERIVED）——派生表被"合并"了 ⭐⭐

### 1.1 实验设计

按视频（MySQL 8.0）的讲法，FROM 里的子查询应产生 DERIVED：
外层扫派生表、内层物化临时表，预期出现两行。

```sql
EXPLAIN SELECT * FROM (SELECT name FROM tb_user WHERE id < 100) t;
```

### 1.2 实测输出

```
+------+-------------+---------+-------+---------------+---------+---------+------+------+-------------+
| id   | select_type | table   | type  | possible_keys | key     | key_len | ref  | rows | Extra       |
+------+-------------+---------+-------+---------------+---------+---------+------+------+-------------+
|    1 | SIMPLE      | tb_user | range | PRIMARY       | PRIMARY | 4       | NULL |   96 | Using where |
+------+-------------+---------+-------+---------------+---------+---------+------+------+-------------+
1 row in set (0.010 sec)
```

**预期是两行（PRIMARY + DERIVED），实际只有一行 SIMPLE——派生表"消失"了。**

### 1.3 原因：派生表合并（derived table merging）

优化器发现这个子查询只是简单的过滤 + 投影（WHERE + SELECT 列，
没有 GROUP BY / 聚合 / DISTINCT / LIMIT / UNION），于是把它**直接改写合并进外层**：

```
写下的 SQL：  SELECT * FROM (SELECT name FROM tb_user WHERE id < 100) t;
实际执行：    SELECT name FROM tb_user WHERE id < 100;
```

不需要"先物化临时表、再扫临时表"，一步到位走主键 range 扫 96 行。

### 1.4 这个优化为什么重要

```
不合并（物化派生表）：              合并后（本例实际执行）：
内层扫索引 → 结果写入临时表          直接在原表上走主键 range
        ↓                                  ↓
外层全扫临时表（ALL，无索引！）       96 行、有索引可用
```

派生表一旦物化，就变成**没有索引的临时表**，外层对它通常只能 ALL——
合并优化从根上消灭了这个隐患。

### 1.5 什么情况下不合并（才会看到 DERIVED）

子查询包含以下任一要素时，无法合并，只能物化（出现 `<derivedN>`）：

- UNION
- GROUP BY
- DISTINCT
- 聚合函数
- LIMIT / OFFSET

### 1.6 验证实验（待补实测）

```sql
-- GROUP BY + LIMIT 阻止合并，预期出现两行结构：
-- | 1 | PRIMARY | <derived2> | ALL   |
-- | 2 | DERIVED | tb_user    | range |
EXPLAIN SELECT * FROM
  (SELECT name FROM tb_user WHERE id < 100 GROUP BY name LIMIT 5) t;
```

> 💡 实践启示：FROM 子查询能简单就简单，避免不必要的 GROUP BY/DISTINCT/LIMIT
> 把子查询困在派生表里失去索引；反之若刻意想隔离结果集，才知道代价是临时表。

## 二、实验二：UNION 三行结构——标准教材 ⭐⭐

### 2.1 实验设计

```sql
EXPLAIN SELECT name FROM tb_user WHERE id < 3
UNION
SELECT name FROM tb_user WHERE id > 499000;
```

### 2.2 实测输出

```
+------+--------------+------------+-------+---------------+---------+---------+------+------+-------------+
| id   | select_type  | table      | type  | possible_keys | key     | key_len | ref  | rows | Extra       |
+------+--------------+------------+-------+---------------+---------+---------+------+------+-------------+
|    1 | PRIMARY      | tb_user    | range | PRIMARY       | PRIMARY | 4       | NULL |    2 | Using where |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
|    2 | UNION        | tb_user    | range | PRIMARY       | PRIMARY | 4       | NULL | 1028 | Using where |
| NULL | UNION RESULT | <union1,2> | ALL   | NULL          | NULL    | NULL    | NULL | NULL |             |
+------+--------------+------------+-------+---------------+---------+---------+------+------+-------------+
3 rows in set (0.008 sec)
```

### 2.3 逐行解读

**第一行：id=1，PRIMARY**

- UNION 的**第一个** SELECT 身份标为 PRIMARY；
- 主键 range，rows=2（id=1、2 两行），估算精确。

**第二行：id=2，UNION**

- 第二段 SELECT 身份标为 UNION（"UNION 的第二段及以后"）；
- 主键 range，rows=1028——`id > 499000` 是从 B+Tree 最右端往左数
  的范围，优化器估算约 1028 行；
- 注意：UNION 各分支**各自独立执行、互不依赖**，此处 id 大小
  不代表"大的先执行"（与子查询场景的 id 规则不同）。

**第三行：id=NULL，UNION RESULT ⭐ 关键行**

- **id=NULL**：不属于任何 SELECT 分支，是两个分支结果合并后的收尾操作；
- **table=`<union1,2>`**：尖括号 = 临时表，含义是"由 id=1 和 id=2
  两个分支的结果拼成的临时表"；
- **type=ALL**：对这张临时表全扫——**临时表上没有索引**；
- 它的实际工作是**去重**：`UNION` 默认等价于 `UNION DISTINCT`。

### 2.4 由此得出的性能结论（面试高频）

```sql
-- 确认两段结果无重复时，务必用 UNION ALL：
EXPLAIN SELECT name FROM tb_user WHERE id < 3
UNION ALL
SELECT name FROM tb_user WHERE id > 499000;
```

`UNION ALL` 的执行计划会**少掉 UNION RESULT 那一行**——不建临时表、
不去重，两段结果直接拼接输出。数据量大时是零成本的差异。

## 三、知识点汇总

### select_type 实测覆盖情况（与《性能分析（三）》合计已全覆盖）

| select_type | 触发场景 | 实测结果 |
|---|---|---|
| SIMPLE | 简单查询 / **被合并的派生表外层** | 《三》各例 + 本篇实验一 |
| PRIMARY | 最外层 / UNION 第一段 | 《三》子查询例 + 本篇 UNION 第一段 |
| SUBQUERY | WHERE 非相关子查询 | 《三》MAX(id) 例 |
| DERIVED | FROM 子查询 | **本篇：MariaDB 默认合并优化，未直接出现**；需 GROUP BY/LIMIT 等阻止 |
| UNION | UNION 第二段及以后 | 本篇实测（id=2） |
| UNION RESULT | UNION 去重收尾 | 本篇实测（id=NULL + `<union1,2>` 临时表 ALL） |

### 本篇两个新知识点

1. **派生表合并**：简单 FROM 子查询会被优化器合并进外层，
   避免"物化成无索引临时表再全扫"；含 UNION / GROUP BY / DISTINCT /
   聚合 / LIMIT 的子查询无法合并。
2. **UNION vs UNION ALL**：UNION 额外产生 `UNION RESULT` 临时表去重
   （type=ALL）；确认无重复时用 UNION ALL，省一趟临时表操作。

### 回顾《性能分析（三）》的核心口诀

- id：相同往下走，不同大的先（UNION 分支除外，各自独立）
- type 天梯（MariaDB 实测）：NULL > const > eq_ref > ref > range > index > ALL
- key_len：n×4+2+NULL 字节，判断复合索引用了几列
- Extra：Using index 最优；Using temporary / Using filesort 要优化

## 📌 待办

- [ ] 补跑 1.6：GROUP BY + LIMIT 版派生表，实测 DERIVED 两行结构
- [ ] 补跑 2.4：UNION ALL 对比实验，确认 UNION RESULT 行消失
- [ ] 补跑：`GROUP BY gender` 触发 Using temporary（《三》遗留）
- [ ] 索引失效场景（二）：函数运算 / 隐式转换 / OR / NOT / 数据分布
