+++  
date = '2024-12-10T15:52:09+08:00'
title = '数据库原理、常见问题和处理方式(以mysql为例)'  
categories  = ['ServerTech']   
tags = ['mysql']  
+++

# 数据库原理、常见问题和处理方式(以mysql为例)  

## 数据库的三大范式  
为了更好地理解三大范式，我们使用一个用户订单管理系统的例子。

### 第一范式（1NF）  
**定义**：表中的每一列都是不可再分的原子值，即每一个字段都是单一值。
**示例**：

```markdown
| 订单ID | 用户名      | 购买商品             |
|--------|-------------|----------------------|
| 1      | 张三        | 手机, 充电器         |
| 2      | 李四        | 电脑, 鼠标, 键盘     |
```

**问题**：购买商品列包含多个值，不符合1NF。  
**修正**：  

```markdown
| 订单ID | 用户名      | 购买商品  |
|--------|-------------|-----------|
| 1      | 张三        | 手机      |
| 1      | 张三        | 充电器    |
| 2      | 李四        | 电脑      |
| 2      | 李四        | 鼠标      |
| 2      | 李四        | 键盘      |
```

### 第二范式（2NF）  
**定义**：在符合第一范式的基础上，表中的每一个非主键列都完全依赖于主键，不能依赖于主键的一部分。  
**示例**：  

```markdown
| 订单ID | 商品ID | 用户名      | 商品名     |
|--------|--------|-------------|------------|
| 1      | 101    | 张三        | 手机       |
| 1      | 102    | 张三        | 充电器     |
| 2      | 103    | 李四        | 电脑       |
| 2      | 104    | 李四        | 鼠标       |
| 2      | 105    | 李四        | 键盘       |
```

**问题**：用户名依赖于订单ID，而商品名依赖于商品ID，存在部分依赖，不符合2NF。  
**修正**：  

将表拆分成多张表：

```markdown
用户表：
| 用户ID | 用户名      |
|--------|-------------|
| 1      | 张三        |
| 2      | 李四        |

商品表：  
| 商品ID | 商品名  |
|--------|---------|
| 101    | 手机    |
| 102    | 充电器  |
| 103    | 电脑    |
| 104    | 鼠标    |
| 105    | 键盘    |

订单表：  
| 订单ID | 用户ID |
|--------|--------|
| 1      | 1      |
| 2      | 2      |

订单商品表：
| 订单ID | 商品ID |
|--------|--------|
| 1      | 101    |
| 1      | 102    |
| 2      | 103    |
| 2      | 104    |
| 2      | 105    |
```

### 第三范式（3NF）  
**定义**：在符合第二范式的基础上，表中的每一个非主键列都直接依赖于主键，不能依赖于其他非主键列。  
**示例**：  
假设用户表包含用户地址：  

```markdown
| 用户ID | 用户名      | 地址           |
|--------|-------------|----------------|
| 1      | 张三        | 北京市         |
| 2      | 李四        | 上海市         |
```

如果有另一张表记录地址信息：

```markdown
地址表：
| 地址ID | 地址     |
|--------|----------|
| 1      | 北京市   |
| 2      | 上海市   |

用户表：
| 用户ID | 用户名      | 地址ID |
|--------|-------------|--------|
| 1      | 张三        | 1      |
| 2      | 李四        | 2      |
```

这样可以避免用户表中的地址信息冗余和不一致，符合3NF。 

## 哪些场景 sql 会索引失效
在 MySQL 8 中，查询的索引可能会失效的一些场景包括：

### 1.查询条件不使用索引
如果查询条件中的列不包含索引，MySQL 优化器可能会选择全表扫描而不是使用索引。当我们对表执行查询时，MySQL 优化器会尝试选择最优的执行计划。如果查询条件中的列不包含索引，或者没有正确使用索引，那么 MySQL 优化器可能会选择全表扫描而不是使用索引。

#### 导致索引失效的常见原因

**a. 查询条件中使用了函数**:
   当在查询条件中对索引列使用了函数或表达式时，索引可能会失效。例如：

   ```sql
   SELECT * FROM users WHERE LEFT(username, 3) = 'abc';
   ```

   在这个例子中，`username` 列上即使有索引，索引也会因为使用了 `LEFT()` 函数而失效。

