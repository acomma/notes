结合[读《SQL反模式》 - 方跃明的文章 - 知乎](https://zhuanlan.zhihu.com/p/37534634)一起阅读效果更佳。

## 第 2 章 乱穿马路

目标：存储多值属性

>在一列中存储一系列相关数据的集合。

反模式：格式化的逗号分隔列表

>使用逗号分隔的列表来避免在多对多的关系中创建交叉表。

最大的弊端在于如何选择合适的分隔符。

解决方案：创建一张交叉表。

    联合表、多对多表、映射表都是指同一个东西。

本质是范式化，用大白话讲就是建立多对多的关系。针对的是已经存在了两张表，需要建立两张表之间的关系这种情况，比如在 RBAC 模型中用户和角色的关系，可以选择反模式在用户表表中存储用户拥有哪些角色，但最好还是创建一张交叉表来保存这种关系。这里的场景区别于“第 8 章 多列属性”描述的场景。

## 第 3 章 单纯的树

目标：分层存储与查询

这个问题太常见了，组织架构图、后台系统菜单、用户评论与回复等。

反模式：总是依赖父节点

>添加 `parent_id` 字段。

也叫邻接表。

解决方案：使用其他树模型

    路径枚举、嵌套集、闭包表

它们各有优劣，闭包表简单、优雅、更通用。

## 第 4 章 需要 ID

每个表都有一个主键，通常使用无意义的称为*伪主键*或*代理键*的 `id` 表示。作者认为 `id` 不是一个好的名字，更好的名字应该是明确的，比如 `bug_id` 这样的名字。

从实践上来看，应该是两者的结合，每个表都有一个伪主键，它由数据库生成，比如 `MySQL` 数据库的 `AUTO_INCREMENT`，它与业务无关。另外再加一个自己生成的唯一 ID 作为表之间关联关系使用的列。

```sql
CREATE TABLE `bug` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `bug_id` varchar(20) COLLATE utf8_bin NOT NULL,
  # 其他字段
  PRIMARY KEY (`id`) USING BTREE,
  UNIQUE KEY `unique_bug` (`bug_id`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin;
```

## 第 5 章 不用钥匙的入口

本章的观点总结起来就是通过*声明约束*来保证*引用完整性*，也就是 `FOREIGN KEY` 和 `ON UPDATE CASCADE`、`ON DELETE RESTRICT`、`ON DELETE SET DEFAULT` 的应用。但是在互联网公司一般是通过程序来保证引用完整性。

## 第 6 章 实体-属性-值

目标：支持可变的属性

反模式：使用泛型属性表

也就是*实体-属性-值*（EVA）。

```sql
CREATE TABLE `issue` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `issue_id` varchar(20) COLLATE utf8_bin NOT NULL,
  # 其他字段
  PRIMARY KEY (`id`) USING BTREE,
  UNIQUE KEY `unique_issue` (`issue_id`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin;

CREATE TABLE `attribute` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `issue_id` varchar(20) COLLATE utf8_bin NOT NULL,
  `attr_name` varchar(64) COLLATE utf8_bin NOT NULL,
  `attr_value` varchar(64) COLLATE utf8_bin,
  PRIMARY KEY (`id`) USING BTREE,
  UNIQUE KEY `unique_attribute` (`issue_id`,`attr_name`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin;
```

解决方案：模型化子类型

    单表继承、实体表继承、类表继承、半结构化数据模型

这几种方案在我们的订单系统中基本都得到了应用。订单系统有两种类型的订单：充值订单和消费订单，充值的时候会赠送优惠券、积分或者储值金。

在赠送表的设计上使用了单表继承，优惠券和积分的数量可以统一用一个整型字段来表示，储值金。就只能使用双精度浮点型字段来表示。

```sql
CREATE TABLE `presentation` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `presentation_id` varchar(20) COLLATE utf8_bin NOT NULL,
  `presentation_type` varchar(64) COLLATE utf8_bin NOT NULL,
  `quantity` integer(11),
  `amount` decimal(10,2),
  # 其他字段
  PRIMARY KEY (`id`) USING BTREE,
  UNIQUE KEY `unique_presentation` (`presentation_id`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin;
```

充值订单和消费订单都有商家信息、支付信息相关的字段，但是设计成了两张表，采用的即是实体表继承。

类表继承未使用到。

消费订单中有一些扩展信息，它们不重要但又必须存储，于是使用了一个扩展字段，里面存储了自定义的格式的数据 `key1@value1;key2@value2`，也就是半结构化的数据。没有使用书中说到的 XML 或 JSON 格式。

但是在电子商务相关的系统中，商品的属性可以考虑使用 EVA 设计。

## 第 7 章 多态关联

目标：引用多个父表

反模式：使用双用途外键

>也叫*多态关联*，*杂乱关联*

在我们的订单系统中使用了这个反模式。在订单系统中充值时可能赠送优惠券，消费时也可能赠送优惠券，在赠送表中增加了一个订单类型（order_type）字段来区分该条赠送计算是充值赠送的还是消费赠送的。

```sql
CREATE TABLE `presentation` (
  `id` bigint(20),
  `presentation_id` varchar(20),
  `presentation_type` varchar(64),
  `quantity` integer(11),
  `amount` decimal(10,2),
  `order_id` varchar(20),
  `order_type` varchar(32),
  # 其他字段
  PRIMARY KEY (`id`) USING BTREE,
  UNIQUE KEY `unique_presentation` (`presentation_id`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin;
```

解决方案：让关系变得简单

    反向引用、创建交叉表、创建共用的超级表

反向引用就是不在赠送表里保存订单 ID，而是在订单表里保存赠送 ID。

创建交叉表也是不在赠送表里保存订单 ID，但是它将订单 ID 和 赠送 ID 的关系保存到保存到独立的关系表中。

创建共用的超级表就是第 6 章说的类表继承，把充值订单和消费订单中公共的字段（包括订单 ID）提取到一张公共订单表中，赠送表保存这个公共订单表中的订单 ID 并去掉订单类型字段。

## 第 8 章 多列属性

目标：存储多值属性

反模式：创建多个列

解决方案：创建从属表

和“第 2 章 乱穿马路”是类似的，只是这里的场景是一开始不存在第二张表，只是为了范式化而创建从属表。创建多个列这种反模式有时也是可以的，比如一个 `bug` 有报告人、解决人、复核人，这三个人都来自系统的 `account` 表，这种情况就可以创建多个列来表示这种关系。

## 第 9 章 元数据分裂

目标：支持可扩展性

反模式：克隆表与克隆列

克隆表就是目前“分库分表”中的分表。

    克隆列就是将一列拆分成多个子列，使用别的列中的不同值给拆分出来的列命名。

目前还没见到克隆列这种实践。

解决方案：分区及标准化

    水平分区、垂直分区、创建关联表

这里的水平分区有别于“分库分表”相关的水平拆分的概念，这里的水平分区只利用数据库的分区特性，比如 MySQL 中的 `PARTITION BY...PARTITIONS...`。于“分库分表”相关的水平拆分就是上面说的克隆表。

垂直分区类似于“分库分表”相关的垂直拆分的概念，针对的是表中的列，将某些非常庞大的列或者很少使用的列抽出来独立成表，通过外键于主表做一对一的关联。

## 第 10 章 取整错误

在需要精确表示数字的场景，尽可能的选择 `NUMERIC` 或 `DECIMAL` 类型。

## 第 11 章 每日新花样

目标：限定列的有效值

反模式：在列定义上指定可选值

>比如对某一列定义一个检查约束项。在 MySQL 中可以使用 `ENUM` 关键字。其他的方案是*域*以及*用户自定义类型（UDT）*。

解决方案：在数据中指定值

>创建一张*检查表*，每一行包含一个允许在其他表的列中出现的候选值；然后定义一个外键约束，让其他表的列引用这个新表。

可以在检查表中加其他属性列来表示候选值得状态。

这有点像字典表。

## 第 12 章 幽灵文件

目标：存储图片或其他多媒体大文件

反模式：假设你必须使用文件系统

解决方案：在需要时使用 `BLOB` 类型

作者倾向于使用 `BLOB` 类型，但是在实际的实践中更多的是使用文件系统。

## 第 13 章 乱用索引

目标：优化性能

反模式：无规划地使用索引

>1. 不使用索引或索引不足
>2. 使用了太多的索引或者使用了一些无效索引
>3. 执行一些让索引无能为力的查询

注意*最左原则*可能导致有些查询方式无法使用索引和*低分离率索引*导致的索引使用效率低下。

解决方案：*MENTOR* 你的索引

    测量（Measure）、解释（Explain）、挑选（Nominate）、测试（Test）、优化（Optimize）、重建（Rebuild）
    
注意有时数据库并不是瓶颈。

虽然索引会一定程度引起 `INSERT`、`UPDATE` 和 `DELETE` 语句执行效率的下降，但相对索引带来的好处，这点效率是可接受的，即应该尽量使用索引。

## 第 14 章 对未知的恐惧

目标：辨别悬空值

反模式：将 `NULL` 作为普通的值，反之亦然

解决方案：将 `NULL` 视为特殊值

`NULL` 与 `0`、空字符串、`FALSE` 都是不一样的。应该在合适的场景使用合适的值，应该为未知的场景就应该使用 `NULL`。在我们的订单表的设计中，我们使用了一个特殊的 `00000000000000000000` 来表示非会员的 ID，而不是 `NULL`，这在作者看来不是一个好的实践。我们在表示金额、数量字段的地方使用默认值 `0`，方便计算和统计。

## 第 15 章 模棱两可的分组

目标：获取每组的最大值

反模式：引用分组列

    单值规则

>跟在 `SELECT` 之后的选择列表中的每一列，对于每个分组来说都必须返回且仅返回一个值。

解决方案：无歧义地使用列

简单地说，选择列表中只能出现分组列字段或应用了聚合函数的字段。

    使用关联子查询

```sql
SELECT bp1.product_id, b1.date_reported AS latest, b1.bug_id
FROM bugs b1 JOIN bugs_products bp1 USING (bug_id)                #1
WHERE NOT EXIST (                                                 #2
    SELECT * FROM bugs b2 JOIN bugs_products bp2 USING (bug_id)   #3
    WHERE bp1.product_id = bp2.product_id AND b1.date_reported < b2.date_reported
);
```

`#1` 和 `#3` 的 `JOIN` 构建了两个笛卡尔积，`#2` 的 `WHERE NOT EXIST` 类似于做了一个减法。

    使用衍生表

```sql
SELECT m.product_id, m.latest, b1.bug_id
FROM bugs b1 JOIN bugs_products bp1 USING (bug_id) JOIN (
    SELECT bp2.product_id, MAX(b2.date_reported) AS latest
    FROM bugs b2 JOIN bugs_products bp2 USING (bug_id)
    GROUP BY bp2.product_id
) m ON (bp1.product_id = m.product_id AND b1.date_reported = m.latest);
```

更进一步

```sql
SELECT m.product_id, m.latest, MAX(b1.bug_id) AS latest_bug_id
FROM bugs b1 JOIN (
    SELECT product_id, MAX(date_reported) AS latest
    FROM bugs b2 JOIN bugs_products USING (bug_id)
    GROUP BY product_id
) m ON (b1.date_reported = m.latest)
GROUP BY m.product_id, m.latest;
```

    使用 JOIN

```sql
SELECT bp1.product_id, b1.date_reported AS latest, b1.bug_id
FROM bugs b1 JOIN bugs_products bp1 ON (b1.bug_id = bp1.bug_id)
    LEFT OUTER JOIN (
        bugs AS b2 JOIN bugs_products AS bp2 ON (b2.bug_id = bp2.bug_id)
    ) ON (
        bp1.product_id = bp2.product_id AND (
            b1.date_reported < b2.date_reported
                OR b1.date_reported = b2.date_reported
                    AND b1.bug_id < b2.bug_id
        )
    )
WHERE b2.bug_id IS NULL;
```

总结起来就是做笛卡尔积，然后从笛卡尔积排除不满足条件的记录。

## 第 16 章 随机选择

目标：获取样本记录

反模式：随机排序

```sql
SELECT * FROM bugs ORDER BY RAND() LIMIT 1;
```

`RAND()` 被称为不定表达式，这种方案的弱点有：

1. 排序过程中无法利用索引，会导致全表遍历。
2. 大多数的结果都浪费了。

在数据量比较少时可以使用这种方案。

解决方案：没有具体的顺序······

    从 1 到最大值之间随机选择

```sql
SELECT b1.*
FROM bugs AS b1 JOIN (
    SELECT CEIL(RAND() * (SELECT MAX(bug_id) FROM bugs)) AS rand_id
) AS b2 ON (b1.bug_id = b2.rand_id);
```

>这种方案假设主键的值是从 1 开始并且保持连续。

    选择下一个最大值

```sql
SELECT b1.*
FROM bugs AS b1 JOIN (
    SELECT CEIL(RAND() * (SELECT MAX(bug_id) FROM bugs)) AS rand_id
) AS b2
WHERE b1.bug_id >= b2.rand_id
ORDER BY b1.bug_id
LIMIT 1;
```

>解决了在 1 到最大值之间有缝隙的情况。

>This solves the problem of a random number that misses any key value（这句话中文版翻译有误，正确的大意应该是，这种方案解决了随机数可能不匹配任何一个主键的问题，举例来说就是，这种方案解决了 `rand_id` 不与任何 `bug_id` 匹配的问题），这也意味着在一个缝隙之后的那个值被选中概率会增大。

>当队列中的缝隙不大并且每个值要被等概率选择的重要性不高时，可以考虑使用这种方案。

    获取所有的键值，随机选择一个

这要依赖程序代码，但是避免了对全表的排序，并且选择每个键的概率相等。

需要注意的是，这种方案在数据量大时仍然有可能超出程序所能处理的内存极限，同时查询必须执行两次。

```sql
#1
SELECT bug_id FROM bugs;
#2
#程序代码随机选择一个 bug_id
#3
SELECT * FROM bugs WHERE bug_id = ?;
```

查询逻辑简单、数据量适中、需要处理非连续值时可以考虑这种方案。

    使用偏移量选择随机行

1. 计算总的数据行数。
2. 随机选择 0 到总行数之间的一个值。
3. 用这个值作为位移来获取随机行。

`MySQL`、`PostgreSQL`、`SQLite` 中

```sql
SELECT ROUND(RAND() * SELECT COUNT(*) FROM bugs));  #1
SELECT * FROM bugs LIMIT 1 OFFSET :offset;          #2
```

`Oracle`、`Microsoft SQL Server`、`IBM DB2` 中使用 `ROW_NUMBER()` 函数，下面是在 `Oracle` 中的用法

```sql
#1
SELECT 1 + MOD(ABS(dbms_random.random()), (SELECT COUNT(*) FROM bugs)) AS offset FROM dual;
#2
WITH numbered_bugs AS (
    SELECT b.*, ROW_NUMBER() OVER (ORDER BY bug_id) AS RN FROM Bugs b
) SELECT * FROM numbered_bugs WHERE RN = :offset;
```

当主键可能不连续，又需要每行有相同的选中概率时，选择这个方案。

    专有解决方案

比如 `Microsoft SQL Server 2005` 的 `TABLE-SAMPLE` 子句

```sql
SELECT * FROM bugs TABLESAMPLE (1 ROWS);
```

`Oracle` 也有类似的 `SAMPLE` 子句

```sql
SELECT * FROM (SELECT * FROM bugs SAMPLE (1) ORDER BY dbms_random.value) WHERE ROWNUM = 1;
```

## 第 17 章 可怜人的搜索引擎

目标：全文搜索

反模式：模式匹配语言

    LIKE 断言、正则表达式

无法使用索引，存在性能问题；会返回意料之外的结果。

解决方案：使用正确的工具

    数据库扩展、第三方搜索引擎（Sphinx Search、Apache Lucene、Solr）、实现自己的搜索引擎

首先，定义一张 `keywords` 表来记录所有用户用来搜索的关键字。

```sql
CREATE TABLE keywords(
    keyword_id SERIAL PRIMARY KEY,
    keyword VARCHAR(40) NOT NULL,
    UNIQUE KEY (keyword)
);
```

接下来，将每个关键字和其所匹配到的 bug 添加到 `bugs_keywords` 表中。

```sql
CREATE TABLE bugs_keywords(
    keyword_id BIGINT UNSIGNED NOT NULL,
    bug_id BIGINT UNSIGNED NOT NULL,
    PRIMARY KEY (keyword_id, bug_id),
    FOREIGN KEY (keyword_id) REFERENCES Keywords(keyword_id),
    FOREIGN KEY (bug_id) REFERENCES Bugs(bug_id)
);
```

接下来，写一个存储过程来简化对一个给定关键字的搜索。

```sql
CREATE PROCEDURE bugs_search(keyword VARCHAR(40))
BEGIN
    SET @keyword = keyword;

    #1
    PREPARE s1 FROM 'SELECT MAX(keyword_id) INTO @k FROM keywords WHERE keyword = ?';
    EXECUTE s1 USING @keyword;
    DEALLOCATE PREPARE s1;
    
    IF (@k IS NULL) THEN
        #2
        PREPARE s2 FROM 'INSERT INTO keywords (keyword) VALUES (?)';
        EXECUTE s2 USING @keyword;
        DEALLOCATE PREPARE s2;
        #3
        SELECT LAST_INSERT_ID() INTO @k;
        
        #4
        PREPARE s3 FROM 'INSERT INTO bugs_keywords(bug_id, keyword_id)
            SELECT bug_id, ? FROM bugs
            WHERE summary REGEXP CONCAT( '' [[:<:]] '' , ?, '' [[:>:]] '' )
                OR description REGEXP CONCAT( '' [[:<:]] '' , ?, '' [[:>]] '' )';
        EXECUTE s3 USING @k, @keyword, @keyword;
        DEALLOCATE PREPARE s3;
    END IF;
    
    #5
    PREPARE s4 FROM 'SELECT b.* FROM bugs b JOIN bugs_keywords k USING (bug_id) WHERE k.keyword_id = ?';
    EXECUTE s4 USING @k;
    DEALLOCATE PREPARE s4;
END
```

`#1` 搜索用户指定关键字。

`#2` 如果未找到该词，将它做新词插入。

`#3` 查询 `keywords` 生成的主键值。

`#4` 通过搜索有新关键字的 `bug` 来填入交叉表。

`#5` 最后，查询符合 `keyword_id` 的整行。

在有新文档添加到数据库中时，需要定义一个触发器以填充交叉表。

```sql
CREATE TRIGGER bugs_insert AFTER INSERT ON bugs
FOR EACH ROW
BEGIN
    INSERT INTO bugs_keywords(bug_id, keyword_id)
        SELECT NEW.bug_id, k.keyword_id FROM keywords k
        WHERE NEW.description REGEXP CONCAT('[[:<:]]', k.keyword, '[[:>:]]')
            OR NEW.summary REGEXP CONCAT('[[:<:]]', k.keyword, '[[:>:]]');
END
```

如果要支持对 bug 描述的更新操作，还要写一个触发器去重新分析文本，然后添加或删除 `bugs_keywords` 表中的记录。

## 第 18 章 意大利面条式查询

总结起来就是尽量写简单的查询语句，将复杂的查询需求拆分成多个简单的查询，分而治之。

使用*代码生成*技术生成相似的 SQL 语句。

## 第 19 章 隐式的列

简单地说就是在查询时需要明确列出列命，少用或不用 `*`。

## 第 20 章 明文密码

不要使用明文存储密码，应该“先哈希，后存储”，还可以加盐。`SHA1`、`MD5` 已被认为是不安全的了，应该使用更高级的 `SHA256` 等哈希算法。

将*身份认证*和*验证*区分开来，身份认证是说你是谁，验证时说你确实是你说的谁。所以通常是在程序代码中通过用户名查询出用户，即身份认证；然后再程序代码中校验密码，即验证。

用户登录时一般通过网路传输的是明文的用户名和密码，这也可能不安全，会被劫持。一种做法是在客户端完成密码的哈希，但又要提前获取盐，获取盐又可能是不安全的。另一种是使用安全连接（`https`）。

在重置密码时不要将密码直接发给用户，而是将一个临时密码发给用户，同时这个临时密码有一个很短的过期时间。另外一种方案是，在数据库记录下重置密码请求，并为它分配一个唯一的令牌作为标识，将这个令牌发给用户，同时令牌有一个很短的过期时间，重置密码请求需要携带该令牌。

## 第 21 章 `SQL` 注入

目标：编写 `SQL` 动态查询

反模式：将未经验证的输入作为代码执行

治愈良方

1. 转义（部分有效）
2. 查询参数（非常有效）
    1. 也就是使用“参数占位符”
    2. 因为查询参数总是被视为一个字面值，所以它有如下弊端
        1. 多个值的列表不可以当成单一参数
        2. 表名无法作为参数
        3. 列命无法作为参数
        4. `SQL` 关键字不能作为参数
3. 存储过程（也有安全隐患）
4. 数据访问框架（无法解决）
5. 良好的规范（不太有效）

解决方案：不信任任何人

    过滤输入内容、参数化动态内容、给动态输入的值加引号、将用户与代码隔离、找个可靠的人来帮你审查代码

“将用户与代码隔离”这个不是很好理解，原文为“Isolate User Input from Code”，翻译大意为“将用户输入从程序代码中隔离出来”，结合下文的“将请求的参数作为索引值去查找预先定义好的值，然后用预先定义好的值来组织 `SQL` 查询语句”，这样看来这个方案是说不要相信用户的输入，应该将用户的输入映射为我们预定义的值，预定义的值可以提前保证它不会导致 `SQL` 注入。这种方案的优势是

1. 从不将用户的输入与 `SQL` 查询语句连接，因此减少了 `SQL` 注入的风险。
2. 可以让 `SQL` 语句中的任意部分变得动态化，包括标识、 `SQL` 关键字，甚至整句表达式。
3. 使用这个方法验证用户得输入变得很简单且高效。
4. 能将数据库查询的细节和用户界面解耦。（比如用户可以输入 `up`，真正查询时用 `ASC`）

## 第 22 章 伪键洁癖

直白的说 `id` 字段可能存在断档或缝隙，也就是不连续，不要尝试使它连续，不连续导致心理上的不适只是一种心理作用，需要客服这种心理作用。早点下班不好吗？

这章提供的一个有用知识点是，如何找到第一个未被使用的 `id`。

```sql
SELECT b1.bug_id + 1
FROM bugs b1 LEFT OUTER JOIN bugs AS b2 ON (b1.bug_id + 1 = b2.bug_id)
WHERE b2.bug_id IS NULL
ORDER BY b1.bug_id
LIMIT 1;
```

另一个知识点是，行号和 `id` 是两回事，`SQL:2003` 定义了包括 `ROW_NUMBER()` 在内的一些*窗口函数*，可以用来作为行号。

## 第 23 章 非礼勿视

不要蛮干，不要忽略程序返回的错误信息和异常信息，并尝试重错误或异常中恢复。

## 第 24 章 外交豁免权

目标：采用最佳实践

反模式：将 `SQL` 视为二等公民

解决方案：建立一个质量至上的文化

*质量保证*流程

1. 清晰地定义项目需求，并且写成文档。
2. 设计并实现一个解决方案来满足需求。
3. 验证并测试解决方案满足需求。、

与数据库相关配置、`SQL` 语句也应该纳入公司的最佳实践中，而不仅仅是代码。

## 第 25 章 魔豆

目标：简化 MVC 的模型

反模式：模型仅仅是活动记录

解决方案：模型包含活动记录

不要使用“贫血模型”，要使用“领域模型”。现在只能理解到包含了属性和业务方法的模型就是领域模型或充血模型，这也是 Java 中类本来的样子。

另一个知识点是，*抽象泄露*（Leaky Abstractions）。

>抽象简化了一些技术的内部工作原理并且让其更加方便调用。但当由于需要更高效的生产效率而不得不了解内部细节的时候，就称之为抽象泄露。

[面向对象编程的弊端是什么？ - 大宽宽的回答 - 知乎](https://www.zhihu.com/question/20275578/answer/643911658)中提到

>在大量使用继承作为设计方法时，也没有起到任何实质的隔离作用。如果你尝试扩展一个继承体系，往往需要了解整个继承体系才能写对代码——这时，复杂性并没有被隐藏起来。你也许只是代码写的少了而已。对于这种复杂度没有降低，编写代码只是写的少，但是要看懂还是得结合整个体系才能做到的方式，不是抽象，是“压缩”。压缩只能少写代码，却会让系统更难以理解了。

和抽象泄露是一个概念。我们的订单项目中，有大量的使用继承来完成抽象的类，写代码时是爽了，为了写的爽又不得不将整个继承体系看懂。

抽象是在进行信息压缩，为了理解抽象后的结果又需要解压信息。

END