# 索引笔记（八）：使用规则（二）——最左前缀法则

> 接《索引笔记（七）：验证索引效率》（实测加速比 718 倍）。
> 本篇实测联合索引 idx_user_pro_age_sta 的使用规则，并挖出两个
> 视频里没有的进阶知识点：**possible_keys vs key（能用≠值得用）**
> 和 **覆盖索引扭转成本决策**。
> 环境：MariaDB 13，demo.tb_user，实测 500,025 行。

## 一、原理：联合索引是一栋"先 profession 再 age 再 status"的楼

idx_user_pro_age_sta 的 B+Tree 排序规则是层层嵌套的：

```
先按 profession 排序
profession 相同时，再按 age 排序
age 也相同时，最后按 status 排序
```

只有确定了左边（前列）的值，右边（后列）的排序才有意义。
"所有 age=23 的行"散落在每个 profession 段的内部，段间不连续——
**跳过最左列，索引结构上就无从定位**（possible_keys 直接为 NULL）。

## 二、实验矩阵与实测结果（6 组 EXPLAIN）

```sql
EXPLAIN SELECT * FROM tb_user WHERE profession = '软件工程';                              -- 实验1
EXPLAIN SELECT * FROM tb_user WHERE profession = '软件工程' AND age = 23;                 -- 实验2
EXPLAIN SELECT * FROM tb_user WHERE profession = '软件工程' AND age = 23 AND status='6';  -- 实验3
EXPLAIN SELECT * FROM tb_user WHERE age = 23;                                            -- 实验4
EXPLAIN SELECT * FROM tb_user WHERE profession = '软件工程' AND status = '6';             -- 实验5
EXPLAIN SELECT * FROM tb_user WHERE age = 23 AND profession = '软件工程' AND status='6';  -- 实验6
```

### 实测汇总表 ⭐

| 实验 | WHERE 条件 | type | possible_keys | key | key_len | rows |
|---|---|---|---|---|---|---|
| 1 | profession | **ALL** | idx_user_pro_age_sta | **NULL** | NULL | 497140 |
| 2 | pro+age | ref | 同上 | 同上 | **49** | 4314 |
| 3 | pro+age+status | ref | 同上 | 同上 | **54** | **1** |
| 4 | **只有 age** | **ALL** | **NULL** | NULL | NULL | 497140 |
| 5 | pro+status（断中间） | **ALL** | idx_user_pro_age_sta | **NULL** | NULL | 497140 |
| 6 | 乱序三列 | ref | 同上 | 同上 | **54** | 1 |

### 逐条解读

**实验 2→3：逐列递进，key_len 精确对账**

key_len 49 → 54，status 贡献 5 字节。真实表结构（SHOW CREATE TABLE）下的
记账公式（utf8mb4，每字符 4 字节）：

| 列 | 类型 | 计算 | key_len |
|---|---|---|---|
| profession | varchar(11) NULL | 11×4 + 2前缀 + 1 NULL | 47 |
| age | tinyint unsigned NULL | 1 + 1 NULL | 2 |
| status | char(1) NULL | 1×4 + 1 NULL（定长无前缀） | 5 |

三列合计 47+2+5 = 54 ✅ 与实测完全一致。
**过滤能力阶梯：4314 行（2 列）→ 1 行（3 列），每加一列掉一个数量级。**

**实验 4 vs 实验 1/5：两种"不走索引"本质不同** ⭐⭐

| | 实验 4（只查 age） | 实验 1/5（pro 或 pro+status） |
|---|---|---|
| possible_keys | **NULL**（结构上用不了） | **有值**（能用） |
| key | NULL | NULL（**选择不用**） |
| 本质 | 最左前缀失效，无从定位 | **成本决策：用了反而不划算** |

**实验 6：SQL 编写顺序不影响索引使用**——WHERE 条件写成 age 在前
也照样走满 3 列（key_len=54），优化器自动重排条件匹配索引列序。

**实验 5：部分失效不是全失效**——status 断层后，索引只能用 profession
一列，可筛选的行数和实验 1 一样多，同样触发成本决策放弃使用。

## 三、成本决策实证：为什么优化器"能用却不用" ⭐⭐

```sql
SELECT COUNT(*) FROM tb_user;                                -- 500,025
SELECT COUNT(*) FROM tb_user WHERE profession='软件工程';     -- 125,033
-- 占比 = 125033 / 500025 ≈ 25.0%（整整四分之一！）
```

走索引 vs 全表扫的真实代价：

```
走索引：125,033 次索引定位 + 125,033 次回表（随机 IO，最伤）
全表扫：497,140 行顺序读一遍（顺序 IO，一次流过）
```

四分之一的行都要回表时，随机 IO 成本碾压顺序扫——**优化器放弃索引是
正确决策，不是 bug**。视频（MySQL 8）里同样 SQL 走了 ref，因为
数据分布不同。结论：**possible_keys 说"能不能用"，key 说"值不值得用"，
两者都 NULL 才是索引彻底失效。**

## 四、翻盘实验：改 SELECT 列表，执行计划反转 ⭐⭐（下一节引子）

```sql
EXPLAIN SELECT id, profession FROM tb_user WHERE profession = '软件工程';
```

```
type=ref | key=idx_user_pro_age_sta | key_len=47 | Extra: Using index
```

同一条 WHERE，只把 SELECT \* 改成 id, profession——执行计划从 ALL 变 ref！
原理：二级索引叶子节点本身就存着"索引列 + 主键 id"，这两个列索引里都有，
**不需要回表**，125,033 条记录顺序扫完即出结果，随机 IO 消失，成本骤降。
`Extra: Using index` 就是覆盖索引生效的标志——下一节专题展开。

## 五、踩坑与附带发现

1. **EXPLAIN 的 rows 是估算值**：覆盖索引实验估 260,898 行，实际
   COUNT 125,033，高了约 2 倍——批量灌数后索引统计信息过时。
   `ANALYZE TABLE tb_user;` 可刷新统计，让估算更准；
2. **引号里的空格是隐患**：实验 6 曾写成 `' 软件工程'`（前导空格），
   EXPLAIN 照样给出 rows=1 的"完美计划"——EXPLAIN 只规划不验证值，
   真执行会返回空结果。字符串字面量要逐字符核对；
3. **key_len 必须按真实表结构算**：按视频假设的 varchar(50)+int 估算
   得 203/208/215，实际 varchar(11)+tinyint 是 47/49/54——
   `SHOW CREATE TABLE` 是对账的唯一依据；
4. 索引清单修正：phone（唯一）、idx_user_email（唯一）、idx_user_name2、
   idx_user_pro_age_sta、idx_createtime。

## 六、结论

1. **最左前缀法则**：联合索引必须从最左列开始、连续匹配才逐列生效；
   跳过最左列=完全用不了（possible_keys=NULL）；断中间列=部分失效；
2. **possible_keys ≠ key**：能用但占比太高时优化器会算账后放弃——
   回表随机 IO 是"值得用"的判断核心；
3. **覆盖索引能扭转成本决策**：SELECT 只要索引里有的列，免回表，
   ALL 也能变 ref；
4. SQL 编写顺序不影响，索引列序才影响。

## 📌 待办

- [ ] ANALYZE TABLE 后重跑覆盖索引 EXPLAIN，观察 rows 估算修正
- [ ] 下一节：范围查询（profession+age>xx 的 key_len 变化）
- [ ] 覆盖索引专题（本篇第四节已埋引子）
- [ ] 老账：《索引失效（一）》笔记文件补传；性能分析（四）3 条待补实验