**b. 查询条件中使用了不等式操作符**:
   使用不等式操作符（如 `<`, `>`, `!=`, `<>` 等）也可能导致索引失效。例如：

   ```sql
   SELECT * FROM products WHERE price != 100;
   ```

   这种情况下，`price` 列上的索引可能不会被使用。

**c. 查询条件中使用了`OR`**:
   当查询条件中使用了 `OR` 操作符时，如果 `OR` 的每个条件都没有使用索引，那么索引可能会失效。例如：

   ```sql
   SELECT * FROM orders WHERE customer_id = 1 OR order_date = '2023-12-10';
   ```

   如果 `customer_id` 和 `order_date` 列上没有合适的复合索引，查询可能不会使用索引。

**d. 查询条件中的类型不匹配**:
   当查询条件中的列和查询值的类型不匹配时，索引可能会失效。例如，字符串列与数字比较：

   ```sql
   SELECT * FROM users WHERE phone_number = 1234567890;
   ```

   如果 `phone_number` 是字符串类型，查询可能不会使用索引。

**e. 查询条件中的通配符位置**:
   使用 `LIKE` 进行模糊查询时，如果通配符 `%` 在开头，索引可能会失效。例如：

   ```sql
   SELECT * FROM employees WHERE name LIKE '%John';
   ```

   这种情况下，索引不会被使用，因为无法确定匹配的开始位置。
   
#### 如何避免索引失效  
- 避免在索引列上使用函数或表达式，改用索引列直接比较。  
- 尽量使用等值比较，而不是不等式操作。  
- 合理使用复合索引，特别是在使用 OR 操作符时。  
- 确保查询条件中的类型匹配，避免隐式转换。  
- 优化LIKE查询，尽量避免在索引列的开头使用 %

### 2.索引覆盖查询
如果查询仅需要索引中的所有列，而不需要表中的其他列，MySQL 优化器可能会选择使用覆盖索引，而不是其他索引。索引覆盖查询是指查询所需的所有列都包含在某个索引中，查询可以仅通过访问索引来获取数据，而不需要访问实际的表数据。这种查询通过避免访问表数据提高了查询性能。
示例：  
假设我们有一个用户表 users：  

```sql
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    username VARCHAR(50),
    email VARCHAR(100),
    age INT,
    created_at DATETIME,
    INDEX (username, email)
);
```
假设我们执行以下查询：
```sql  
SELECT username, email FROM users WHERE username = 'john_doe';
``` 
索引覆盖查询的条件：  
- 查询中所需的所有列（username 和 email）都包含在索引中。
- 查询条件（WHERE username = 'john_doe'）使用了索引列。

由于 username 和 email 都在索引中，这个查询可以完全通过索引来完成，而不需要访问表数据。
**注意事项**：
- **选择性**：索引覆盖查询的性能提升在大数据集上尤为明显。
- **索引大小**：过大的索引可能会导致维护成本增加（如插入、更新、删除操作的性能下降）。

可以使用 `EXPLAIN` 语句来确认查询是否使用了索引覆盖：
```sql
EXPLAIN SELECT username, email FROM users WHERE username = 'john_doe';
```
如果 `Extra` 列中显示 `Using index`，则表示这是一个索引覆盖查询。 
 
### 3.数据量小  
如果表中的数据量很小，MySQL 优化器可能会认为全表扫描比使用索引更高效。
### 4.索引失效  
在 MySQL 中，当表数据发生大量的插入、更新或删除操作时，索引的统计信息可能会变得不准确，从而影响查询优化器的决策。这种情况下，优化器可能会选择不使用索引，直到统计信息更新为止。

#### 索引失效的原因

1. **统计信息过期**：
   - 当数据表频繁进行大量的插入、更新或删除操作时，表的统计信息（如行数、键分布等）可能会变得不准确。统计信息是 MySQL 优化器用来决定查询执行计划的重要依据。
   - 如果统计信息不准确，优化器可能会错误地估计使用某个索引的成本，导致选择全表扫描或其他非索引的执行计划。

2. **索引碎片化**：
   - 大量的插入和删除操作可能会导致索引变得碎片化。碎片化的索引会增加磁盘 I/O，降低查询性能。
   - 在这种情况下，优化器可能会决定不使用已碎片化的索引，而是选择全表扫描。

#### 解决方案  
1. **重新分析表**：
   - 使用 `ANALYZE TABLE` 命令可以更新表的统计信息，使优化器能够更准确地评估使用索引的成本。

   ```sql
   ANALYZE TABLE table_name;
   ```

