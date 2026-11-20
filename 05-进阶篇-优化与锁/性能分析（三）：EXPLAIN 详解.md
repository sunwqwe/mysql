# 性能分析（三）：EXPLAIN 详解

> 接《性能分析（二）：SHOW PROFILES 与索引优化实战》。本篇系统过一遍 EXPLAIN 的 12 个字段，
> 并在 tb_user（50 万行）上逐一实测验证：访问类型天梯、key_len 计算规则、
> 联合索引最左前缀的量化实证、MariaDB 与 MySQL 的 system 差异等。
> 所有输出均为 MariaDB 13 真实运行结果。

## 一、EXPLAIN 是什么

**不执行 SQL，让优化器告诉你"如果执行，我打算怎么干"**——走不走索引、走哪个索引、
扫多少行、要不要排序。它是 SQL 优化流程中"诊断"环节的核心工具：

```
慢查询日志 / SHOW PROFILES 发现慢 SQL
        ↓
EXPLAIN 分析执行计划    ← 本篇主角
        ↓
针对性优化（建索引 / 改写 SQL）
        ↓
再 EXPLAIN 验证效果
```

## 二、基本用法

```sql
EXPLAIN   SELECT * FROM tb_user WHERE name = 'user1';
DESC      SELECT * FROM tb_user WHERE name = 'user1';   -- DESC 是 EXPLAIN 的别名
DESCRIBE  SELECT * FROM tb_user WHERE name = 'user1';   -- 三种写法完全等价
```

实测三种写法输出完全一致：

```
+------+-------------+---------+------+----------------+----------------+---------+-------+------+-----------------------+
| id   | select_type | table   | type | possible_keys  | key            | key_len | ref   | rows | Extra                 |
+------+-------------+---------+------+----------------+----------------+---------+-------+------+-----------------------+
|    1 | SIMPLE      | tb_user | ref  | idx_user_name2 | idx_user_name2 | 202     | const |    1 | Using index condition |
+------+-------------+---------+------+----------------+----------------+---------+-------+------+-----------------------+
```

## 三、12 个字段总览

```
id | select_type | table | partitions | type | possible_keys | key | key_len | ref | rows | filtered | Extra
```

重点掌握 6 个：**id、type、key、key_len、rows、Extra**。

阅读顺序口诀：**先 type 定级 → 再 key 确认索引 → rows 看量级 → Extra 找隐患**。

---

## 四、id：执行顺序

**口诀：相同往下走，不同大的先。**

### 实测：子查询（id 不同）

```sql
EXPLAIN SELECT * FROM tb_user WHERE id = (SELECT MAX(id) FROM tb_user);
```

```
+------+-------------+---------+-------+---------------+---------+---------+-------+------+------------------------------+
| id   | select_type | table   | type  | possible_keys | key     | key_len | ref   | rows | Extra                        |
+------+-------------+---------+-------+---------------+---------+---------+-------+------+------------------------------+
|    1 | PRIMARY     | tb_user | const | PRIMARY       | PRIMARY | 4       | const |    1 |                              |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
|    2 | SUBQUERY    | NULL    | NULL  | NULL          | NULL    | NULL    | NULL  | NULL | Select tables optimized away |
+------+-------------+---------+-------+---------------+---------+---------+-------+------+------------------------------+
```

三个看点：

1. **id=2 先执行**（id 大的先）——子查询先算出常量，外层再拿常量查主键；
2. **id=2 全 NULL**：`MAX(id)` 触发了 `Select tables optimized away`——主键 B+Tree 最右端
   叶子节点就是最大值，优化器直接读索引端点，**连表都不用扫，O(1)**；
3. 外层 type=const：子查询结果是运行期常量，等值查主键命中 1 行。

> 💡 启示：能用 `MAX(id)` 就别写 `ORDER BY id DESC LIMIT 1`（后者要真扫索引）。

### 实测：不访问表

```sql
EXPLAIN SELECT 1 + 1;
```

```
type = NULL，table = NULL，Extra = No tables used
```

## 五、select_type：SELECT 的"身份"

| 值 | 含义 |
|---|---|
| SIMPLE | 简单查询，无子查询/UNION |
| PRIMARY | 最外层 SELECT |
| SUBQUERY | WHERE 中的非相关子查询 |
| DEPENDENT SUBQUERY | 相关子查询，外层每行执行一次（**性能杀手**） |
| DERIVED | FROM 里的派生表 |
| UNION / UNION RESULT | UNION 的第二段 / 去重结果 |

实测：`where profession = (select ... where id = 100)` → PRIMARY + SUBQUERY，
子查询 type=const（主键等值），外层 ALL（profession 未单独建索引）。

## 六、type ⭐ 访问类型天梯

