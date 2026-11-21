# 索引笔记（九）：覆盖索引与 SQL 提示

> 接《索引笔记（八）：最左前缀法则》。本篇两块内容：
> ① 覆盖索引——实测 3.1 倍收益 + 覆盖的脆弱性；
> ② SQL 提示（USE/IGNORE/FORCE INDEX）——实测 USE 可被无视、
> FORCE 强制走索引反而慢 28%（强制的代价实锤）。
> 环境：MariaDB 13，demo.tb_user，500,025 行。
> ⚠️ 踩坑实录：索引提示放 WHERE 后面会 ERROR 1064。

## 一、覆盖索引：免回表的力量

### 原理回顾（笔记八已埋引子）

回表 = 拿着二级索引里的主键 id → 去聚簇索引再走一遍树找完整行。
每条命中记录都要额外经历一次 B+Tree 查找（随机 IO）。
**查询所需的列全在索引里（索引列+主键）→ 免回表 → Extra: Using index。**

### 实测：同一条 WHERE，只改 SELECT 列表

```sql
-- 回表版
EXPLAIN SELECT * FROM tb_user WHERE profession = '软件工程' AND age = 23;
-- 覆盖版
EXPLAIN SELECT id, profession, age FROM tb_user WHERE profession = '软件工程' AND age = 23;
-- 聚合也能覆盖
EXPLAIN SELECT COUNT(*) FROM tb_user WHERE profession = '软件工程' AND age = 23;
-- 破坏覆盖：多要一个索引外的列
EXPLAIN SELECT id, profession, age, name FROM tb_user WHERE profession = '软件工程' AND age = 23;
```

### 实测汇总 ⭐

| 实验 | type | rows | Extra | 实测耗时 |
|---|---|---|---|---|
| SELECT \*（回表） | ref | 4314 | Using index condition | **0.0235s** |
| SELECT 三列（覆盖） | ref | 4314 | **Using index** | **0.0076s** |
| COUNT(\*)（覆盖） | ref | 4314 | Using where; **Using index** | — |
| 多要 name（破坏） | ref | 4314 | Using index condition | — |

**覆盖收益 ≈ 3.1 倍**（4314 次回表 vs 0 次）。命中 12.5 万行的场景差距会更悬殊
——覆盖索引收益与命中行数成正比。

**三条子结论**：
1. 聚合函数（COUNT）也能享受覆盖索引——生产 COUNT 优化常用手段；
2. **覆盖很脆弱**：SELECT 列表多加一个索引外的列（name），Using index 立即消失；
3. 上一节 25% 占比场景（ALL vs 覆盖版 ref）的反转，本篇从"现象"升级为"原理"。

## 二、SQL 提示：给优化器递纸条

### 语法 ⚠️ 踩坑实录

**提示必须紧跟表名，放在 WHERE 之前**——放在 WHERE 后面直接
ERROR 1064（本次实验四条全翻车在此）：

```sql
-- ❌ 错误：ERROR 1064 near 'USE INDEX (...)'
SELECT * FROM tb_user WHERE profession = '软件工程' USE INDEX (idx_user_pro_age_sta);
-- ✅ 正确：口诀"提示贴表名，WHERE 前面站"
SELECT * FROM tb_user USE INDEX (idx_user_pro_age_sta) WHERE profession = '软件工程';
```

### 三张牌的语气

| 提示 | 语法位置 | 语气 |
|---|---|---|
| USE INDEX | 表名后 | 建议：优先考虑，**可无视** |
| IGNORE INDEX | 表名后 | 别用这个索引 |
| FORCE INDEX | 表名后 | 强制：再贵也必须用 |

### 实测 1：USE INDEX 被无视 ⭐

```sql
EXPLAIN SELECT * FROM tb_user USE INDEX (idx_user_pro_age_sta)
WHERE profession = '软件工程';
-- 实测：type=ALL | possible_keys=idx_user_pro_age_sta | key=NULL
```

语法正确、索引可用（possible_keys 有值），优化器**依然选全表扫**——
USE 只是建议，25% 占比场景下它算完账还是拒绝。

### 实测 2：FORCE 强制生效，但反而慢 28% ⭐⭐（本篇灵魂数据）

```sql
SELECT * FROM tb_user WHERE profession = '软件工程';                                -- 0.447s
SELECT * FROM tb_user FORCE INDEX (idx_user_pro_age_sta)
WHERE profession = '软件工程';                                                      -- 0.571s
```

强制走索引 = 125,033 行全部"索引定位 → 回表"，12.5 万次随机 IO
碾压顺序扫全表。**优化器上次拒绝走索引是正确的；FORCE 把错误决策
强加给它，代价 +0.124s（+28%）。**

### SQL 提示的正确打开方式（结论）

提示是**纠偏工具**（优化器被过时统计信息误导时的逃生舱），
**不是性能优化工具**。默认信优化器，除非能用 COUNT 等
亲手证明它错了——而本系列恰好实测证明：它算得很对。

## 三、踩坑与杂项记录

1. 索引提示位置错误 → ERROR 1064（四连翻车，见上文口诀）；
2. 粘贴断行跑出半截 SQL（`from tb_user profession = ...` 丢 WHERE），
   报错秒败进 PROFILES——失败查询也计时，别误读；
3. A3 手敲的 `'软件工 程'` 疑似多空格仍返回 4314 行——待验证
   collation 是否忽略空格（`SELECT '软件工 程'='软件工程';`）；
4. 待办遗留：ANALYZE TABLE 后 rows 估算修正、实验 2b 补档（前篇）。

## 四、结论汇总

1. **覆盖索引**：SELECT 列表 ⊆ 索引列+主键 → 免回表，实测 3.1 倍；
   聚合可用；加一个索引外列就破坏；
2. **USE INDEX = 建议（可无视），FORCE INDEX = 强制（可致慢 28%）**；
3. **提示语法紧跟表名**；
4. **优化器默认是对的**——提示只在它能被证明错了时才出手。

## 📌 待办

- [ ] ANALYZE TABLE + rows 估算修正验证（前篇遗留）
- [ ] FORCE 的 EXPLAIN 留档（预期 type=ref）
- [ ] 下一节：范围查询（key_len 截断现象）→ 前缀索引 → 索引设计收尾
