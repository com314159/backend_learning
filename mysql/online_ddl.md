好的，这是一个非常重要且实用的数据库运维话题。**Online DDL** 是指在数据库表结构变更（如加字段、加索引）时，不阻塞表的读写操作（或只短时间阻塞），最大限度地保证业务可用性。

MySQL（特别是 InnoDB 引擎）从 5.6 版本开始，官方系统地支持了 Online DDL 功能，并在后续版本中不断增强。

下面我将详细为你讲解哪些操作可以进行 Online DDL，其背后的原理，以及注意事项。

---

### 一、Online DDL 的两种算法

在 MySQL 的 `ALTER TABLE` 语句中，可以通过 `ALGORITHM` 和 `LOCK` 子句来控制 DDL 行为：

1.  **`ALGORITHM` (算法)**：
    *   **`COPY`**：**非在线**。MySQL 会创建一个临时空表，将原表数据逐行拷贝到新表，期间会阻塞写操作。完成后将新表重命名为原表名。效率低，影响大。
    *   **`INPLACE`**：**在线**。大多数操作不需要复制整个表数据。引擎会在内部进行更改，通常比重建整个表要快得多。但“INPLACE”并不总是意味着“不锁表”。
    *   **`INSTANT`** (MySQL 8.0+): **瞬间**。最新特性，某些元数据操作只需修改数据字典（元数据）即可完成，无需触碰表数据，也无需重建表，速度极快。

2.  **`LOCK` (锁级别)**：
    *   **`NONE`**: 允许并发读写。这是最理想的情况。
    *   **`SHARED`**: 允许并发读，但阻塞写。
    *   **`EXCLUSIVE`**: 阻塞所有读写。相当于锁表。

**最佳实践**：在执行 DDL 时，显式指定算法和锁级别，让数据库按你的预期执行，避免意外。
```sql
ALTER TABLE table_name ADD COLUMN ... ALGORITHM=INPLACE, LOCK=NONE;
```

---

### 二、支持 Online DDL 的常见操作

下表总结了常见的 DDL 操作及其在 MySQL 8.0 下的支持情况（因版本而异，8.0 支持得最好）。

| 操作类型 | 具体操作 | 算法（通常） | 锁级别（通常） | 说明与注意事项 |
| :--- | :--- | :--- | :--- | :--- |
| **索引操作** | **`ADD INDEX`** / `CREATE INDEX` | `INPLACE` | `NONE` | **最经典的 Online DDL 场景**。全程允许读写。 |
| | **`DROP INDEX`** | `INPLACE` | `NONE` | 非常简单快速，允许读写。 |
| **列操作** | **`ADD COLUMN`** (加到列尾) | `INPLACE` | `NONE` | 8.0 下通常很快。但若指定 `AFTER col_name` 或 `FIRST` 可能导致表重建（`COPY`）。 |
| | `DROP COLUMN` | `INPLACE` | `NONE` | 需要重建表，但是在线操作。开销较大。 |
| | `RENAME COLUMN` | `INPLACE` | `NONE` | 8.0+ 支持，仅修改元数据，极其快速。 |
| | **修改列类型** (e.g., `INT` -> `BIGINT`) | `COPY` | `EXCLUSIVE` | **通常不支持 Online DDL！** 需要锁表并重建，影响巨大。 |
| | `SET DEFAULT` / `DROP DEFAULT` | `INPLACE` | `NONE` | 8.0+ 支持，仅修改元数据，瞬间完成。 |
| **表操作** | `OPTIMIZE TABLE` | `INPLACE` | `NONE` | 本质上是通过 `ALTER TABLE ... FORCE` 在线重建表并整理碎片。 |
| | `RENAME TABLE` | `COPY` | `EXCLUSIVE` | **`RENAME` 操作是原子的，但需要排它锁**。不过它执行极快，短时间锁表。 |
| | `MODIFY COLUMN` ... `NOT NULL` | 可能 `INPLACE` | 可能 `NONE` | 8.0+ 对某些情况支持在线，但复杂，建议测试。 |

