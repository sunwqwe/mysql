# 第 17 课：索引语法（进阶篇）

> 课程来源：黑马程序员《MySQL数据库入门到精通》进阶篇 · 第 17 集（时长 11:19）
> 主题：创建、查看、删除索引的完整语法
> 所有语句均在 MariaDB 中实际运行验证

---

## 一、索引的分类（语法操作的对象）

写语法前先明确操作对象。按**功能**分四类，另有一个横向扩展形态：

| 索引类型 | 关键字 | 特点 |
|---|---|---|
| 主键索引 | `PRIMARY KEY` | 唯一 + 非空，每表仅一个，建表自动生成 |
| 唯一索引 | `UNIQUE` | 列值不能重复，**允许 NULL** |
| 普通索引 | `INDEX` | 无约束，纯加速 |
| 全文索引 | `FULLTEXT` | 文本关键词检索，倒排索引结构 |
| 联合索引 | 多列组合 | 多列构成**一个**索引，列顺序有讲究 |

> 联合索引不是第五种类型，而是普通/唯一索引的横向扩展——`CREATE INDEX idx ON t(a,b,c)` 与 `CREATE UNIQUE INDEX idx ON t(a,b,c)` 都可以是联合的。

---

## 二、四大核心语法

### 2.1 创建索引

**语法：**

```sql
CREATE [ UNIQUE | FULLTEXT ] INDEX index_name ON table_name (
    index_col_name [ASC | DESC], ...
);
```

- 不加修饰 → 普通索引
- `UNIQUE` → 唯一索引
- `FULLTEXT` → 全文索引
- `ASC/DESC`：索引内排序方向，默认 ASC

**四种类型示例：**

```sql
-- ① 普通索引
CREATE INDEX idx_user_name ON tb_user(name);

-- ② 唯一索引
CREATE UNIQUE INDEX idx_user_email ON tb_user(email);

-- ③ 联合索引（本课重点）
CREATE INDEX idx_user_pro_age_sta ON tb_user(profession, age, status);

-- ④ 全文索引
CREATE FULLTEXT INDEX idx_user_pro ON tb_user(profession);
```

**ALTER TABLE 等价写法（两种都要会）：**

```sql
ALTER TABLE tb_user ADD INDEX idx_user_name(name);
ALTER TABLE tb_user ADD UNIQUE INDEX idx_user_email(email);
ALTER TABLE tb_user ADD INDEX idx_user_pro_age_sta(profession, age, status);
ALTER TABLE tb_user ADD FULLTEXT INDEX idx_user_pro(profession);
```

> **注意**：`CREATE INDEX` 无法创建主键索引。主键只能通过 `CREATE TABLE` 时声明 `PRIMARY KEY`，或 `ALTER TABLE ... ADD PRIMARY KEY` 添加——主键是表结构的一部分，不是独立索引对象。

### 2.2 查看索引

```sql
SHOW INDEX FROM table_name;
-- 加 \G 竖排显示更易读：SHOW INDEX FROM tb_user\G
```

**关键字段解读：**

| 字段 | 含义 |
|---|---|
| `Key_name` | 索引名（PRIMARY 表示主键索引） |
| `Non_unique` | 0 = 唯一索引，1 = 普通索引 |
| `Seq_in_index` | 联合索引中该列的位置，从 1 开始 |
| `Column_name` | 索引列名 |
| `Cardinality` | 索引中不重复值数量的估算值 |
| `Index_type` | BTREE（InnoDB 默认） |
| `Sub_part` | 前缀索引长度，整列索引为 NULL |

### 2.3 删除索引

```sql
DROP INDEX index_name ON table_name;
-- 等价于：
ALTER TABLE table_name DROP INDEX index_name;
```

示例：

```sql
DROP INDEX idx_user_name ON tb_user;
DROP INDEX idx_user_pro_age_sta ON tb_user;  -- 联合索引 3 行一起消失
```

> **联合索引没有"删其中一列"的语法**——整个索引是一个对象，调整列只能删了重建。

---

## 三、四种索引逐一实操

### 3.1 主键索引（建表自动生成）

```sql
CREATE TABLE tb_user (
    id         INT PRIMARY KEY AUTO_INCREMENT COMMENT '主键，自动生成主键索引',
    name       VARCHAR(50) NOT NULL COMMENT '姓名',
    phone      VARCHAR(11) NOT NULL COMMENT '手机号',
    email      VARCHAR(50)          COMMENT '邮箱',
    profession VARCHAR(50)          COMMENT '专业',
    age        INT                  COMMENT '年龄',
    status     CHAR(1)              COMMENT '状态',
    createtime DATETIME             COMMENT '创建时间'
) COMMENT '用户表';
```

### 3.2 唯一索引 + 约束验证

```sql
CREATE UNIQUE INDEX idx_user_email ON tb_user(email);
```

验证约束效果：

```sql
INSERT INTO tb_user(name, phone, email) VALUES ('测试1', '13900000001', 'a@x.com');
-- Query OK, 1 row affected

INSERT INTO tb_user(name, phone, email) VALUES ('测试2', '13900000002', 'a@x.com');
-- ERROR 1062 (23000): Duplicate entry 'a@x.com' for key 'idx_user_email'
```

**结论**：唯一索引双重身份——既是索引（加速查询），又是约束（保证不重复）。

### 3.3 联合索引（本课重点）

```sql
CREATE INDEX idx_user_pro_age_sta ON tb_user(profession, age, status);
SHOW INDEX FROM tb_user;
```

观察到的现象：

| 现象 | 含义 |
|---|---|
| 一个索引出现 3 行 | 联合索引每列各占一行 |
| `Seq_in_index` = 1, 2, 3 | 列顺序 = 建索引时的书写顺序 |
| `Cardinality` 逐列递减 | 第一列区分度最高 |

