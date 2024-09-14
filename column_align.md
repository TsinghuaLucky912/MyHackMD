
# 背景说明

上周有幸参加了PostgreSQL技术峰会西安站的大会，见到了很多老朋友(灿灿 腾讯云向博他们 亚信梁博老师还有老东家某炫的同事) 中间也学到了很多好东西。最后灿总压轴分享的优化相关的内容，可谓是收获满满。可能是饿了，也可能是学着忘着 印象很深刻的一个知识点是关于对齐的，如下是灿总文档中的一页：

![72b06227aeff04ff0518abff0f4ad8f](https://hackmd.io/_uploads/HyiauaGpR.png)

同样的表，因列顺序的不同而造成不必要的填充 对存储和性能上有一定的影响。这点我之前从来没有关注过，或者说要去关注。

---

下面是第二象限(EDB)的一篇文档(原文名称: **On Rocks and Sand** 这个名字起的也非常形象)，在列对齐这方面影响很大 这里我谨将翻译放在这里，不做任何更改，有兴趣的小伙伴可以查看原文。

<u>[原文链接：https://www.2ndquadrant.com/en/blog/on-rocks-and-sand/](https://www.2ndquadrant.com/en/blog/on-rocks-and-sand/)</u>

:::info
在进行数据库容量规划时，需要考虑很多变量，Postgres 在这方面也不例外。需要管理的元素之一是存储。然而，存储的一个方面几乎无一例外地逃避了检查，它隐藏在列之间的阴影中。
:::

## 对齐基础

在大多数低级计算机语言（如 C 语言）中，无论数据类型的实际大小如何，数据类型都是通过其最大大小来寻址的。因此，一个标准的 32 位整数（可以存储超过 20 亿的值）必须作为一个整体来读取。这意味着即使 0 的值也需要 4 个字节的存储空间。

此外，Postgres 的设计使其自身的内部自然对齐为 8 个字节，这意味着在某些情况下，不同大小的连续固定长度列必须用空字节填充。我们可以通过这个例子看到这一点：

```sql=
SELECT pg_column_size(row()) AS empty,
       pg_column_size(row(0::SMALLINT)) AS byte2,
       pg_column_size(row(0::BIGINT)) AS byte8,
       pg_column_size(row(0::SMALLINT, 0::BIGINT)) AS byte16;

 empty | byte2 | byte8 | byte16 
-------+-------+-------+--------
    24 |    26 |    32 |     40
```

这表明，一个空的 Postgres 行需要 24 个字节的各种标题元素，SMALLINT 是 2 个字节，BIGINT 是 8 个字节，将它们组合起来是……16 个字节？这没错；Postgres 正在填充较小的列以匹配下一行的大小，以便进行对齐。我们的计算结果不是 2 + 8 = 10，而是 8 + 8 = 16。

## 强度间隔

就其本身而言，这可能不一定是个问题。但请考虑这样一个设计出来的订购系统(这里称为 **表设计1**)：

```sql=
CREATE TABLE user_order (
  is_shipped    BOOLEAN NOT NULL DEFAULT false,
  user_id       BIGINT NOT NULL,
  order_total   NUMERIC NOT NULL,
  order_dt      TIMESTAMPTZ NOT NULL,
  order_type    SMALLINT NOT NULL,
  ship_dt       TIMESTAMPTZ,
  item_ct       INT NOT NULL,
  ship_cost     NUMERIC,
  receive_dt    TIMESTAMPTZ,
  tracking_cd   TEXT,
  id            BIGSERIAL PRIMARY KEY NOT NULL
);
```

如果看起来有点奇怪，那是因为它确实很奇怪。匆忙的开发人员简单地记下属性或从任意散列键位置生成输出的 ORM 决定列顺序的情况并不少见。看到末尾的 id 列是一个很好的指标，表明列顺序不是架构或规划路线图的一部分。

鉴于此，这就是 Postgres 所看到的：

```sql=
SELECT a.attname, t.typname, t.typalign, t.typlen
  FROM pg_class c
  JOIN pg_attribute a ON (a.attrelid = c.oid)
  JOIN pg_type t ON (t.oid = a.atttypid)
 WHERE c.relname = 'user_order'
   AND a.attnum >= 0
 ORDER BY a.attnum;

   attname   |   typname   | typalign | typlen 
-------------+-------------+----------+--------
 is_shipped  | bool        | c        |      1
 user_id     | int8        | d        |      8
 order_total | numeric     | i        |     -1
 order_dt    | timestamptz | d        |      8
 order_type  | int2        | s        |      2
 ship_dt     | timestamptz | d        |      8
 item_ct     | int4        | i        |      4
 ship_cost   | numeric     | i        |     -1
 receive_dt  | timestamptz | d        |      8
 tracking_cd | text        | i        |     -1
 id          | int8        | d        |      8
```

typalign 列根据 pg_type 手册页中列出的值指示预期的对齐类型。

> typalign char
> 
> typalign 是存储此类型的值时所需的对齐方式。它适用于磁盘上的存储以及 PostgreSQL 内部值的大多数表示。当多个值连续存储时（例如在磁盘上表示整行时），会在这种类型的数据前插入填充，以便它从指定的边界开始。对齐参考是序列中第一个数据的开头。可能的值包括：
> * c = char 对齐，即不需要对齐
> * s = 短对齐（大多数机器上为 2 个字节）
> * i = int 对齐（大多数机器上为 4 个字节）
> * d = 双对齐（许多机器上为 8 个字节，但绝不是全部）

从前面的讨论中，我们可以看到 INT 类型很容易映射到它们各自的字节大小。对于 NUMERIC 和 TEXT，事情有点棘手，但我们稍后需要解决这个问题。现在，考虑恒定大小转换及其可能对对齐产生的影响。

## 一个小的填充空间

为了避免这种疯狂，Postgres 会填充每个较小的列以匹配下一个连续列的大小。由于这种特殊的列排列，几乎每个列对之间都有少量的缓冲。

让我们用一百万行填充表格并检查结果大小：

```sql=
INSERT INTO user_order (
    is_shipped, user_id, order_total, order_dt, order_type,
    ship_dt, item_ct, ship_cost, receive_dt, tracking_cd
)
SELECT true, 1000, 500.00, now() - INTERVAL '7 days',
       3, now() - INTERVAL '5 days', 10, 4.99,
       now() - INTERVAL '3 days', 'X5901324123479RROIENSTBKCV4'
  FROM generate_series(1, 1000000);

SELECT pg_relation_size('user_order') AS size_bytes,
       pg_size_pretty(pg_relation_size('user_order')) AS size_pretty;

 size_bytes | size_pretty 
------------+-------------
  141246464 | 135 MB
```

现在我们可以将其用作我们想要超越的值的基准。下一个问题是：其中有多少是填充？好吧，根据声明的对齐，我们估算了我们浪费的最小值：

* is_shipped 和 user_id 之间有 7 个字节
* order_total 和 order_dt 之间有 4 个字节
* order_type 和 ship_dt 之间有 6 个字节
* receive_dt 和 id 之间有 4 个字节

因此，我们可能每行损失大约 21 个字节，声明的类型占用 59 个字节的实际空间，总行长为 80 个字节（不含行开销）。同样，这仅基于对齐存储。事实证明，NUMERIC 和 TEXT 列对总数的影响比对齐建议的要大一些，但这对于估计来说还算不错。

如果我们更接近，这意味着我们可能能够将表缩小 26%，即大约 37MB。

## 一些基本规则

执行此类操作的诀窍是获得理想的列对齐，从而增加的字节数最少。为此，我们确实需要考虑 NUMERIC 和 TEXT 列。由于这些是可变长度类型，因此它们会得到特殊处理。例如，考虑关于 NUMERIC 的以下情况：

```sql=
SELECT pg_column_size(row()) AS empty_row,
       pg_column_size(row(0::NUMERIC)) AS no_val,
       pg_column_size(row(1::NUMERIC)) AS no_dec,
       pg_column_size(row(9.9::NUMERIC)) AS with_dec,
       pg_column_size(row(1::INT2, 1::NUMERIC)) AS col2,
       pg_column_size(row(1::INT4, 1::NUMERIC)) AS col4,
       pg_column_size(row(1::NUMERIC, 1::INT4)) AS round8;

 empty_row | no_val | no_dec | with_dec | col2 | col4 | round8 
-----------+--------+--------+----------+------+------+--------
        24 |     27 |     29 |       31 |   31 |   33 |     36
```

这些结果表明，我们可以将 NUMERIC 视为未对齐的，但有一些注意事项。NUMERIC 中的单个数字也需要 5 个字节，但它也不会像 INT8 那样影响前一列。

以下是与 TEXT 相同的概念：

```sql=
SELECT pg_column_size(row()) AS empty_row,
       pg_column_size(row(''::TEXT)) AS no_text,
       pg_column_size(row('a'::TEXT)) AS min_text,
       pg_column_size(row(1::INT4, 'a'::TEXT)) AS two_col,
       pg_column_size(row('a'::TEXT, 1::INT4)) AS round4;

 empty_row | no_text | min_text | two_col | round4 
-----------+---------+----------+---------+--------
        24 |      25 |       26 |      30 |     32
```

从中我们可以看出，变量类型根据下一列的类型四舍五入到最接近的 4 个字节。这意味着我们可以整天链接可变长度的列，而不会在右边界之外引入填充。因此，我们可以推断，只要可变长度的列位于列列表的末尾，就不会引入膨胀。

如果恒定长度的列类型根据下一列进行调整，则意味着最大类型应该排在前面。除此之外，我们可以“打包”列，使连续的列累计消耗 8 个字节。

再次，我们可以看到实际效果：

```sql=
SELECT pg_column_size(row()) AS empty_row,
       pg_column_size(row(1::SMALLINT)) AS int2,
       pg_column_size(row(1::INT)) AS int4,
       pg_column_size(row(1::BIGINT)) AS int8,
       pg_column_size(row(1::SMALLINT, 1::BIGINT)) AS padded,
       pg_column_size(row(1::INT, 1::INT, 1::BIGINT)) AS not_padded;

 empty_row | int2 | int4 | int8 | padded | not_padded 
-----------+------+------+------+--------+------------
        24 |   26 |   28 |   32 |     40 |         40
```

## 列俄罗斯方块

前面几节是“按 pg_type 中定义的类型长度对列进行排序”的一种花哨说法。幸运的是，我们可以通过稍微调整用于输出列类型的查询来获取该信息：

```sql=
SELECT a.attname, t.typname, t.typalign, t.typlen
  FROM pg_class c
  JOIN pg_attribute a ON (a.attrelid = c.oid)
  JOIN pg_type t ON (t.oid = a.atttypid)
 WHERE c.relname = 'user_order'
   AND a.attnum >= 0
 ORDER BY t.typlen DESC;

   attname   |   typname   | typalign | typlen 
-------------+-------------+----------+--------
 id          | int8        | d        |      8
 user_id     | int8        | d        |      8
 order_dt    | timestamptz | d        |      8
 ship_dt     | timestamptz | d        |      8
 receive_dt  | timestamptz | d        |      8
 item_ct     | int4        | i        |      4
 order_type  | int2        | s        |      2
 is_shipped  | bool        | c        |      1
 tracking_cd | text        | i        |     -1
 ship_cost   | numeric     | i        |     -1
 order_total | numeric     | i        |     -1
```

在其他条件相同的情况下，我们可以混合搭配具有匹配类型长度的列，如果我们够大胆或想要更漂亮的列排序，可以在必要时组合长度较短的类型。

让我们看看如果采用这种表格设计会发生什么(**表设计2**)：

```sql=
DROP TABLE user_order;

CREATE TABLE user_order (
  id            BIGSERIAL PRIMARY KEY NOT NULL,
  user_id       BIGINT NOT NULL,
  order_dt      TIMESTAMPTZ NOT NULL,
  ship_dt       TIMESTAMPTZ,
  receive_dt    TIMESTAMPTZ,
  item_ct       INT NOT NULL,
  order_type    SMALLINT NOT NULL,
  is_shipped    BOOLEAN NOT NULL DEFAULT false,
  order_total   NUMERIC NOT NULL,
  ship_cost     NUMERIC,
  tracking_cd   TEXT
);
```

如果我们重复之前插入的 100 万行，新表的大小为 117,030,912 字节，或大约 112MB。通过简单地重新组织表列，我们节省了 21% 的总空间。

这对于单个表来说可能意义不大，但对数据库实例中的每个表重复此操作，可以大大减少存储消耗。在数据仓库环境中，数据通常只加载一次，永远不会再修改，仅由于涉及的规模，10-20% 的存储减少是值得考虑的。我见过 60TB 的 Postgres 数据库；想象一下，在不实际删除任何数据的情况下将其减少 6-12TB。

## 已解决的差异

就像用岩石、鹅卵石和沙子填充罐子一样，声明 Postgres 表的最有效方法是通过列对齐类型。首先是较大的列，然后是中等列，最后是小列，而像 NUMERIC 和 TEXT 这样的奇怪例外则被钉在最后，就像我们类比中的灰尘一样。这就是我们使用指针的结果。

就目前而言，对表列进行更“自然”的声明可能如下：

```sql=
CREATE TABLE user_order (
  id            BIGSERIAL PRIMARY KEY NOT NULL,
  user_id       BIGINT NOT NULL,
  order_type    SMALLINT NOT NULL,
  order_total   NUMERIC NOT NULL,
  order_dt      TIMESTAMPTZ NOT NULL,
  item_ct       INT NOT NULL,
  ship_dt       TIMESTAMPTZ,
  is_shipped    BOOLEAN NOT NULL DEFAULT false,
  ship_cost     NUMERIC,
  tracking_cd   TEXT,
  receive_dt    TIMESTAMPTZ
);
```

在这种情况下，我们恰好离理想值有 8MB 的差距，或者说大约大了 7.7%。为了演示，我们故意弄乱了列排序以说明这一点。真正的表可能介于最佳情况和最坏情况之间，而确保泄漏最少的唯一方法是手动重新排序列。

有些人可能会问为什么 Postgres 没有内置此功能。它当然知道理想的列排序，并且有能力将用户的可见映射与实际到达磁盘的内容分离。这是一个合理的问题，但回答起来要困难得多，而且涉及大量的知识。

将物理表示与逻辑表示分离的一个主要好处是 Postgres 最终将允许列重新排序，或在特定位置添加列。如果用户希望他们的列列表在多次修改后看起来很漂亮，为什么不让他们这样做呢？

这都是关于优先级的。至少从 2006 年起，就有一项 TODO 事项来解决这个问题。从那时起，补丁就不断出现，而且每次讨论最终都无法得出明确的结论。这显然是一个难以解决的问题，正如人们所说，还有更重要的事情要做。

如果有足够的需求，有人会赞助一个补丁直到完成，即使它需要多个 Postgres 版本才能体现必要的底层更改。在此之前，如果影响对于特定用例足够紧迫，一个简单的查询就可以神奇地揭示理想的列顺序。


---

# 字段对齐思路

关于这个问题的讨论由来已久，有兴趣的小伙伴可以查看：

- <u>[Postgres 表中列的顺序会影响性能吗？](https://stackoverflow.org.cn/questions/12604744)</u>
- <u>[Calculating and saving space in PostgreSQL](https://stackoverflow.com/questions/2966524/calculating-and-saving-space-in-postgresql/7431468)</u>



## SQL/视图

有人在上面SQL的基础上做了一些新的想法：
```sql=
SELECT a.attname, t.typname, t.typalign, t.typlen
  FROM pg_class c
  JOIN pg_attribute a ON (a.attrelid = c.oid)
  JOIN pg_type t ON (t.oid = a.atttypid)
 WHERE c.relname = 'user_order'
   AND a.attnum >= 0
 ORDER BY t.typlen DESC;
```

如下：

```sql=
CREATE OR REPLACE VIEW tabletetris
    AS SELECT n.nspname, c.relname,
        a.attname, t.typname, t.typstorage, t.typalign, t.typlen
    FROM pg_class c
    JOIN pg_namespace n ON (n.oid = c.relnamespace)
    JOIN pg_attribute a ON (a.attrelid = c.oid)
    JOIN pg_type t ON (t.oid = a.atttypid)
    WHERE a.attnum >= 0
    ORDER BY n.nspname ASC, c.relname ASC,
        t.typlen DESC, t.typalign DESC, a.attnum ASC;

-- 使用如下：
SELECT * FROM tabletetris WHERE relname='mytablename';

-- 作者的想法：
-- 但是您可以在 nspname（表所在的架构）上添加过滤器。
-- 我还添加了存储类型，这是有用的信息，有助于确定要内联哪些 -1 和/或在哪里排序，并保持现有列的相对顺序，否则将使用相同的排序键。 
```


## postgres_dba

这是一个关于 Erwin 的列重新排序建议的很酷的工具：

- <u>[postgres_dba github链接，点击前往](https://github.com/NikolayS/postgres_dba)</u>

准确来说(对于列排序建议)，是它的命令`p1` 如下：

![image](https://hackmd.io/_uploads/HyPDBOfpA.png)

下面我们来看一下其内部实现(关于其他子命令，有兴趣的小伙伴可以自行了解)，如下：

![image](https://hackmd.io/_uploads/HyRBU_GpC.png)

主要就是这个`\sql\p1_alignment_padding.sql`脚本，我们可以直接在psql上执行即可，然后它会自动向您显示所有表上的列重新排序的真正潜力：


```sql=
postgres=# CREATE TABLE user_order (
postgres(#   is_shipped    BOOLEAN NOT NULL DEFAULT false,
postgres(#   user_id       BIGINT NOT NULL,
postgres(#   order_total   NUMERIC NOT NULL,
postgres(#   order_dt      TIMESTAMPTZ NOT NULL,
postgres(#   order_type    SMALLINT NOT NULL,
postgres(#   ship_dt       TIMESTAMPTZ,
postgres(#   item_ct       INT NOT NULL,
postgres(#   ship_cost     NUMERIC,
postgres(#   receive_dt    TIMESTAMPTZ,
postgres(#   tracking_cd   TEXT,
postgres(#   id            BIGSERIAL PRIMARY KEY NOT NULL
postgres(# );
CREATE TABLE
postgres=# INSERT INTO user_order (
postgres(#     is_shipped, user_id, order_total, order_dt, order_type,
postgres(#     ship_dt, item_ct, ship_cost, receive_dt, tracking_cd
postgres(# )
postgres-# SELECT true, 1000, 500.00, now() - INTERVAL '7 days',
postgres-#        3, now() - INTERVAL '5 days', 10, 4.99,
postgres-#        now() - INTERVAL '3 days', 'X5901324123479RROIENSTBKCV4'
postgres-#   FROM generate_series(1, 1000000);
INSERT 0 1000000
postgres=# 
postgres=# \i /home/postgres/test/bin/p1_alignment_padding.sql
   Table    | Table Size |     Comment      |    Wasted *     |      Suggested Columns Reorder      
------------+------------+------------------+-----------------+-------------------------------------
 user_order | 135 MB     | Includes VARLENA | ~15 MB (11.32%) | is_shipped, order_type, item_ct    +
            |            |                  |                 | order_total, ship_cost, tracking_cd+
            |            |                  |                 | id, order_dt, receive_dt           +
            |            |                  |                 | ship_dt, user_id
(1 row)

postgres=#
```

注：这里的浪费值是一个估计量，然后 我们再试一下上面的 设计2，如下：
```sql=
postgres=# \d+ user_order
                                                                  Table "public.user_order"
   Column    |           Type           | Collation | Nullable |                Default                 | Storage  | Compression | Stats target | Description 
-------------+--------------------------+-----------+----------+----------------------------------------+----------+-------------+--------------+-------------
 id          | bigint                   |           | not null | nextval('user_order_id_seq'::regclass) | plain    |             |              | 
 user_id     | bigint                   |           | not null |                                        | plain    |             |              | 
 order_dt    | timestamp with time zone |           | not null |                                        | plain    |             |              | 
 ship_dt     | timestamp with time zone |           |          |                                        | plain    |             |              | 
 receive_dt  | timestamp with time zone |           |          |                                        | plain    |             |              | 
 item_ct     | integer                  |           | not null |                                        | plain    |             |              | 
 order_type  | smallint                 |           | not null |                                        | plain    |             |              | 
 is_shipped  | boolean                  |           | not null | false                                  | plain    |             |              | 
 order_total | numeric                  |           | not null |                                        | main     |             |              | 
 ship_cost   | numeric                  |           |          |                                        | main     |             |              | 
 tracking_cd | text                     |           |          |                                        | extended |             |              | 
Indexes:
    "user_order_pkey" PRIMARY KEY, btree (id)
Access method: heap

postgres=# \i /home/postgres/test/bin/p1_alignment_padding.sql
   Table    | Table Size |     Comment      | Wasted * | Suggested Columns Reorder 
------------+------------+------------------+----------+---------------------------
 user_order | 112 MB     | Includes VARLENA |          | 
(1 row)

postgres=#
```

然后我们试一下它建议的顺序，如下(**表设计3**)：

```sql=
postgres=# CREATE TABLE user_order (
postgres(#   is_shipped    BOOLEAN NOT NULL DEFAULT false,
postgres(#   order_type    SMALLINT NOT NULL,
postgres(#   item_ct       INT NOT NULL,
postgres(#   order_total   NUMERIC NOT NULL,
postgres(#   ship_cost     NUMERIC,
postgres(#   tracking_cd   TEXT,
postgres(#   id            BIGSERIAL PRIMARY KEY NOT NULL,
postgres(#   order_dt      TIMESTAMPTZ NOT NULL,
postgres(#   receive_dt    TIMESTAMPTZ,
postgres(#   ship_dt       TIMESTAMPTZ,
postgres(#   user_id       BIGINT NOT NULL
postgres(# );
CREATE TABLE
postgres=# 
postgres=# INSERT INTO user_order (
    is_shipped, user_id, order_total, order_dt, order_type,
    ship_dt, item_ct, ship_cost, receive_dt, tracking_cd
)
SELECT true, 1000, 500.00, now() - INTERVAL '7 days',
       3, now() - INTERVAL '5 days', 10, 4.99,
       now() - INTERVAL '3 days', 'X5901324123479RROIENSTBKCV4'
  FROM generate_series(1, 1000000);
INSERT 0 1000000
postgres=# SELECT pg_relation_size('user_order') AS size_bytes,
postgres-#        pg_size_pretty(pg_relation_size('user_order')) AS size_pretty;
 size_bytes | size_pretty 
------------+-------------
  117030912 | 112 MB
(1 row)

postgres=# \i /home/postgres/test/bin/p1_alignment_padding.sql
   Table    | Table Size |     Comment      | Wasted * | Suggested Columns Reorder 
------------+------------+------------------+----------+---------------------------
 user_order | 112 MB     | Includes VARLENA |          | 
(1 row)

postgres=#
```

注：以上两种方式在平时的使用都是可以的，具体方案由大家自行选择。ps：两种看似不同的列顺序，在最终的效果上**几乎一致** 值得深究！


# 字段对齐原则

这块内容大家可以自行查看`pg_type`系统表，也可以使用`pg_column_size`函数进行上面的实验进行测试。下面是一位老哥的博客，请大家自行阅读：

- <u>[字段对齐规则，点击前往](https://yieldnull.com/blog/08a509370a04d5051eddbe84e15bb186b9ddcbad/)</u>


接下来我想就上面两种推荐的列顺序(不同)但能达到同样效果做一个分析(<u>如何计算和规避padding</u>)，如下：

```sql=
-- 表设计1
postgres=# \d+ user_order
                                                                  Table "public.user_order"
   Column    |           Type           | Collation | Nullable |                Default                 | Storage  | Compression | Stats target | Description 
-------------+--------------------------+-----------+----------+----------------------------------------+----------+-------------+--------------+-------------
 is_shipped  | boolean                  |           | not null | false                                  | plain    |             |              | 
 user_id     | bigint                   |           | not null |                                        | plain    |             |              | 
 order_total | numeric                  |           | not null |                                        | main     |             |              | 
 order_dt    | timestamp with time zone |           | not null |                                        | plain    |             |              | 
 order_type  | smallint                 |           | not null |                                        | plain    |             |              | 
 ship_dt     | timestamp with time zone |           |          |                                        | plain    |             |              | 
 item_ct     | integer                  |           | not null |                                        | plain    |             |              | 
 ship_cost   | numeric                  |           |          |                                        | main     |             |              | 
 receive_dt  | timestamp with time zone |           |          |                                        | plain    |             |              | 
 tracking_cd | text                     |           |          |                                        | extended |             |              | 
 id          | bigint                   |           | not null | nextval('user_order_id_seq'::regclass) | plain    |             |              | 
Indexes:
    "user_order_pkey" PRIMARY KEY, btree (id)
Access method: heap

postgres=#
```

然后插入一行，如下：

```sql=
INSERT INTO user_order (
    is_shipped, user_id, order_total, order_dt, order_type,
    ship_dt, item_ct, ship_cost, receive_dt, tracking_cd
)
SELECT true, 1000, 500.00, now() - INTERVAL '7 days',
       3, now() - INTERVAL '5 days', 10, 4.99,
       now() - INTERVAL '3 days', 'X5901324123479RROIENSTBKCV4';
       
postgres=# table user_order;
-[ RECORD 1 ]------------------------------
is_shipped  | t
user_id     | 1000
order_total | 500.00
order_dt    | 2024-09-06 22:36:10.406433-07
order_type  | 3
ship_dt     | 2024-09-08 22:36:10.406433-07
item_ct     | 10
ship_cost   | 4.99
receive_dt  | 2024-09-10 22:36:10.406433-07
tracking_cd | X5901324123479RROIENSTBKCV4
id          | 1

postgres=#
```

---

```sql=
-- 按照表设计的顺序 依次计算：
postgres=# select pg_column_size(user_order.*), pg_column_size(is_shipped),pg_column_size(user_id),pg_column_size(order_total),pg_column_size(order_dt),pg_column_size(order_type),pg_column_size(ship_dt),pg_column_size(item_ct),pg_column_size(ship_cost),pg_column_size(receive_dt),pg_column_size(tracking_cd),pg_column_size(id) from user_order;
-[ RECORD 1 ]--+----
pg_column_size | 136
pg_column_size | 1
pg_column_size | 8
pg_column_size | 5
pg_column_size | 8
pg_column_size | 2
pg_column_size | 8
pg_column_size | 4
pg_column_size | 7
pg_column_size | 8
pg_column_size | 28
pg_column_size | 8

postgres=#
postgres=# select pg_column_size(user_order.*), 
postgres-#    pg_column_size(row(is_shipped)),
postgres-#    pg_column_size(row(is_shipped, user_id)),
postgres-#    pg_column_size(row(is_shipped, user_id, order_total)),
postgres-#    pg_column_size(row(is_shipped, user_id, order_total, order_dt)),
postgres-#    pg_column_size(row(is_shipped, user_id, order_total, order_dt, order_type)),
postgres-#    pg_column_size(row(is_shipped, user_id, order_total, order_dt, order_type, ship_dt)),
postgres-#    pg_column_size(row(is_shipped, user_id, order_total, order_dt, order_type, ship_dt, item_ct)),
postgres-#    pg_column_size(row(is_shipped, user_id, order_total, order_dt, order_type, ship_dt, item_ct, ship_cost)),
postgres-#    pg_column_size(row(is_shipped, user_id, order_total, order_dt, order_type, ship_dt, item_ct, ship_cost, receive_dt)),
postgres-#    pg_column_size(row(is_shipped, user_id, order_total, order_dt, order_type, ship_dt, item_ct, ship_cost, receive_dt, tracking_cd)),
postgres-#    pg_column_size(row(is_shipped, user_id, order_total, order_dt, order_type, ship_dt, item_ct, ship_cost, receive_dt, tracking_cd, id)) from user_order;
-[ RECORD 1 ]--+----
pg_column_size | 136
pg_column_size | 25
pg_column_size | 40
pg_column_size | 45
pg_column_size | 56
pg_column_size | 58
pg_column_size | 72
pg_column_size | 76
pg_column_size | 83
pg_column_size | 96
pg_column_size | 124
pg_column_size | 136

postgres=#
```

这么一行 总的size是136，可是这些相加才 111，那么少的25 如下(**在这一行数据的基础上**)：

![image](https://hackmd.io/_uploads/BypMbnG60.png)

---

下面还是这一行数据，设计2 如下：

```sql=
postgres=# CREATE TABLE user_order (
postgres(#   id            BIGSERIAL PRIMARY KEY NOT NULL,
postgres(#   user_id       BIGINT NOT NULL,
postgres(#   order_dt      TIMESTAMPTZ NOT NULL,
postgres(#   ship_dt       TIMESTAMPTZ,
postgres(#   receive_dt    TIMESTAMPTZ,
postgres(#   item_ct       INT NOT NULL,
postgres(#   order_type    SMALLINT NOT NULL,
postgres(#   is_shipped    BOOLEAN NOT NULL DEFAULT false,
postgres(#   order_total   NUMERIC NOT NULL,
postgres(#   ship_cost     NUMERIC,
postgres(#   tracking_cd   TEXT
postgres(# );
CREATE TABLE
postgres=# 
postgres=# INSERT INTO user_order (
postgres(#     is_shipped, user_id, order_total, order_dt, order_type,
postgres(#     ship_dt, item_ct, ship_cost, receive_dt, tracking_cd
postgres(# )
postgres-# SELECT true, 1000, 500.00, now() - INTERVAL '7 days',
postgres-#        3, now() - INTERVAL '5 days', 10, 4.99,
postgres-#        now() - INTERVAL '3 days', 'X5901324123479RROIENSTBKCV4';
INSERT 0 1
postgres=# select pg_column_size(user_order.*), pg_column_size(id),pg_column_size(user_id),pg_column_size(order_dt),pg_column_size(ship_dt),pg_column_size(receive_dt),pg_column_size(item_ct),pg_column_size(order_type),pg_column_size(is_shipped),pg_column_size(order_total),pg_column_size(ship_cost),pg_column_size(tracking_cd) from user_order;
-[ RECORD 1 ]--+----
pg_column_size | 111
pg_column_size | 8
pg_column_size | 8
pg_column_size | 8
pg_column_size | 8
pg_column_size | 8
pg_column_size | 4
pg_column_size | 2
pg_column_size | 1
pg_column_size | 5
pg_column_size | 7
pg_column_size | 28

postgres=#
postgres=# select pg_column_size(user_order.*),
postgres-#    pg_column_size(row(id)),
postgres-#    pg_column_size(row(id, user_id)),
postgres-#    pg_column_size(row(id, user_id, order_dt)),
postgres-#    pg_column_size(row(id, user_id, order_dt, ship_dt)),
postgres-#    pg_column_size(row(id, user_id, order_dt, ship_dt, receive_dt)),
postgres-#    pg_column_size(row(id, user_id, order_dt, ship_dt, receive_dt, item_ct)),
postgres-#    pg_column_size(row(id, user_id, order_dt, ship_dt, receive_dt, item_ct, order_type)),
postgres-#    pg_column_size(row(id, user_id, order_dt, ship_dt, receive_dt, item_ct, order_type, is_shipped)),
postgres-#    pg_column_size(row(id, user_id, order_dt, ship_dt, receive_dt, item_ct, order_type, is_shipped, order_total)),
postgres-#    pg_column_size(row(id, user_id, order_dt, ship_dt, receive_dt, item_ct, order_type, is_shipped, order_total, ship_cost)),
postgres-#    pg_column_size(row(id, user_id, order_dt, ship_dt, receive_dt, item_ct, order_type, is_shipped, order_total, ship_cost, tracking_cd)) from user_order;
-[ RECORD 1 ]--+----
pg_column_size | 111
pg_column_size | 32
pg_column_size | 40
pg_column_size | 48
pg_column_size | 56
pg_column_size | 64
pg_column_size | 68
pg_column_size | 70
pg_column_size | 71
pg_column_size | 76
pg_column_size | 83
pg_column_size | 111

postgres=#
```

如上这么一行 总的size是111，列顺序堪称完美 无须进行填充！

---

设计3，如下(**这样的设计 多想一想俄罗斯方块**)：

```sql=
postgres=# CREATE TABLE user_order (
postgres(#   is_shipped    BOOLEAN NOT NULL DEFAULT false,
postgres(#   order_type    SMALLINT NOT NULL,
postgres(#   item_ct       INT NOT NULL,
postgres(#   order_total   NUMERIC NOT NULL,
postgres(#   ship_cost     NUMERIC,
postgres(#   tracking_cd   TEXT,
postgres(#   id            BIGSERIAL PRIMARY KEY NOT NULL,
postgres(#   order_dt      TIMESTAMPTZ NOT NULL,
postgres(#   receive_dt    TIMESTAMPTZ,
postgres(#   ship_dt       TIMESTAMPTZ,
postgres(#   user_id       BIGINT NOT NULL
postgres(# );
CREATE TABLE
postgres=# 
postgres=# INSERT INTO user_order (
    is_shipped, user_id, order_total, order_dt, order_type,
    ship_dt, item_ct, ship_cost, receive_dt, tracking_cd
)
SELECT true, 1000, 500.00, now() - INTERVAL '7 days',
       3, now() - INTERVAL '5 days', 10, 4.99,
       now() - INTERVAL '3 days', 'X5901324123479RROIENSTBKCV4';
INSERT 0 1
postgres=# 
postgres=# select pg_column_size(user_order.*),
postgres-#    pg_column_size(row(is_shipped)),
postgres-#    pg_column_size(row(is_shipped, order_type)),
postgres-#    pg_column_size(row(is_shipped, order_type, item_ct)),
postgres-#    pg_column_size(row(is_shipped, order_type, item_ct, order_total)),
postgres-#    pg_column_size(row(is_shipped, order_type, item_ct, order_total, ship_cost)),
postgres-#    pg_column_size(row(is_shipped, order_type, item_ct, order_total, ship_cost, tracking_cd)),
postgres-#    pg_column_size(row(is_shipped, order_type, item_ct, order_total, ship_cost, tracking_cd, id)),
postgres-#    pg_column_size(row(is_shipped, order_type, item_ct, order_total, ship_cost, tracking_cd, id, order_dt)),
postgres-#    pg_column_size(row(is_shipped, order_type, item_ct, order_total, ship_cost, tracking_cd, id, order_dt, receive_dt)),
postgres-#    pg_column_size(row(is_shipped, order_type, item_ct, order_total, ship_cost, tracking_cd, id, order_dt, receive_dt, ship_dt)),
postgres-#    pg_column_size(row(is_shipped, order_type, item_ct, order_total, ship_cost, tracking_cd, id, order_dt, receive_dt, ship_dt, user_id)) from user_order;
-[ RECORD 1 ]--+----
pg_column_size | 112
pg_column_size | 25
pg_column_size | 28
pg_column_size | 32
pg_column_size | 37
pg_column_size | 44
pg_column_size | 72
pg_column_size | 80
pg_column_size | 88
pg_column_size | 96
pg_column_size | 104
pg_column_size | 112

postgres=# select pg_column_size(user_order.*), pg_column_size(is_shipped),pg_column_size(order_type),pg_column_size(item_ct),pg_column_size(order_total),pg_column_size(ship_cost),pg_column_size(tracking_cd),pg_column_size(id),pg_column_size(order_dt),pg_column_size(receive_dt),pg_column_size(ship_dt),pg_column_size(user_id) from user_order;
-[ RECORD 1 ]--+----
pg_column_size | 112
pg_column_size | 1
pg_column_size | 2
pg_column_size | 4
pg_column_size | 5
pg_column_size | 7
pg_column_size | 28
pg_column_size | 8
pg_column_size | 8
pg_column_size | 8
pg_column_size | 8
pg_column_size | 8

postgres=#
```

这么一行 总的size是112，可是这些相加才 111，发生了1处填充：

![image](https://hackmd.io/_uploads/rko363MTA.png)

---

若是这一行内容，如下(改了一个 numeric 类型的值：4.99 7字节 -> 5.00 5字节)：

```sql=
postgres=# INSERT INTO user_order (
postgres(#     is_shipped, user_id, order_total, order_dt, order_type,
postgres(#     ship_dt, item_ct, ship_cost, receive_dt, tracking_cd
postgres(# )
postgres-# SELECT true, 1000, 500.00, now() - INTERVAL '7 days',
postgres-#        3, now() - INTERVAL '5 days', 10, 5.00,
postgres-#        now() - INTERVAL '3 days', 'X5901324123479RROIENSTBKCV4';
INSERT 0 1
postgres=#
```

```sql!
postgres=# select pg_column_size(user_order.*), pg_column_size(is_shipped),pg_column_size(order_type),pg_column_size(item_ct),pg_column_size(order_total),pg_column_size(ship_cost),pg_column_size(tracking_cd),pg_column_size(id),pg_column_size(order_dt),pg_column_size(receive_dt),pg_column_size(ship_dt),pg_column_size(user_id) from user_order;
-[ RECORD 1 ]--+----
pg_column_size | 112
pg_column_size | 1
pg_column_size | 2
pg_column_size | 4
pg_column_size | 5
pg_column_size | 5
pg_column_size | 28
pg_column_size | 8
pg_column_size | 8
pg_column_size | 8
pg_column_size | 8
pg_column_size | 8

postgres=# select pg_column_size(user_order.*),
postgres-#    pg_column_size(row(is_shipped)),
postgres-#    pg_column_size(row(is_shipped, order_type)),
postgres-#    pg_column_size(row(is_shipped, order_type, item_ct)),
postgres-#    pg_column_size(row(is_shipped, order_type, item_ct, order_total)),
postgres-#    pg_column_size(row(is_shipped, order_type, item_ct, order_total, ship_cost)),
postgres-#    pg_column_size(row(is_shipped, order_type, item_ct, order_total, ship_cost, tracking_cd)),
postgres-#    pg_column_size(row(is_shipped, order_type, item_ct, order_total, ship_cost, tracking_cd, id)),
postgres-#    pg_column_size(row(is_shipped, order_type, item_ct, order_total, ship_cost, tracking_cd, id, order_dt)),
postgres-#    pg_column_size(row(is_shipped, order_type, item_ct, order_total, ship_cost, tracking_cd, id, order_dt, receive_dt)),
postgres-#    pg_column_size(row(is_shipped, order_type, item_ct, order_total, ship_cost, tracking_cd, id, order_dt, receive_dt, ship_dt)),
postgres-#    pg_column_size(row(is_shipped, order_type, item_ct, order_total, ship_cost, tracking_cd, id, order_dt, receive_dt, ship_dt, user_id)) from user_order;
-[ RECORD 1 ]--+----
pg_column_size | 112
pg_column_size | 25
pg_column_size | 28
pg_column_size | 32
pg_column_size | 37
pg_column_size | 42
pg_column_size | 70
pg_column_size | 80
pg_column_size | 88
pg_column_size | 96
pg_column_size | 104
pg_column_size | 112

postgres=#
```
这么一行 总的size是112，可是这些相加才 109，发生了2处填充：

![image](https://hackmd.io/_uploads/H1XfMpGp0.png)

---

:::success
小结一下：
1. 当数据自然对齐时，CPU 可以有效地执行对内存的读取和写入。因此，PostgreSQL 中的每种数据类型都有特定的对齐要求。当多个属性连续存储在一个 Tuples 中时，会在属性之前插入填充，以便它从所需的对齐边界开始。更好地了解这些对齐要求可能有助于在磁盘上存储 Tuples 时最大限度地减少所需的填充量，从而节省磁盘空间
2. 想象一下：在包含数百万行的关系中，每个 Tuples 保存几个字节可能会导致节省一些重要的存储空间。此外，如果我们可以在数据页中容纳更多的元组，则可以由于更少的 I/O 活动而提高性能
3. 上面SQL和脚本非常巧妙，可以在平时的使用中加以利用
4. 正如《On Rocks and Sand》所说：一个真实的 table 可能介于最好和最坏的情况之间 过分追求实则大可不必(主要是不便阅读)
5. 关于numeric text等类型这里不再赘述(多使用`pg_column_size`实验即可)，其他类型有兴趣的可以深入
:::