2. **重建索引**：
   - 使用 `OPTIMIZE TABLE` 命令可以重建表和索引，减少索引的碎片化。

   ```sql
   OPTIMIZE TABLE table_name;
   ```

3. **定期维护**：
   - 定期分析和优化表，保持索引的高效性和准确的统计信息。
   - 配置 MySQL 自动更新统计信息的参数，如 `innodb_stats_auto_recalc` 和 `innodb_stats_persistent`，以便 MySQL 自动更新统计信息。

### 5.查询优化器的选择  
MySQL 优化器会根据查询的具体情况选择最佳执行计划，有时可能会选择不使用索引。
### 6.索引失效的优化器提示
如果使用了 `IGNORE INDEX` 或 `FORCE INDEX` 优化器提示，可能会导致索引失效。


## 如何查看知道自己的查询是否走了索引  
### 如何查看查询是否使用了索引

为了确保你的查询在使用索引，可以使用 MySQL 提供的 `EXPLAIN` 语句。

#### `EXPLAIN`返回的内容

`EXPLAIN` 语句用于查看查询的执行计划，显示查询如何访问数据表以及使用了哪些索引。执行 `EXPLAIN` 语句后，你会看到一张包含以下字段的结果表：

1. **id**: 查询的标识符。
2. **select_type**: 查询的类型，例如 `SIMPLE` 表示简单查询。
3. **table**: 被访问的表的名称。
4. **partitions**: 匹配的分区。
5. **type**: 联接类型，表示表访问的方法，例如 `index`, `range`, `ref`, `eq_ref`, `const`, `system`, `NULL`。
6. **possible_keys**: 查询中可能使用的索引。
7. **key**: 查询实际使用的索引。
8. **key_len**: 使用索引的长度。
9. **ref**: 显示使用哪个列或常量与 key 一起从表中选择行。
10. **rows**: MySQL 估计需要读取的行数。
11. **filtered**: 经过查询条件过滤后，估计的行百分比。
12. **Extra**: 额外的信息，比如 `Using index` 表示覆盖索引，`Using where` 表示使用了 WHERE 子句。

#### 示例

假设我们有以下几个表用于管理用户订单信息：

```sql
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    username VARCHAR(50),
    email VARCHAR(100),
    INDEX (username)
);

CREATE TABLE products (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(100),
    price DECIMAL(10, 2),
    INDEX (product_name)
);

CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    user_id INT,
    order_date DATETIME,
    INDEX (user_id)
);

CREATE TABLE order_items (
    order_item_id INT PRIMARY KEY,
    order_id INT,
    product_id INT,
    quantity INT,
    INDEX (order_id, product_id)
);
```

假设我们要执行以下查询来获取某个用户的订单详情：

```sql
SELECT u.username, o.order_id, p.product_name, oi.quantity
FROM users u
JOIN orders o ON u.user_id = o.user_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE u.username = 'john_doe';
```

使用 `EXPLAIN` 语句查看查询的执行计划：

```sql
EXPLAIN 
SELECT u.username, o.order_id, p.product_name, oi.quantity
FROM users u
JOIN orders o ON u.user_id = o.user_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE u.username = 'john_doe';
```

执行后你可能会得到如下结果：

```markdown
| id | select_type | table       | partitions | type  | possible_keys | key              | key_len | ref          | rows | filtered | Extra       |
|----|-------------|-------------|------------|-------|---------------|------------------|---------|--------------|------|----------|-------------|
| 1  | SIMPLE      | u           | NULL       | ref   | username      | username         | 152     | const        | 1    | 100.00   | Using index |
| 1  | SIMPLE      | o           | NULL       | ref   | PRIMARY       | PRIMARY          | 4       | u.user_id    | 10   | 100.00   | NULL        |
| 1  | SIMPLE      | oi          | NULL       | ref   | order_id      | order_id         | 4       | o.order_id   | 10   | 100.00   | NULL        |
| 1  | SIMPLE      | p           | NULL       | eq_ref| PRIMARY       | PRIMARY          | 4       | oi.product_id| 1    | 100.00   | Using index |
```

**解释**：
- **users 表 (u)**: 
  - **key** 列显示使用了 `username` 索引。
  - **Extra** 列显示 `Using index`，表示这是一个覆盖索引查询。
  
