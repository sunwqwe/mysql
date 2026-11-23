# 索引笔记（十二）：单列 vs 联合索引设计——const 直达、index_merge 三道门槛与覆盖复仇局

> 索引使用规则（六）· 收官篇 · 实验数据：demo.tb_user 50 万行（吕布 | lvbu666@163.com）· 环境：Arch Linux + MariaDB 13

## 一、本节核心问题

教科书结论："AND 组合查询，与其建两个单列索引，不如建一个联合索引。"

但 MariaDB 有 index_merge（上次 OR 已实测 union 形态），等值 AND 场景它到底怎么选？本节五轮实验给出了比教科书更精确的答案——**优化器的选型优先级：const > index_merge > 单索引+回表过滤**。

## 二、完整实验命令（可直接照跑）

> ⚠️ 关键前置：先查真实数据再拼查询。黑马数据的 email 是 @163.com / @139.com 等真实格式，**不是 @test.com**；且 name 和 email 必须取自**同一行**，否则触发 Impossible WHERE。

```sql
-- ============ E0 盘点现有索引 ============
SHOW INDEX FROM tb_user;
-- 看点：idx_user_email 的 Non_unique=0（唯一索引）——本节实验的关键变量
-- 收尾提醒：idx_name_9 是上节遗留对照索引，记得清理

-- ============ E0.5 取真实数据（防张冠李戴） ============
SELECT name, email FROM tb_user WHERE name = '吕布';
-- → 吕布 | lvbu666@163.com（吕布自己的邮箱，后面的查询统一用它）

-- ============ E1 单列索引各自的工作情况 ============
EXPLAIN SELECT * FROM tb_user WHERE name  = '吕布';
-- 预期：type=ref | key=idx_user_name2 | key_len=202 | ICP

EXPLAIN SELECT * FROM tb_user WHERE email = 'user499999@test.com';
-- 预期：type=NULL | Impossible WHERE（值不存在，演示"判死"第一形态）

-- ============ E2 错误组合演示：两条件分属两行 ============
SELECT email FROM tb_user LIMIT 1;   -- → 17799990@139.com（user0 的）
EXPLAIN SELECT * FROM tb_user WHERE name = '吕布' AND email = '17799990@139.com';
-- 预期：依然 Impossible WHERE（email 存在但属另一行，"判死"第二形态）

-- ============ E3 ⭐ 正确组合：const 直达 ============
EXPLAIN SELECT * FROM tb_user WHERE name = '吕布' AND email = 'lvbu666@163.com';
-- 预期：type=const | key=idx_user_email | key_len=123 | Extra 干净为空

-- ============ E4 联合索引建+测 ============
CREATE INDEX idx_name_email ON tb_user(name, email);   -- 记耗时

EXPLAIN SELECT * FROM tb_user WHERE name = '吕布';
-- 预期：possible_keys 四个，仍选 idx_user_name2 + ICP

-- ============ E5 ⭐ 覆盖复仇局：联合 vs 前缀 ============
EXPLAIN SELECT id, name, email FROM tb_user WHERE name = '吕布';
-- 预期：key=idx_name_email | Using index（覆盖成功）

EXPLAIN SELECT id, name, email FROM tb_user FORCE INDEX(idx_name_10) WHERE name = '吕布';
-- 预期：key_len=42 | Using where（前缀永不覆盖）

-- ============ E6 联合索引清理 ============
DROP INDEX idx_name_email ON tb_user;

-- ============ 补测：普通索引请不动 intersect ============
CREATE INDEX idx_email_normal ON tb_user(email);
EXPLAIN SELECT * FROM tb_user WHERE name = '吕布' AND email = 'lvbu666@163.com';
-- 预期：possible_keys 五个，仍选唯一索引 type=const
DROP INDEX idx_email_normal ON tb_user;   -- 测完即删

-- ============ 终局清理 ============
DROP INDEX idx_name_9 ON tb_user;
```

## 三、实验 E1 解读：EXPLAIN 天梯第 13 格——type=NULL

`Impossible WHERE noticed after reading const tables` 不是报错，是优化器在**优化阶段就宣判死刑**：

1. email 有唯一索引 → 等值条件升级为 **const 表查找**；
2. 优化器启动预读这张 const 表 → **查无此行**（该邮箱根本不存在）；
3. 条件永远不可能成立 → 零扫描返回。

**天梯新增一格：`NULL < const < eq_ref < ... < ALL`——NULL 比 const 还快，快到不用访问任何东西。**

## 四、实验 E2 解读：唯一索引的行级交叉验证

E2 的 email **存在**，但优化器两步 const 查找后交叉验证：查到的行 name 不是吕布 → 两条件无法在同一行同时成立 → 判死。

**两 Impossible 对比**：

| 场景 | 判死原因 |
|---|---|
| E1：email 值不存在 | const 查找查无此行 |
| E2：email 存在但属另一行 | 两条件分属两行，交叉验证失败 |

**⭐ 新知识点：唯一索引不只防重复，还给优化器免费提供"优化阶段行级事实核查"能力**——普通索引有重复值，const 查找无法保证拿到唯一那行，做不了这种验证。

## 五、实验 E3 ⭐：const 直达，index_merge 被短路

四个 possible_keys 摆桌（idx_user_email / idx_user_name2 / idx_name_10 / idx_name_9），优化器一票选**唯一索引**：

