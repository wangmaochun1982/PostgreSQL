https://www.database.ink/postgresql/index-type-selection.html


# B-tree 索引

B-tree 是 PostgreSQL 默认的索引类型，也是最常用的索引类型。它使用平衡树结构，支持等值查询、范围查询、前缀匹配，以及 MIN、MAX 聚合操作。

B-tree 索引维护一个有序的数据结构，每个节点包含多个键值对和指向子节点的指针。B-tree 索引适用于大多数数据类型，包括数值、字符串、日期时间等。

```sql

-- 创建 B-tree 索引
CREATE INDEX idx_users_id ON users (id);

-- 创建复合 B-tree 索引
CREATE INDEX idx_orders_user_date ON orders (user_id, order_date);

-- 创建前缀索引
CREATE INDEX idx_users_email_prefix ON users (email varchar_pattern_ops);

```

# Hash 索引

Hash 索引将键值通过哈希函数转换为固定长度的哈希值，然后存储在哈希表中。

当查询时，同样的哈希函数被用于计算查询键值的哈希值，然后在哈希表中查找。

```sql
-- 创建 Hash 索引
CREATE INDEX idx_users_email_hash ON users USING hash (email);

-- 只能用于等值查询
SELECT * FROM users WHERE email = 'test@example.com';

-- 不支持范围查询，不会使用索引
SELECT * FROM users WHERE email > 'a' AND email < 'z';
```

# GiST 索引

GiST 索引常用于几何数据类型（point、line、polygon、circle 等）的空间查询，全文搜索（tsvector 类型）的文本搜索，以及范围类型的范围查询。

GiST 索引支持多种操作符，如重叠（&&）、相交（<@）、包含（@>）、被包含（<@）等。GiST 索引的设计使其能够高效处理多维数据和复杂的数据类型，适合于需要进行高级空间查询或文本搜索的场景。

```sql

-- 创建 GiST 索引用于空间查询
CREATE INDEX idx_locations_gist ON locations USING gist (location);

-- 使用 GiST 索引进行空间查询
SELECT * FROM locations WHERE location @> 'POINT(10 20)'::point;

-- 创建 GiST 索引用于全文搜索
CREATE INDEX idx_articles_gist ON articles USING gist (to_tsvector('english', content));

-- 使用 GiST 索引进行全文搜索
SELECT * FROM articles WHERE to_tsvector('english', content) @@ to_tsquery('english', 'database');
```
