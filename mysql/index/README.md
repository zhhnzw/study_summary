## 索引

索引（Index）是帮助MySQL高效获取数据的**数据结构**。

索引能够加快访问数据的速度，因为存储引擎不再需要进行全表扫描来获取需要的数据，取而代之的是从索引的根节点开始进行搜索。

#### 优势和劣势

* 优势：提高了数据检索的效率，降低数据库的IO成本；降低数据排序的成本，降低了CPU的消耗。
* 劣势：索引也要占用空间，会降低更新表的速度（更新表时也要更新索引）。

#### 索引的分类

* 单列索引
* 复合索引：该类索引包含多个列。
* 唯一索引：该类索引列的值必须唯一，但允许有空值。

#### 索引的数据结构类型

* B+Tree索引
* 哈希索引
* 空间数据索引
* 全文索引

#### 哪些情况不要建索引？

* 表记录太少
* 经常增删改的表（频繁增删改表数据时，MySQL底层还要额外更新索引文件，反而降低了效率）
* 数据重复率高且分布平均的字段（通过索引要能筛选掉绝大部分不需要的数据）