从好到坏（**MariaDB 实测有效的等级**）：

```
NULL > const > eq_ref > ref > range > index > ALL
```

> ⚠️ **MariaDB 与 MySQL 的差异**：MySQL 8.0 在表只有 1 行时显示 system 级，
> **MariaDB 未实现该优化**。实测单行表：

```sql
CREATE TABLE t_one (id INT PRIMARY KEY, name VARCHAR(10));
INSERT INTO t_one VALUES (1, 'a');
EXPLAIN SELECT * FROM t_one;
-- type = ALL, rows = 1  ← 不是 system！MariaDB 特有行为
```

### const —— 主键 / 唯一索引等值，最多 1 行

```sql
EXPLAIN SELECT * FROM tb_user WHERE id = 100;
```

```
type=const | key=PRIMARY | key_len=4  | rows=1
```

```sql
EXPLAIN SELECT * FROM tb_user WHERE phone = '13000000100';
```

```
type=const | key=phone    | key_len=46 | rows=1
```

### eq_ref —— JOIN 被驱动表走主键/唯一索引

```sql
EXPLAIN SELECT u.* FROM tb_user u, tb_user u2 WHERE u.id = u2.id AND u.id < 5;
```

```
+------+-------------+-------+--------+---------------+---------+---------+-----------+------+-------------+
| id   | select_type | table | type   | possible_keys | key     | key_len | ref       | rows | Extra       |
+------+-------------+-------+--------+---------------+---------+---------+-----------+------+-------------+
|    1 | SIMPLE      | u     | range  | PRIMARY       | PRIMARY | 4       | NULL      | 4    | Using where |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
|    1 | SIMPLE      | u2    | eq_ref | PRIMARY       | PRIMARY | 4       | demo.u.id | 1    |             |
+------+-------------+-------+--------+---------------+---------+---------+-----------+------+-------------+
```

- 两行 **id 相同 → 从上往下**执行：先扫 u（4 行），再对每行去 u2 精确找；
- u2 的 **eq_ref**：外层每 1 行精确命中 ≤1 行（主键），区别于 ref 的"可能多行"；
- **ref 列 = `demo.u.id`**：显示 u2 用外层哪一列做索引查找——分析 JOIN 数据流全靠它；
- 逗号写法与 `JOIN ON` 执行计划完全相同（内连接两种写法等价），但推荐显式 JOIN。

### ref —— 非唯一索引等值

```sql
EXPLAIN SELECT * FROM tb_user WHERE name = 'user1';
-- type=ref, key=idx_user_name2, key_len=202, rows=1, Using index condition
```

### range —— 索引范围扫描

```sql
EXPLAIN SELECT * FROM tb_user WHERE id BETWEEN 100 AND 200;
-- type=range, key=PRIMARY, rows=101（≈区间长度）

EXPLAIN SELECT * FROM tb_user WHERE name LIKE 'abc%';
-- type=range（前缀匹配 = 范围扫描，见《索引失效（一）》）

EXPLAIN SELECT * FROM tb_user WHERE age < 30;
-- type=ALL！age 列没有索引——范围条件不会自动有索引，没索引就全扫
```

### index —— 扫整棵索引树

```sql
EXPLAIN SELECT id FROM tb_user;
```

```
+------+-------------+---------+-------+---------------+----------------+---------+------+--------+-------------+
| id   | select_type | table   | type  | possible_keys | key            | key_len | ref  | rows   | Extra       |
+------+-------------+---------+-------+---------------+----------------+---------+------+--------+-------------+
|    1 | SIMPLE      | tb_user | index | NULL          | idx_createtime | 6       | NULL | 496694 | Using index |
+------+-------------+---------+-------+---------------+----------------+---------+------+--------+-------------+
```

**两个反直觉**：

1. 无 WHERE 条件，优化器却选 idx_createtime 而非主键——二级索引叶子节点本身存着主键值，
   **扫一棵更"瘦"的索引树就能拿到全部 id**（datetime 只有 6 字节）；
2. Extra=Using index：**type=index ≠ 覆盖索引**！type=index 是"扫描方式"，
   `Using index` 才是"覆盖索引（不回表）"，两者可独立出现，初学最易混淆。

### ALL —— 全表扫描

```sql
EXPLAIN SELECT * FROM tb_user WHERE name LIKE '%abc%';
-- type=ALL, rows=496694
```

**性能底线：核心 SQL 的 type 至少 range，最好 ref 及以上。**

## 七、key_len ⭐ 判断"复合索引用了几列"

### 计算规则

```
定长：int=4；tinyint=1；char(n)=n×字符宽
变长：varchar(n) = n×字符宽 + 2（长度前缀）
可空：每列再加 1 字节（NOT NULL 则不加）
utf8mb4 每字符 4 字节；utf8 每字符 3 字节
```

