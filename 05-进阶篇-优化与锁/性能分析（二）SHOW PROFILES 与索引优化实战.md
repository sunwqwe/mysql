# 性能分析（二）：SHOW PROFILES 与索引优化实战

> 接《性能分析（一）：执行频次与慢查询日志》，本篇用 `SHOW PROFILES` + `EXPLAIN` 走一遍完整的优化闭环：
> 定位慢 SQL → 拆解耗时阶段 → 诊断原因 → 建索引 → 对比验证。
> 实验中同时验证了 **LIKE 通配符位置对索引的影响**（对应"索引失效场景"知识点）。

## 🧪 实验环境

- 服务端：Arch Linux 虚拟机 + MariaDB 13
- 数据库：`demo`，表 `tb_user`（9 字段，含 `name`/`phone`/`email`/`profession`/`createtime` 等）
- 初始数据：空表 + 2 个唯一索引（phone/email）+ 2 个普通索引（name/profession）

```sql
DESC tb_user;
```

```
+------------+---------------------+------+-----+---------+----------------+
| Field      | Type                | Null | Key | Default | Extra          |
+------------+---------------------+------+-----+---------+----------------+
| id         | int(11)             | NO   | PRI | NULL    | auto_increment |
| --- | --- | --- | --- | --- | --- |
| name       | varchar(50)         | NO   | MUL | NULL    |                |
| phone      | varchar(11)         | NO   | UNI | NULL    |                |
| email      | varchar(30)         | YES  | UNI | NULL    |                |
| profession | varchar(11)         | YES  | MUL | NULL    |                |
| age        | tinyint(3) unsigned | YES  |     | 1       |                |
| gender     | char(1)             | YES  |     | NULL    |                |
| status     | char(1)             | YES  |     | NULL    |                |
| createtime | datetime            | YES  |     | NULL    |                |
+------------+---------------------+------+-----+---------+----------------+
```

## 1️⃣ 造数据：存储过程批量插入 50 万行

```sql
DELIMITER $$
CREATE PROCEDURE insert_users(IN n INT)
BEGIN
  DECLARE i INT DEFAULT 0;
  WHILE i < n DO
    INSERT INTO tb_user(name, phone, email, profession, age, gender, status, createtime)
    VALUES (
      CONCAT('user', i, IF(i % 100 = 0, 'abc', '')),   -- 每 100 条混入一个含 abc 的
      CONCAT('13', LPAD(i, 9, '0')),
      CONCAT('u', i, '@test.com'),
      ELT(1 + FLOOR(RAND()*4), '软件工程', '会计学', '英语', '美术'),
      18 + FLOOR(RAND()*30),
      IF(RAND()>0.5, '1', '0'),
      IF(RAND()>0.5, '1', '0'),
      NOW() - INTERVAL FLOOR(RAND()*10000) DAY
    );
    SET i = i + 1;
  END WHILE;
END$$
DELIMITER ;

CALL insert_users(500000);
DROP PROCEDURE insert_users;
```

```
Query OK, 500000 rows affected (4 min 12.008 sec)
```

> ⏱ 4 分钟的原因：逐行 INSERT，每行都要同步维护 4 棵 B+Tree（主键 + 2 个唯一索引 + 2 个普通索引中的非冗余项）。
> 下次可改用笛卡尔积一次性插入，速度能快一个数量级。

## 2️⃣ SHOW PROFILES 定位慢 SQL

```sql
SET profiling = 1;
SELECT * FROM tb_user WHERE name LIKE '%abc%' ORDER BY createtime DESC LIMIT 20;
SHOW PROFILES;
```

```
+----------+------------+---------------------------------------------------------------------------------+
| Query_ID | Duration   | Query                                                                            |
+----------+------------+---------------------------------------------------------------------------------+
| 1        | 0.53586217 | select * from tb_user where name like '%abc%' order by createtime desc limit 20 |
+----------+------------+---------------------------------------------------------------------------------+
```

**总耗时 0.536 秒**。注意几个易踩的坑：

- `profiling` 默认 OFF，必须在**同一会话**内 `SET profiling = 1` 后执行目标 SQL；
- 默认只保留最近 15 条，可用 `SET profiling_history_size = 100` 扩容；
- profile 会记录会话内**所有语句**，包括失败的、`show tables` 这类元数据查询——不要拿失败语句的 Query_ID 去分析（其 profile 只有 Starting / Freeing items 几个阶段，无意义）。

## 3️⃣ EXPLAIN 诊断

```sql
EXPLAIN SELECT * FROM tb_user WHERE name LIKE '%abc%' ORDER BY createtime DESC LIMIT 20;
```

```
+------+-------------+---------+------+---------------+------+---------+------+--------+-----------------------------+
| id   | select_type | table   | type | possible_keys | key  | key_len | ref  | rows   | Extra                       |
+------+-------------+---------+------+---------------+------+---------+------+--------+-----------------------------+
|    1 | SIMPLE      | tb_user | ALL  | NULL          | NULL | NULL    | NULL | 496771 | Using where; Using filesort |
+------+-------------+---------+------+---------------+------+---------+------+--------+-----------------------------+
```

**问题清晰了：**

| 字段 | 值 | 含义 |
|---|---|---|
| type | **ALL** | 全表扫描，扫 49.7 万行 |
| key | **NULL** | 没有任何索引被使用 |
| rows | **496771** | 预估扫描行数 ≈ 全表 |
| Extra | Using where; **Using filesort** | Server 层逐行过滤 + 排序阶段未走索引 |

## 4️⃣ 顺带实验：LIKE 通配符位置对索引的影响

```sql
EXPLAIN SELECT * FROM tb_user WHERE name LIKE 'abc%';
```

