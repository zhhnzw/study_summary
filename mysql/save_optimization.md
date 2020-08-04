## 存储优化

### 基本原则

* 更小的通常更好。占用更少的磁盘空间、内存和CPU缓存
* 简单就好。例如：整型比字符操作代价更低
* 尽量避免NULL。可为NULL的列会使用更多的存储空间

### 整型

TINYINT、SMALLINT、MEDIUMINT、INT、BIGINT

有可选的UNSIGNED属性，表示不允许负值，并且可使正值的上限提高一倍

注：INT(11)并不会限制值的合法范围，只是规定了MySQL的一些交互工具用来显示字符的个数。对于存储和计算来说，INT(1)和INT(20)是相同的。

### 实数

DECIMAL、FLOAT、DOUBLE
DECIMAL用于存储精确的小数。如DECIMAL(18, 9)含义是：18是字段值的总长度，9是小数位数的长度，即此时整数部分也是9位长度。

注：DECIMAL需要额外的空间和计算开销，所以应该尽量只在对小数进行精确计算时才使用DECIMAL，例如存储财务数据。但在数据量比较大的时候，可以考虑使用BIGINT代替DECIMAL，将需要存储的货币单位根据小数的位数乘以相应的倍数即可。这样可以同时避免浮点存储计算不精确和DECIMAL精确计算代价高的问题。

### 字符串

**VARCHAR、CHAR**

VARCHAR使用场景：字符串列的最大长度比平均长度大很多

CHAR使用场景：很短的字符串；或者所有值都接近同一个长度，例如密码的md5值；对于经常变更的数据，CHAR也比VARCHAR更好，因为定长的CHAR类型不容易产生碎片。

**TINYTEXT、SMALLTEXT、TEXT、MEDIUMTEXT、LONGTEXT**

**TINYBLOB、SMALLBLOB、BLOB、MEDIUMBLOB、LONGBLOB**

BLOB存储的是二进制数据，没有字符集或排序规则，TEXT类型有字符集和排序规则

**ENUM**

在不少场景中，使用枚举代替字符串是一种比较好的优化方法，不过需要注意一点，对于一系列未来可能会改变的字符串，使用枚举就不是一个好主意了（因为要用ALTER TABLE），除非能接受只在列表末尾添加元素

### 日期和时间

**TIMESTAMP、DATETIME**

* TIMESTAMP和DATETIME 精度都是秒

* DATETIME与时区无关，TIMESTAMP与时区有关

* TIMESTAMP使用更小的存储空间，时间范围也比DATETIME小

注：除特殊情况之外，应该尽量使用TIMESTAMP，因为它比DATETIME空间效率高。如果需要存储比秒更小粒度的时间，可以使用BIGINT类型存储微妙级别的时间戳