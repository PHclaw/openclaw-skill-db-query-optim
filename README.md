# openclaw-skill-db-query-optim

SQL 查询分析与索引优化建议。读取 EXPLAIN 输出，识别慢查询根因，给出优化方案。

## 功能

- EXPLAIN 执行计划分析（type/key/rows/Extra）
- 常见慢查询模式修复（全表扫描、隐式转换、深度分页）
- 索引设计与复合索引顺序原则
- PostgreSQL / MySQL / SQLite 语法适配

## 安装

将 `SKILL.md` 复制到 OpenClaw skills 目录即可。

## 使用场景

- SQL 慢查询分析
- 索引优化建议
- N+1 查询问题
- 数据库性能调优
- 分页查询优化