**核心认知**：联合索引 = 按第一列排序，第一列相同再按第二列排序……这是后续"最左前缀原则"的物理根源。

### 3.4 全文索引

```sql
CREATE FULLTEXT INDEX idx_user_pro ON tb_user(profession);

-- 必须用 MATCH AGAINST，普通 WHERE 走不了全文索引
SELECT * FROM tb_user WHERE MATCH(profession) AGAINST('软件工程');
-- 命中 4 行 ✅

SELECT * FROM tb_user WHERE MATCH(profession) AGAINST('软件');
-- Empty set ❌
```

**中文空集的原因（两个因素叠加）：**

1. 默认分词器按空格/标点切词，"软件工程"被当成一个整体 token，索引里没有"软件"这个词
2. `innodb_ft_min_token_size = 3`，少于 3 字符的词不入索引

**生产建议**：中文全文检索选 Elasticsearch（配 IK 分词器），MySQL/MariaDB 原生 FULLTEXT 对中文基本不可用。

---

## 四、完整演示脚本（可直接运行）

```sql
-- ============================================
-- 第 17 课 索引语法 · 完整演示脚本
-- ============================================

-- 0. 查看初始索引状态（应有 PRIMARY、phone）
SHOW INDEX FROM tb_user;

-- 1. 普通索引：两种写法都练
CREATE INDEX idx_user_name ON tb_user(name);
ALTER TABLE tb_user ADD INDEX idx_user_name2(name);  -- 同列重复建会出 warning
DROP INDEX idx_user_name2 ON tb_user;                -- 清理

-- 2. 唯一索引 + 约束验证
CREATE UNIQUE INDEX idx_user_email ON tb_user(email);
INSERT INTO tb_user(name, phone, email) VALUES ('测试1', '13900000001', 'a@x.com');
-- Query OK
INSERT INTO tb_user(name, phone, email) VALUES ('测试2', '13900000002', 'a@x.com');
-- ERROR 1062: Duplicate entry

-- 3. 联合索引 + 观察形态
CREATE INDEX idx_user_pro_age_sta ON tb_user(profession, age, status);
SHOW INDEX FROM tb_user;  -- 该索引占 3 行

-- 4. 全文索引 + 中文验证
CREATE FULLTEXT INDEX idx_user_pro ON tb_user(profession);
SELECT * FROM tb_user WHERE MATCH(profession) AGAINST('软件工程');  -- 命中
SELECT * FROM tb_user WHERE MATCH(profession) AGAINST('软件');      -- 空集

-- 5. 删除索引
DROP INDEX idx_user_pro ON tb_user;      -- 中文无用，清理

-- 6. EXPLAIN 预验证
EXPLAIN SELECT * FROM tb_user
WHERE profession = '软件工程' AND age = 31 AND status = '1';
EXPLAIN SELECT * FROM tb_user WHERE age = 31 AND status = '1';
EXPLAIN SELECT * FROM tb_user WHERE profession = '软件工程';
```

---

## 五、EXPLAIN 对照实验

用三条查询验证联合索引：

| 查询条件 | type | key | key_len | rows |
|---|---|---|---|---|
| 三列全带 | ref | idx_user_pro_age_sta | 54 | 1 |
| 跳过第一列 | **ALL** | **NULL** | NULL | 25 |
| 只有第一列 | ref | idx_user_pro_age_sta | 47 | 4 |

**规律：**
- 三列全带 → key_len=54（三列全用）+ type=ref + rows=1
- 跳过第一列 → key=NULL，全表扫描 25 行
- 只有第一列 → 用索引但只用到一列，rows=4

key_len 的数学：profession varchar(50) 用 utf8 = 47 字节，age int = 4 字节，status char(1) = 3 字节。**47+4+3=54**。key_len 是判断联合索引用了几列的关键指标。

---

## 六、常见错误与避坑（实测踩过的）

| 坑 | 正确做法 |
|---|---|
| 同列重复建索引 | 建前先 `SHOW INDEX` 查重；产生 warning 时紧跟执行 `SHOW WARNINGS` 查看 |
| 用 `CREATE INDEX` 建主键 | 主键只能 `CREATE TABLE` 或 `ALTER TABLE ADD PRIMARY KEY` |
| 想删联合索引的一列 | 只能整个 `DROP INDEX` 后重建，没有"删一列"语法 |
| 用普通 WHERE 查全文索引 | 必须用 `MATCH(col) AGAINST('词')` |
| `SHOW WARNINGS` 隔条执行 | 必须紧跟产生警告的语句，任何新语句都会清空它 |
| `\G` 后再加 `;` | 报 "No query specified"，无害；`\G` 和 `;` 二选一 |
| `DROP INDEX 表名.索引名` | 正确写法：`DROP INDEX 索引名 ON 表名` |

---

## 七、知识点自查清单

- [ ] 四种索引类型的创建语法都会写
- [ ] CREATE 和 ALTER 两种建法都练过
- [ ] 知道主键索引只能通过建表/ALTER 创建
- [ ] 能读懂 SHOW INDEX 的 Non_unique / Seq_in_index / Cardinality
- [ ] 亲眼见过唯一索引的 ERROR 1062
- [ ] 知道联合索引在 SHOW INDEX 中占多行
- [ ] 知道 FULLTEXT 必须 MATCH AGAINST
- [ ] 了解全文索引对中文的局限
- [ ] 会用 EXPLAIN 看索引是否被使用

---

*整理自黑马程序员 MySQL 进阶篇第 17 集，所有语句与输出均在 MariaDB 实际运行验证。*