**公式范例（实测验证）**：

| 字段 | 类型 | key_len |
|---|---|---|
| id | int NOT NULL | **4** |
| phone | varchar(11) NOT NULL | 11×4+2 = **46** |
| name | varchar(50) NOT NULL | 50×4+2 = **202** |

## 八、联合索引最左前缀的量化实证 ⭐⭐⭐

本篇核心实验。表上已有联合索引 `idx_user_pro_age_sta(profession, age, status)`：

```sql
EXPLAIN SELECT * FROM tb_user WHERE profession='软件工程' AND age=25 AND status='1';
```

```
type=ref | key=idx_user_pro_age_sta | key_len=54 | ref=const,const,const | rows=2061
```

```sql
EXPLAIN SELECT * FROM tb_user WHERE profession='软件工程' AND age=25;
```

```
type=ref | key=idx_user_pro_age_sta | key_len=49 | ref=const,const       | rows=4178
```

```sql
EXPLAIN SELECT * FROM tb_user WHERE age=25 AND status='1';
```

```
type=ALL | key=NULL | possible_keys=NULL | rows=496694   ← 索引完全失效
```

### 三组结果汇总

| WHERE 条件 | key_len | rows | 结论 |
|---|---|---|---|
| 三列全匹配 | **54** | 2061 | 三列全用上 |
| 前两列 | **49** | 4178 | 只用前两列 |
| 跳过第一列 | **NULL** | 496694 | **最左前缀中断，失效** |

### key_len=54 的计算拆解

```
profession varchar(11) NULL：11×4 + 2(长度前缀) + 1(NULL标记) = 47
age       tinyint      NULL：1           + 1(NULL标记)       =  2
status    char(1)      NULL：1×4         + 1(NULL标记)       =  5
──────────────────────────────────────────────
三列 = 54 ✓    两列 = 49 ✓
```

> 💡 每个允许 NULL 的字段都多 1 字节——**确定不为空的字段尽量 NOT NULL** 能省索引空间。

### 三个结论

1. **rows 等比递减验证筛选漏斗**：496694 → 4178 → 2061，联合索引每列都是一次过滤；
2. **跳过最左列时连 possible_keys 都是 NULL**——联合索引先按第一列排序，跳过它时
   后面的列在全树里是乱序的，B+Tree 有序性利用不上，与 `'%abc%'` 失效同根因；
3. Extra 为 `Using index condition`（ICP）而非 `Using index`——因为 `SELECT *` 需回表；
   若改为 `SELECT profession, age, status` 可变覆盖索引（可自行验证）。

## 九、Extra 常见状态速查

| Extra | 含义 | 好坏 |
|---|---|---|
| Select tables optimized away | 聚合直接读索引端点 | ✅ 极优（本次 MAX(id) 实测） |
| Using index | 覆盖索引，不回表 | ✅ 最优（本次 select id 实测） |
| Using index condition | ICP 索引条件下推 | ✅ 良（多次实测） |
| Using where | Server 层过滤 | ⚪ 中性 |
| Using temporary | 需临时表 | ❌ 差 |
| Using filesort | 排序未走索引 | ❌ 差（《性能分析二》已消除一例） |
| Using join buffer | JOIN 无索引 | ❌ 差 |

## 十、type 天梯实测汇总表

| 实验 SQL | type | key_len | rows |
|---|---|---|---|
| `select 1+1` | NULL | — | — |
| `id=100` / `phone=...` | **const** | 4 / 46 | 1 |
| JOIN 的被驱动表 | **eq_ref** | 4 | 1 |
| `name='user1'` | **ref** | 202 | 1 |
| `id between 100 and 200` | **range** | 4 | 101 |
| `like 'abc%'` | **range** | 202 | 1 |
| `select id from tb_user` | **index** | 6 | 496694 |
| `like '%abc%'` / `age<30` | **ALL** | — | 496694 |
| 单行表 t_one（MariaDB） | ALL | — | 1 |

## 📌 小结

1. **id**：相同往下走，不同大的先；SUBQUERY 全 NULL + optimized away 是顶配。
2. **type**：MariaDB 天梯从 const 开始（无 system）；index ≠ 覆盖索引。
3. **key_len**：n×4+2+NULL字节，是判断复合索引利用率的唯一量化手段。
4. **最左前缀**：跳过第一列 = 整个索引失效（possible_keys 都为 NULL）。
5. **范围条件不会自动有索引**：age<30 照样 ALL。

## 🔗 待续

- [ ] 索引失效场景（二）：范围查询右侧列失效 / 函数运算 / 隐式转换 / OR 连接非索引列 / NOT 与 IS NOT NULL
- [ ] SQL 优化（insert / 大分页 / join）