```
+------+-------------+---------+-------+----------------+----------------+---------+------+------+-----------------------+
| id   | select_type | table   | type  | possible_keys  | key            | key_len | ref  | rows | Extra                 |
+------+-------------+---------+-------+----------------+----------------+---------+------+------+-----------------------+
|    1 | SIMPLE      | tb_user | range | idx_user_name2 | idx_user_name2 | 202     | NULL |    1 | Using index condition |
+------+-------------+---------+-------+----------------+----------------+---------+------+------+-----------------------+
```

```sql
EXPLAIN SELECT * FROM tb_user WHERE name LIKE '%abc%';
```

```
+------+-------------+---------+------+---------------+------+---------+------+--------+-------------+
| id   | select_type | table   | type | possible_keys | key  | key_len | ref  | rows   | Extra       |
+------+-------------+---------+------+---------------+------+---------+------+--------+-------------+
|    1 | SIMPLE      | tb_user | ALL  | NULL          | NULL | NULL    | NULL | 496694 | Using where |
+------+-------------+---------+------+---------------+------+---------+------+--------+-------------+
```

### 结论：最左前缀原则在 LIKE 上的体现

| 写法 | 是否走索引 | type | 原因 |
|---|---|---|---|
| `LIKE 'abc%'` | ✅ | range | 等价于 `name >= 'abc' AND name < 'abd'`，B+Tree 可二分定位区间 |
| `LIKE '%abc'` | ❌ | ALL | 后缀匹配，无法定位起始位置 |
| `LIKE '%abc%'` | ❌ | ALL | 中缀匹配，命中行散落全树各处，无连续区间 |

**规律：只要 `%` 不出现在最前面，就能用上索引。**

- `key_len = 202` 的来历：name 是 varchar(50)，utf8mb4 下每字符 4 字节：50 × 4 = 200，再加 2 字节存长度 = 202，说明索引前缀**全字段都用上了**。
- `Using index condition` = **ICP（索引条件下推）**：LIKE 过滤下推到存储引擎层，在索引上直接判断，不满足的行无需回表。
- 失效的本质：**B+Tree 的有序性只能服务"从最左字符开始匹配"的场景**，一旦放弃最左字符，二分定位就完全失效。

### 中缀搜索的替代方案

| 方案 | 适用场景 |
|---|---|
| FULLTEXT 全文索引 | 长文本搜索，MariaDB 10.0.5+ 支持 `WITH PARSER ngram` 处理中文 |
| 倒排搜索引擎（ES / MeiliSearch） | 数据量大、搜索为高频核心业务 |
| 反向存储 | 只有后缀匹配需求（存 reverse(name)，`%abc` 改写为 `cba%`） |
| 接受全扫 + 应用层缓存 | 低频后台查询 |

## 5️⃣ 优化：给 createtime 加索引消除 filesort

```sql
CREATE INDEX idx_createtime ON tb_user(createtime);
EXPLAIN SELECT * FROM tb_user WHERE name LIKE '%abc%' ORDER BY createtime DESC LIMIT 20;
```

```
+------+-------------+---------+-------+---------------+----------------+---------+------+------+-------------+
| id   | select_type | table   | type  | possible_keys | key            | key_len | ref  | rows | Extra       |
+------+-------------+---------+-------+---------------+----------------+---------+------+------+-------------+
|    1 | SIMPLE      | tb_user | index | NULL          | idx_createtime | 6       | NULL |   20 | Using where |
+------+-------------+---------+-------+---------------+----------------+---------+------+------+-------------+
```

### 优化前后对比

| 维度 | 优化前 | 优化后 |
|---|---|---|
| type | ALL | **index** |
| key | NULL | **idx_createtime** |
| rows | 496771 | **20** |
| Extra | Using where; **Using filesort** | Using where（**filesort 消失**） |

### 优化器为什么放弃 name 条件、改走 createtime 索引？

优化器在两个执行计划之间做了成本估算：

- **计划 A（用 name 索引）**：`%abc%` 用不上 → 退化为全表扫 50 万行 + filesort
- **计划 B（用 createtime 索引）**：沿索引从最大时间往回读，每读一行用 `Using where` 过滤 name，凑够 LIMIT 20 条即停

含 abc 的行约占 1%，平均读 ~2000 行就能凑齐 20 条，**远低于计划 A 的成本**，所以优化器主动选 B。

⚠️ 这个策略有前提：**含目标值的行要足够分散、LIMIT 要足够小**。如果表里几乎没匹配行、或 LIMIT 很大，优化器会回退到 ALL + filesort。EXPLAIN 的 rows 是估算，最终以实际耗时为准。

## 📌 小结

```
SET profiling = 1 → 目标 SQL → SHOW PROFILES 看总耗时
    → EXPLAIN 看访问路径（type/key/rows/Extra）
    → 识别瓶颈（ALL = 全扫；Using filesort = 排序未走索引）
    → 建索引 / 改写 SQL
    → 再 EXPLAIN + 实测耗时，对比验证
```

三个核心收获：

1. **LIKE 索引失效规则**：`%` 在最前面必失效，否则能走索引。
2. **ORDER BY 加索引可消除 filesort**，特别是配合 LIMIT 时优化器会主动"绕道"用排序索引。
3. **优化器的选择基于成本估算**，不是死的"规则"——同一个 SQL 在不同数据分布下可能走不同索引。

## 🔗 待续

- [ ] 索引失效八大场景完整版（最左前缀 / 函数运算 / 隐式转换 / OR 连接非索引列 …）
- [ ] SQL 优化（insert 优化 / 大分页优化 / join 优化）
- [ ] 用 `SHOW PROFILE FOR QUERY n` 拆解 Sending data / Sorting result 阶段占比