**特别注意 `INSTANT` 算法 (MySQL 8.0+)**
MySQL 8.0 对 `ADD COLUMN` 等操作进行了极致优化，在大多数情况下（例如新增列放在最后）使用的是 `INSTANT` 算法，**仅修改元数据，耗时在毫秒级**，对性能几乎无影响。

---

### 三、不支持或需要谨慎操作的 Online DDL

以下操作**通常不支持** Online DDL，会引发表重建和长时间锁表，务必在业务低峰期进行：

1.  **修改列的数据类型**：如 `VARCHAR(10)` 改为 `VARCHAR(20)`（注意：**增加 `VARCHAR` 长度在 255 以下和以上行为不同**，但安全起见都视为危险操作），`INT` 改为 `BIGINT` 等。
2.  **修改列的字符集**：如从 `utf8mb3` 改为 `utf8mb4`。
3.  **删除主键**：`DROP PRIMARY KEY`。
4.  **添加自增列**：在某些旧版本中可能触发表重建。
5.  **变更表选项**：如 `ROW_FORMAT`, `KEY_BLOCK_SIZE`, `CHARACTER SET`。

**如何确认一个 DDL 语句是否在线？**
在执行前，使用 `ALTER TABLE ... ALGORITHM=INPLACE, LOCK=NONE`。如果该操作不支持在线模式，MySQL 会直接报错，而不会执行，从而避免了误操作导致的服务中断。
```sql
-- 示例：尝试在线修改列类型（会失败）
ALTER TABLE users MODIFY COLUMN name VARCHAR(100) ALGORITHM=INPLACE, LOCK=NONE;
-- ERROR: 1846 (0A000): ALGORITHM=INPLACE is not supported. Reason: Cannot change column type INPLACE. Try ALGORITHM=COPY.
```

---

### 四、Online DDL 的工作原理与注意事项

*   **原理**：Online DDL 的核心是在执行 `INPLACE` 算法时，MySQL 会为当前表创建一个**临时副本（在 `InnoDB` 内部）** 来进行结构变更。在此期间：
    1.  对原表的 **DML 操作（INSERT/UPDATE/DELETE）** 会被记录到一个**日志文件（Online DDL Log）** 中。
    2.  当表结构变更完成后，MySQL 会**应用**这些日志记录到新表上，保证数据一致性。
    3.  最后，用新表替换旧表。

*   **注意事项**：
    1.  **空间开销**：Online DDL 操作期间，需要额外的磁盘空间来存储临时副本和日志。确保你的磁盘有足够空间（通常需要约等于原表大小的空间）。
    2.  **性能影响**：虽然不锁表，但**大量数据的拷贝和日志应用会消耗大量 I/O 和 CPU**，可能仍然会影响数据库的整体性能。建议在业务低峰期进行。
    3.  **元数据锁（MDL）**：即使指定 `LOCK=NONE`，在 DDL 的开始和结束阶段仍然需要获取一个**短暂的排他元数据锁**，以更新数据字典。如果此时有未提交的**长事务**或**活跃的查询**持有该表的元数据读锁，DDL 操作就会阻塞等待，直到它们释放，反之亦然。这是 Online DDL 在实践中最常见的阻塞点。
    4.  **主从延迟**：即使在主库上在线执行，DDL 语句在从库上回放时同样需要时间，可能导致主从延迟增大。

### 总结与最佳实践

1.  **首选 MySQL 8.0+**：其对 Online DDL 的支持最完善，尤其是 `INSTANT` 算法。
2.  **提前测试**：任何 DDL 操作都应先在测试环境验证其影响（耗时、资源消耗、锁情况）。
3.  **显式指定算法**：在 `ALTER TABLE` 语句中加上 `ALGORITHM=INPLACE, LOCK=NONE`，让数据库按你的预期执行，如果不支持则会报错，避免“惊喜”。
4.  **业务低峰期操作**：即使是在线操作，也选择流量最低的时候进行。
5.  **监控长事务**：在执行 DDL 前，检查并避免存在长时间运行的事务，以防被元数据锁阻塞。
6.  **使用第三方工具**：对于超大规模的表，可以考虑使用 **`pt-online-schema-change`** (Percona Toolkit) 或 **`gh-ost`** (GitHub) 等外部工具，它们通过触发器或 binlog 的方式来实现更安全、影响更小的表结构变更。