- **orders 表 (o)**:
  - **key** 列显示使用了 `PRIMARY` 索引。
  - **ref** 列显示连接条件 `u.user_id`。

- **order_items 表 (oi)**:
  - **key** 列显示使用了 `order_id` 索引。
  - **ref** 列显示连接条件 `o.order_id`。

- **products 表 (p)**:
  - **key** 列显示使用了 `PRIMARY` 索引。
  - **ref** 列显示连接条件 `oi.product_id`。
  - **Extra** 列显示 `Using index`，表示这是一个覆盖索引查询。

## 怎么知道 sql 走了回表  
"回表" 指的是在查询过程中，MySQL 通过索引查找到数据的位置后，还需要再访问一次实际的数据表以获取所需的列数据。回表通常发生在所需的查询列没有被索引覆盖时。使用 EXPLAIN 语句可以帮助你判断查询是否走了回表。
还以订单查询为例：
```sql
EXPLAIN 
SELECT u.username, o.order_id, p.product_name, oi.quantity
FROM users u
JOIN orders o ON u.user_id = o.user_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE u.username = 'john_doe';
```  
返回结果：
```  
| id | select_type | table       | partitions | type  | possible_keys | key              | key_len | ref          | rows | filtered | Extra                     |
|----|-------------|-------------|------------|-------|---------------|------------------|---------|--------------|------|----------|---------------------------|
| 1  | SIMPLE      | u           | NULL       | ref   | username      | username         | 152     | const        | 1    | 100.00   | Using where; Using index  |
| 1  | SIMPLE      | o           | NULL       | ref   | PRIMARY       | PRIMARY          | 4       | u.user_id    | 10   | 100.00   | NULL                      |
| 1  | SIMPLE      | oi          | NULL       | ref   | order_id      | order_id         | 4       | o.order_id   | 10   | 100.00   | NULL                      |
| 1  | SIMPLE      | p           | NULL       | eq_ref| PRIMARY       | PRIMARY          | 4       | oi.product_id| 1    | 100.00   | Using index               |

```  
users 表 (u):
- Extra 列显示 Using where; Using index，表示查询在使用索引的同时也需要访问表数据。
- orders 表 (o)、order_items 表 (oi) 和 products 表 (p):
- Extra 列显示 NULL，表示这些表没有进行回表操作。

注意事项：
Using index 表示查询是索引覆盖查询，没有回表。
Using where; Using index 表示查询需要根据条件过滤数据，且使用了索引，但仍然进行了回表。
NULL 表示查询没有使用索引或没有进行回表操作。  

## 如何知道发生了死锁  
在 MySQL 中，死锁是指两个或多个事务相互等待对方持有的资源，导致它们都无法继续执行。如果发生了死锁，MySQL 会自动检测并解决它，通常是回滚其中一个事务。

### 方法一：查看错误日志
当 MySQL 发生死锁时，会在错误日志中记录详细信息。你可以查看 MySQL 的错误日志来确认死锁。

```sh
# 查看 MySQL 错误日志
tail -f /var/log/mysql/error.log
```

### 方法二：使用 SHOW ENGINE INNODB STATUS
`SHOW ENGINE INNODB STATUS` 命令可以显示 InnoDB 存储引擎的详细状态信息，包括死锁信息。

```sql
SHOW ENGINE INNODB STATUS;
```

执行该命令后，你会看到类似以下的输出：

```markdown
------------------------
LATEST DETECTED DEADLOCK
------------------------
2024-12-10 17:20:00 0x7f2e9c0e1700
*** (1) TRANSACTION:
TRANSACTION 18446744073709551614, ACTIVE 0 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 5 lock struct(s), heap size 1136, 3 row lock(s)
MySQL thread id 5, OS thread handle 140205845360384, query id 12 localhost root updating
UPDATE orders SET status='shipped' WHERE order_id=1
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 3 page no 4 n bits 72 index `PRIMARY` of table `test`.`orders` trx id 18446744073709551614 lock_mode X locks rec but not gap waiting
Record lock, heap no 2 PHYSICAL RECORD: n_fields 5; compact format; info bits 32
 0: len 4; hex 80000001; asc     ;;
 1: len 6; hex 000002745b01; asc    t
```

## 发生了死锁如何解决  
### 如何解决死锁问题

当 MySQL 中发生死锁时，通常会自动检测并解决它，通常是回滚其中一个事务。然而，我们也可以采取一些预防和解决死锁的方法。

