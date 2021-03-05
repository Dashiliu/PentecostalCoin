# Hive中的数据采样 - 数据倾斜第一步

当数据集比较大时，可能需要通过采集一部分数据集进行分析，称之为采样。在HQL中支持三种方式的采样：随机采样(random sampling)、分桶表采样(bucket table sampling)以及块采样（block sampling）。

## 1.随机采样

随机采样使用rand()函数和limit关键字。其中distribute和sort关键字用来保证抽取的数据是随机分布的，这种方式比较有效率。**order by rand()也可以实现上述的功能，但是性能不好**。

例子：

SELECT 
    name 
FROM employee
**DISTRIBUTE BY rand() SORT BY rand()** 
**LIMIT 2;**

## 2.分桶表采样

这是一种特殊的抽样方法，适用于分桶表，并做了优化。SELECT语句指定要采样的具体列 ，当需要采样整行时，可以使用rand()函数。如果采样的列与CLUSTERED BY 列(即分桶列)相同，则采样的效率会更高。

注：tablesample是抽样语句，语法：TABLESAMPLE(BUCKET x OUT OF y) 。
y必须是table总bucket数的倍数或者因子。hive根据y的大小，决定抽样的比例。例如，table总共分了4桶，当y=2时，抽取(4/2=)2个bucket的数据，当y=8时，抽取(4/8=)1/2个bucket的数据。
x表示从哪个bucket开始抽取，如果需要取多个分桶数据，以后的抽取的分桶号为当前分桶号加上y。例如，table总bucket数为4，tablesample(bucket 1 out of 2)，表示总共抽取（4/2=）2个bucket的数据，抽取第1(x)个和第3(x+y)个bucket的数据。
注意：x的值必须小于等于y的值，

例子1：

SELECT
    name
FROM employee
TABLESAMPLE(BUCKET 1 OUT OF 2 ON rand()) a;
例子2：

-- 根据分桶列进行采样，效率更高
SELECT 
    name
FROM employee
**TABLESAMPLE(BUCKET 1 OUT OF 2 ON emp_id) a;**

## 3.块采样

是指随机的选取n行数据，n可以是数据大小的百分比，或者数据字节数。采样的粒度使HDFS块的大小。

例子1：

-- 按行数进行采样
SELECT
   name
FROM employee 
**TABLESAMPLE(1 ROWS) a;**
例子2：

--按数据量的百分比进行采样
SELECT
   name
FROM employee
**TABLESAMPLE(50 PERCENT) a;**
例子3:

-- 按照数据的字节数进行采样
-- 支持 b/B, k/K, m/M, g/G
SELECT 
    name 
FROM employee 
**TABLESAMPLE(1B) a;**