1. 唯一索引查 email → 最多一行 → 整表升级 const；
2. name 条件在这唯一一行上核对 → 成立；
3. type=const，Extra 连 Using where 都没有——**过滤在优化阶段就完成了**。

**intersect 没出场**：index_merge 是"两索引各查一遍、主键集合求交"的迂回方案，有唯一索引直达车，谁坐换乘。**优先级：const > index_merge。**

### key_len=123 对账（可空列 +1 首次实测）

```
email: varchar(30) utf8mb4 可空
30 × 4 + 2（变长）+ 1（NULL 标记）= 123  ✅ 分毫不差
```

key_len 三要素（长度×字符集、变长+2、可空+1）全部到齐。

## 六、实验 E4 解读：等值场景优化器不买联合索引的账

建了 idx_name_email 后，`WHERE name='吕布'` 有 4 个 possible_keys，优化器依然选 **idx_user_name2（202 字节）+ ICP**。

规律延续：**等值窄查询下，优化器偏爱完整存储、有 ICP 的索引**。联合索引能服务此查询（最左前缀）但没被选中——单列 vs 联合的胜负手不在等值速度，在空间与覆盖。

## 七、实验 E5 ⭐：覆盖复仇局——联合 vs 前缀正面对撞

同一查询、同一 SELECT 列表：

| 索引 | key_len | Extra | 结果 |
|---|---|---|---|
| 联合 idx_name_email | 202 | **Using index** | ✅ 免回表 |
| 前缀 idx_name_10 | 42 | Using where | ❌ 回表 |

**上节"前缀索引永不覆盖"与本节"联合索引可覆盖"同框对撞**：联合里 email 是完整存储的（区别于前缀截断），id（二级叶子自带）+ name（key）+ email（完整列）全在索引里，Using index 顺利出现。

**细节**：联合总宽 202+202=404，key_len 只报 202——**key_len 只统计实际参与查找的 key 部分**，email 是"顺路捎上"的被覆盖列，不参与查找就不计入。

## 八、补测解读：普通索引也请不动 intersect

唯一索引和普通索引功能等价，但 const > ref，普通版建了也是陪跑（possible_keys 五个全在，依然选唯一索引）。**只要等值条件里有唯一索引可达路径，merge 永远不被考虑——不管摆几个普通索引候选。**

## 九、index_merge 的三道门槛（intersect 出场条件总账）

| 门槛 | 实验状态 |
|---|---|
| ① 两条件都必须真有行 | E1 email 不存在 ✗ → Impossible WHERE |
| ② 两条件必须属于同一行 | E2 user0 邮箱配吕布 ✗ → Impossible WHERE |
| ③ 不能有更高优先级路径（const） | E3、补测 email 是唯一索引 ✗ → const 直达 |

**intersect 要出场：两个普通单列索引 + 两条件真实同在一行 + 无唯一索引捷径。** 条件苛刻到这地步，恰好解释了教科书为什么教"与其赌 merge，不如直接建联合索引"——**merge 是优化器的保底手段，不是设计方案**。

## 十、教科书的精确化版本

黑马的"AND 双单列不如一个联合"，补全前置条件后：

| 场景 | 优化器选择 |
|---|---|
| 两条件中有一个唯一索引可达 | const 直达，联合/merge 都不必 |
| 两个都是普通单列索引 | 才考虑 index_merge intersect 或单索引+回表过滤 |
| 高频组合查询 | 联合索引仍是正解（覆盖 + 无 merge 开销） |

## 十一、结论卡

1. EXPLAIN 天梯第 13 格：**type=NULL（Impossible WHERE）**，零扫描返回；
2. Impossible WHERE 有两种死法：值不存在 / 条件分属两行；
3. **唯一索引 = 防重复 + 优化阶段行级事实核查**（const 表交叉验证）；
4. 优化器选型优先级：**const > index_merge > 单索引+回表**；
5. **前缀索引永不覆盖，联合索引可以覆盖**——覆盖的分界线是"完整存储 vs 截断存储"；
6. key_len 只算实际参与查找的部分，被覆盖的"顺路列"不计入；
7. 可空列 key_len +1 实测入账（123 = 30×4+2+1）。

## 十二、实验账目总表

| 实验 | 结果 | 结论 |
|---|---|---|
| E0 盘点 | idx_name_9 未清、idx_user_email 是唯一索引 | 唯一索引成为关键变量 |
| E0.5 取真实数据 | 吕布 \| lvbu666@163.com | 防 name/email 张冠李戴 |
| E1 email 单查 | type=NULL，Impossible WHERE | 天梯第 13 格；判死第一形态 |
| E2 错误组合 AND | Impossible WHERE（属另一行） | 判死第二形态；唯一索引交叉验证 |
| E3 正确组合 AND | type=const，key_len=123 | const 直达短路 merge；可空列 +1 |
| E4 联合索引建+测 | 仍选全索引 + ICP | 等值窄查询偏好完整存储 + ICP |
| E5 覆盖对比 ⭐ | 联合 Using index vs 前缀 Using where | 永不覆盖 vs 可覆盖正面对撞 |
| 补测普通索引 | 五候选仍选唯一索引 const | const > index_merge 优先级实锤 |

## 十三、清理

```sql
DROP INDEX idx_name_9 ON tb_user;          -- 对照索引
-- idx_name_email、idx_email_normal 已清 ✅
```

> 下一篇：索引设计原则要点合订 + 《索引章收官总纲》——索引使用规则六节全部闭环 🎉