#### 方法

1. **手动终止事务**：
   - 使用 `SHOW ENGINE INNODB STATUS` 查看哪个事务导致了死锁，然后手动终止该事务。
   - 查看死锁信息：

     ```sql
     SHOW ENGINE INNODB STATUS;
     ```

   - 使用 `KILL` 命令终止事务：

     ```sql
     KILL QUERY <线程ID>;
     ```

2. **调整事务的执行顺序**：
   - 确保所有事务以相同的顺序访问资源。例如，如果多个事务都需要访问表A和表B，确保所有事务都先访问表A再访问表B。

3. **减少锁的持有时间**：
   - 尽量缩短事务的执行时间，避免在事务中执行耗时的操作。

4. **合理使用锁的粒度**：
   - 尽量使用行锁而不是表锁，以减少锁冲突。
   - 如果表中的行数较少，可以考虑使用表锁来提高并发性能。

5. **分解大事务**：
   - 将大事务拆分成多个小事务，这样可以减少锁的持有时间和冲突的可能性。

6. **使用乐观锁**：
   - 在某些场景下，可以使用乐观锁来代替悲观锁，减少锁冲突的机会。乐观锁通常使用版本号来控制并发。

#### 示例

假设我们在处理用户订单时，发生了死锁。我们可以通过以下步骤来解决：

1. 使用 `SHOW ENGINE INNODB STATUS` 查看死锁信息：

   ```sql
   SHOW ENGINE INNODB STATUS;
   ```

2. 查找导致死锁的事务并记录其线程ID。

3. 终止死锁事务：
   
   ```sql
   KILL QUERY <线程ID>;
   ```
4. 调整事务的执行顺序和粒度，确保类似问题不再发生。

## 乐观锁实现的一些常见机制
乐观锁是一种控制并发操作的方法，通常用于防止在不加锁的情况下数据被并发修改。乐观锁通过在更新数据时检测是否发生了冲突来确保数据一致性。
1. **版本号机制**：
   - **基本思想**：为每行数据添加一个版本号字段，每次更新数据时，版本号加1。在进行更新操作时，判断当前数据的版本号是否与提交数据的版本号一致。如果一致，表示数据未被其他事务修改，可以进行更新；否则，表示数据已被修改，需要重新获取数据进行处理。
   - **示例**：

     ```sql
     -- 添加版本号字段
     ALTER TABLE orders ADD COLUMN version INT DEFAULT 0;

     -- 更新操作
     UPDATE orders
     SET status='shipped', version=version+1
     WHERE order_id=1 AND version=<当前版本号>;
     ```

     如果返回受影响的行数为0，表示数据已被修改，需要重新获取数据并重试操作。

2. **时间戳机制**：
   - **基本思想**：与版本号机制类似，但使用时间戳来标记数据的最后更新时间。在进行更新操作时，比较当前数据的时间戳与提交数据的时间戳，如果一致，则进行更新；否则，表示数据已被修改，需要重新获取数据进行处理。
   - **示例**：

     ```sql
     -- 添加时间戳字段
     ALTER TABLE orders ADD COLUMN last_update TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP;

     -- 更新操作
     UPDATE orders
     SET status='shipped'
     WHERE order_id=1 AND last_update=<当前时间戳>;
     ```

     同样，如果返回受影响的行数为0，表示数据已被修改，需要重新获取数据并重试操作。

3. **标志位机制**：
   - **基本思想**：在数据表中添加一个标志位字段，用于标记数据是否正在被操作。在进行更新操作前，首先检查标志位，如果标志位为允许操作的状态，则将其修改为操作状态，进行更新后再恢复标志位；否则，表示数据正在被修改，需要等待或重新获取数据。
   - **示例**：

     ```sql
     -- 添加标志位字段
     ALTER TABLE orders ADD COLUMN is_updating BOOL DEFAULT FALSE;

     -- 更新操作
     BEGIN;

     -- 检查并设置标志位
     SELECT is_updating FROM orders WHERE order_id=1 FOR UPDATE;
     UPDATE orders
     SET is_updating=TRUE
     WHERE order_id=1 AND is_updating=FALSE;

     -- 执行实际更新
     UPDATE orders
     SET status='shipped'
     WHERE order_id=1;

     -- 恢复标志位
     UPDATE orders
     SET is_updating=FALSE
     WHERE order_id=1;

     COMMIT;
     ```
