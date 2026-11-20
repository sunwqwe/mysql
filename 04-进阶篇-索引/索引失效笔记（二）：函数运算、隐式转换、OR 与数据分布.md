# 索引失效笔记（二）：函数运算、隐式转换、OR 与数据分布

> 接《索引失效笔记（一）：LIKE 通配符》。五大失效场景全实测，
> 三个结果与视频预期不同（MariaDB 优化器更激进），逐一剖析。
> 环境：MariaDB 13，demo.tb_user，500,025 行。
> ⚠️ 实验中途服务器崩溃重启一次（Server has gone away），见文末故障记录。

## 一、核心原理：失效的本质是破坏有序性

B+Tree 只对"原样存储的值"有序。凡是对列做了加工（函数/运算/类型转换），
都无法在有序树里定位。五大场景全是这一个原理的马甲：

| 场景 | 破坏方式 |
|---|---|
| 列上用函数 | 对索引值再加工 |
| 隐式类型转换 | 字符串列 vs 数字 → 整列转换 |
| OR 连接非索引列 | 一边没索引拖垮整体 |
| IS NOT NULL / != | 不禁止走索引，命中比例决定 |
| 数据分布差 | 能用但不值得用（成本决策） |

## 二、实验 1：函数运算——MariaDB 竟然没失效 ⭐⭐

```sql
-- 1a 等值基准：type=const | key=phone | key_len=46 | rows=1（唯一索引等值=const）
EXPLAIN SELECT * FROM tb_user WHERE phone = '17799990000';
-- 1b 函数写法（预期失效）：
EXPLAIN SELECT * FROM tb_user WHERE substring(phone,1,3) = '177';
-- 1c LIKE 前缀写法：
EXPLAIN SELECT * FROM tb_user WHERE phone LIKE '177%';
```

### 实测结果

| 写法 | type | key | key_len | rows |
|---|---|---|---|---|
| 1a 等值 | const | phone | 46 | 1 |
| 1b substring | **range** | phone | 46 | 24 |
| 1c LIKE '177%' | range | phone | 46 | 24 |

**1b 和 1c 的执行计划完全相同！** MariaDB 优化器把
`substring(phone,1,3)='177'` 自动改写成了区间扫描
（等价于 phone >= '177' AND phone < '178'），函数被"翻译"掉了。

### ⚠️ 重要警告

这是 **MariaDB 特有优化**，MySQL 8 在此步是真失效（视频演示即 ALL）。
**不要依赖**：可移植的写法永远是 LIKE 前缀或等值，
`substring(col,...)=...` 在 MySQL 生产环境就是全表扫。

key_len=46 对账：varchar(11) NOT NULL utf8mb4 = 11×4+2 = 46（NOT NULL 无 NULL 位）。

## 三、实验 2：隐式类型转换——不对称性 ⭐⭐（面试必考）

```sql
-- 2a 字符串列 vs 数字字面量（phone 不加引号）
EXPLAIN SELECT * FROM tb_user WHERE phone = 17799990000;
-- 2b 数字列 vs 字符串字面量（id 加引号）
EXPLAIN SELECT * FROM tb_user WHERE id = '250000';
```

### 实测结果（完美镜像）

| 实验 | type | key | 结论 |
|---|---|---|---|
| 2a | **ALL** | **NULL**（possible_keys=phone 但未采用） | **失效** |
| 2b | const | PRIMARY | 不失效 |

**原理**：字符串列 vs 数字 → 每行的列值都要转成数字（对整列做函数，
有序性破坏 → 失效）；数字列 vs 字符串 → 只转换那一个**常量**（列不动，
索引照走）。

**面试标准答案：转换发生在哪一侧决定生死——常量随便转，列不能转。**

## 四、实验 3：OR 连接——index_merge 惊喜 ⭐⭐

```sql
-- 3a 两边都有索引
EXPLAIN SELECT * FROM tb_user WHERE name = '吕布' OR phone = '17799990000';
-- 3b 一边无索引（gender 无索引）
EXPLAIN SELECT * FROM tb_user WHERE name = '吕布' OR gender = '1';
```

### 实测结果

| 实验 | type | key | Extra |
|---|---|---|---|
| 3a | **index_merge** | idx_user_name2,phone | **Using union(idx_user_name2,phone)** |
| 3b | ALL | NULL | Using where |

3a：**两个索引各查一遍再合并**——name 索引命中 1 行 + phone 索引命中
1 行 = rows 2，key_len=202,46（两列各自记账：name 50×4+2=202 ✅）。
3b：gender 无索引 → 一边拖垮整体，退回全表扫。

**准确表述：不是"OR 失效"，是 OR 的某一边没索引时全盘失效；
两边都有索引时 index_merge(union) 能救。**

## 五、实验 4：IS NOT NULL / IS NULL / != ——都不是"失效" ⭐

```sql
EXPLAIN SELECT * FROM tb_user WHERE phone IS NOT NULL;  -- 4a
EXPLAIN SELECT * FROM tb_user WHERE age IS NULL;        -- 4b
EXPLAIN SELECT * FROM tb_user WHERE gender != '1';      -- 4c
```

### 实测结果与真实原因

| 实验 | type | possible_keys | 真实原因 |
|---|---|---|---|
| 4a | ALL | phone | phone 列 NOT NULL，条件对所有行成立，**命中率 100%，成本决策不走**（同前篇 25% 占比逻辑） |
| 4b | ALL | **NULL** | age **无独立索引**（联合索引第二位，最左前缀用不了）；且实测 NULL 行数=0 |
| 4c | ALL | **NULL** | gender 无索引；实测分布 249,776 vs 250,248（**49.9% vs 50.1% 五五开**），有索引也必然全扫 |

**修正视频结论：IS NOT NULL / != 并不禁止走索引，最终走不走
完全取决于命中比例（在列有索引的前提下）。
谈"失效"之前先 SHOW INDEX 确认列上有没有索引。**

COUNT 附带观察：三条全表扫描耗时 0.490s / 0.150s / 0.816s——
同样的全表扫耗时差 5 倍（缓存噪声），**再次验证判断计划看 type/rows，
别迷信单次耗时**。

## 六、实测结论汇总

1. **函数运算**：MySQL 8 失效；MariaDB 能改写 substring 为 range——
   但可移植写法永远用 LIKE 前缀；
2. **隐式转换**：字符串列 vs 数字 = 失效；数字列 vs 字符串 = 正常。
   记忆点：**常量随便转，列不能转**；
3. **OR**：两边都有索引 → index_merge union 可用；一边没索引 → 全盘 ALL；
4. **IS NOT NULL / !=**：不是失效，是成本决策（命中比例决定）；
5. **谈失效前先 SHOW INDEX**——4b/4c 的 ALL 根本是"无索引可用"，
   和失效无关。

## 七、故障记录：实验中途 Server has gone away ⚠️

- 现象：实验 2a 执行时报 ERROR 2006，客户端自动重连（新连接 id=7）；
- 影响：会话级设置重置（profiling 归零、PROFILES 清空）、旧连接丢失；
  InnoDB 崩溃恢复，**数据无损**；
- 待办：虚拟机 `journalctl -u mariadb` 查崩溃原因，疑似内存不足（OOM），
  灌 100 万行数据前建议把虚拟机内存调至 2G+。

## 📌 待办

- [ ] 查明服务器崩溃原因（journalctl / free -h）
- [ ] 实验 2b 输出重跑留档（本次粘贴损坏）
- [ ] ANALYZE TABLE 后观察 rows 估算修正（前篇遗留）
- [ ] 下一节：SQL 优化（insert 批量插入 / 大分页 / join）或覆盖索引专题
