# Hive SQL 血缘分析器（lineage_cli）

Rust 实现的 Hive SQL 血缘分析命令行工具：
- 列级血缘：产出 `db.table.column <- sources` 形式的映射
- 表级血缘：产出 `db.target <- db.source1, db.source2`
- 支持 CTAS、INSERT、CREATE VIEW、CTE、窗口函数、过滤与别名等常见语法

## 安装

### 从 GitHub Releases 下载

访问 [Releases](https://github.com/YOUR_USERNAME/lineage_cli/releases) 页面下载适合您平台的预编译二进制文件。

支持的平台：
- Linux (x86_64, i686, ARM64, ARMv7, s390x)
- macOS (Intel, Apple Silicon)
- Windows (x64, x86, ARM64)

### 从源码构建

```bash
# 克隆仓库
git clone https://github.com/YOUR_USERNAME/lineage_cli.git
cd lineage_cli

# 构建 release 版本
cargo build --release

# 可执行文件位于
./target/release/lineage_cli
```

## 使用方法

### 基本用法

```bash
# 分析 SQL 文件
lineage_cli path/to/query.sql

# 从标准输入读取
cat query.sql | lineage_cli -
```

### 命令行参数

- `-t, --tables` - 输出表级血缘（默认为列级血缘）
- `-j, --json` - 以 JSON 格式输出
- `-p, --pretty` - JSON 美化输出
- `-` - 从标准输入读取 SQL

### 示例

输入 SQL（`test.sql`）：
```sql
USE db1;
CREATE TABLE t AS SELECT id, upper(name) AS uname FROM s;
```

**列级输出（默认）：**
```bash
$ lineage_cli test.sql
db1.t.id <- db1.s.id
db1.t.uname <- db1.s.name
```

**表级输出：**
```bash
$ lineage_cli --tables test.sql
db1.t <- db1.s
```

**JSON 输出：**
```bash
$ lineage_cli --json --pretty test.sql
[
  {
    "stmt_index": 0,
    "stmt_type": "CTAS",
    "start_line": 2,
    "start_col": 1,
    "end_line": 2,
    "end_col": 54,
    "target": "db1.t",
    "column": "id",
    "sources": ["db1.s.id"],
    "expr": null,
    "statement": "CREATE TABLE t AS SELECT id, upper(name) AS uname FROM s"
  },
  {
    "stmt_index": 0,
    "stmt_type": "CTAS",
    "start_line": 2,
    "start_col": 1,
    "end_line": 2,
    "end_col": 54,
    "target": "db1.t",
    "column": "uname",
    "sources": ["db1.s.name"],
    "expr": null,
    "statement": "CREATE TABLE t AS SELECT id, upper(name) AS uname FROM s"
  }
]
```

## 作为库使用

在 `Cargo.toml` 中添加依赖：
```toml
[dependencies]
lineage_cli = "0.1"
```

示例代码：
```rust
use lineage_cli::analyze_sql_lineage;

fn main() -> anyhow::Result<()> {
    let sql = "USE db1; CREATE TABLE t AS SELECT id FROM s;";
    let lines = analyze_sql_lineage(sql)?;
    for l in lines { 
        println!("{}", l); 
    }
    Ok(())
}
```

**可用的公共 API：**
- `analyze_sql_lineage(sql: &str)` - 返回列级血缘字符串列表
- `analyze_sql_tables(sql: &str)` - 返回表级血缘字符串列表
- `analyze_sql_lineage_detailed(sql: &str)` - 返回详细的列级血缘结构体
- `analyze_sql_tables_detailed(sql: &str)` - 返回详细的表级血缘结构体

## 开发与测试

```bash
# 运行测试
cargo test

# 代码格式化
cargo fmt

# 静态检查
cargo clippy

# 构建 release 版本
cargo build --release
```

## 跨平台支持

本项目使用 GitHub Actions 自动构建多平台二进制文件。当推送 `v*` 标签时，会自动构建并发布以下平台的包：

- Linux: x86_64 (musl), i686, ARM64, ARMv7, s390x
- macOS: Intel (x86_64), Apple Silicon (ARM64)
- Windows: x64 (MSVC/GNU), x86, ARM64

## 默认数据库（schema）

- 若 SQL 中未通过 `USE db;` 指定当前数据库，则未带库名的表引用默认使用 `default` 库。
- 例如：
  - 输入：`INSERT OVERWRITE TABLE sdl.tae SELECT id, n AS name FROM sals;`
  - 解析：源表 `sals` 被解析为 `default.sals`

## 表达式回退策略

- **目标**：当投影表达式中没有任何明确的列依赖时，尽可能给出合理、可追踪的归属。

- **单表归属**：
  - 若当前作用域内仅有一个真实来源表（或最终可解析为唯一来源表），则将该表达式归属到该表，使用"原始表达式文本"作为伪列名。
  - 示例：`SELECT 1 AS n1 FROM s` 在 `INSERT/CTAS/VIEW` 中会产出 `db.s.1`，因此血缘行形如 `db.t.n1 <- db.s.1`。

- **多表或不唯一**：
  - 若作用域内存在多个来源且无法唯一确定归属，则保留原始表达式字符串作为右侧（表示无法定位到具体列）。
  - 对于"匿名投影"（未命名表达式）在不唯一场景下，还会回退为基础表的 `*`（例如 `a.*`, `b.*`），用以提示列级无法精确定位。

- **作用范围**：
  - 该策略同时适用于派生子查询（Derived Table）、CTE 以及顶层 SELECT。
  - 当上层引用派生/CTE 列且该列来源为常量/纯表达式时，若其子查询/CTE 的来源唯一，则会将其转化为对该唯一表的"伪列表达"（例如 `db.s.upper('abc')`）。

- **简单字面量定义**（触发"单表归属"）：
  - 所有"不包含字段引用"的表达式均视为简单字面量。
  - 包括但不限于：数值/字符串/布尔/NULL 字面量、带一元正负号（如 `-1`）、`CAST(1 AS INT)`、函数调用（如 `upper('abc')`）、CASE 表达式、`IN` 列表等，以及它们的括号嵌套形式。

## JSON 详细输出示例（含回退）

### 单表常量回退（sources 带伪列名，expr 为空）

输入：
```sql
USE db1; 
CREATE TABLE t AS SELECT 1 AS c FROM s;
```

输出（`--json --pretty`）：
```json
{
  "stmt_type": "CTAS",
  "target": "db1.t",
  "column": "c",
  "sources": ["db1.s.1"],
  "expr": null,
  ...
}
```

### 多表且表达式不唯一（sources 缺失，expr 保留原始表达式）

输入：
```sql
USE db1; 
CREATE TABLE t2 AS SELECT 1 AS c FROM a JOIN b ON a.id=b.id;
```

输出（`--json --pretty`）：
```json
{
  "stmt_type": "CTAS",
  "target": "db1.t2",
  "column": "c",
  "sources": null,
  "expr": "1",
  ...
}
```

说明：此时无法将常量表达式唯一归属到某一来源表，因此以 `expr` 字段承载。

### 函数/CASE/IN 等不依赖字段的复杂表达式（单表归属）

输入：
```sql
USE d; 
CREATE TABLE s(id INT); 
CREATE TABLE t AS SELECT 
  upper('abc') AS u, 
  CASE WHEN 1=1 THEN 2 ELSE 3 END AS c, 
  2 IN (1,2,3) AS f 
FROM s;
```

输出（节选，每列一条）：
```json
{ "stmt_type": "CTAS", "target": "d.t", "column": "u", "sources": ["d.s.upper('abc')"], "expr": null, ... }
{ "stmt_type": "CTAS", "target": "d.t", "column": "c", "sources": ["d.s.CASE WHEN 1 = 1 THEN 2 ELSE 3 END"], "expr": null, ... }
{ "stmt_type": "CTAS", "target": "d.t", "column": "f", "sources": ["d.s.2 IN (1, 2, 3)"], "expr": null, ... }
```

## 贡献

请查看 `AGENTS.md` 获取项目结构、编码规范、测试与提交流程等贡献指南。

## 许可证

本项目采用 Apache-2.0 license 许可证。
