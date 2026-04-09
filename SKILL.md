# openclaw-skill-db-query-optim

## 简介

SQL 查询分析与索引优化建议 Skill。读取本 SKILL.md 并严格遵循。

## 触发场景

- "SQL 慢查询"、"query slow"
- "索引优化"、"add index"
- "数据库性能"、"db performance"
- "EXPLAIN"、"执行计划"
- "N+1"、"查询优化"
- "分页"、"limit"

## 核心流程

### 第一步：分析 EXPLAIN 输出

对每条慢查询，必须先获取 EXPLAIN 输出：

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) <your_query>;
```

重点关注：
- **type**：const > eq_ref > ref > range > index > ALL（ALL 是问题）
- **key**：实际用的索引
- **rows**：扫描行数（越大越慢）
- **Extra**：Using filesort / Using temporary 是问题信号

### 第二步：识别典型问题

**全表扫描（type=ALL）：**
```sql
-- 问题
SELECT * FROM orders WHERE DATE(created_at) = '2024-01-01';

-- 解决：范围查询可以走索引
SELECT * FROM orders WHERE created_at >= '2024-01-01' AND created_at < '2024-01-02';
```

**隐式类型转换：**
```sql
-- 问题：user_id 是 int，传入 string
SELECT * FROM users WHERE user_id = '12345';

-- 解决：类型一致
SELECT * FROM users WHERE user_id = 12345;
```

**LIKE 前缀通配：**
```sql
-- 问题：索引失效
SELECT * FROM products WHERE name LIKE '%phone%';

-- 解决：全文索引 / ES，或改为前缀查询
SELECT * FROM products WHERE name LIKE 'phone%';
```

**SELECT *：**
```sql
-- 问题：无法覆盖索引，回表次数多
-- 解决：只查需要的字段
SELECT id, name FROM users WHERE id = 1;
```

**分页深度：**
```sql
-- 问题：OFFSET 大时扫描行数多
SELECT * FROM logs ORDER BY id LIMIT 1000000, 20;

-- 解决：游标分页
SELECT * FROM logs WHERE id > 1000000 ORDER BY id LIMIT 20;
```

### 第三步：索引设计原则

**创建原则：**
1. 等值查询（WHERE a = ?）→ 在 a 上建索引
2. 范围查询（WHERE a > ?）→ 在 a 上建索引（复合索引需放最左）
3. 排序（ORDER BY a）→ 在 a 上建索引（避免 filesort）
4. 覆盖索引：查询字段都在索引里 → 无需回表

**复合索引顺序：**
- 区分度高的字段放前面
- 等值条件在最左，范围条件在最后
- 例如：`WHERE status = 'active' AND created_at > '2024-01-01'`
  → 索引：`idx_status_created(status, created_at)`

**删除无用索引：**
- 用 `SHOW INDEX FROM table` 查看
- 用工具识别：`pt-duplicate-key-checker`（Percona Toolkit）
- 高频 DML 表减少索引数量

### 第四步：输出格式

```
## 查询分析

### 问题查询
```sql
<原始 SQL>
```

### EXPLAIN 关键信息
- type: ALL（问题）
- rows: 100000（扫描行数过多）
- Extra: Using filesort（需优化）

### 问题根因
1. 全表扫描
2. 隐式类型转换
3. ...

### 优化方案
```sql
<优化后 SQL>
```

### 新增索引
```sql
CREATE INDEX idx_xxx ON table(column);
```

### 预期效果
- 扫描行数：100000 → 10
- 执行时间：2s → 10ms
```

## 注意事项

- PostgreSQL 和 MySQL 语法有差异，先确认数据库类型
- 写操作（INSERT/UPDATE）不要建过多索引，影响写入性能
- 批量导入前删除索引，导入完重建