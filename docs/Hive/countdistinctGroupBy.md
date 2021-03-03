count(distinct) 与 group by 区别

很多情况下，尤其是对文本类型的字段，直接使用count distinct的查询效率非常低，而先做group by再count往往能提升查询效率。

但是，实验表明，对于不同的字段，count distinct与count  group by的性能并不一样，而且其效率与目标数据集的数据重复度相关。

举例：分别使用count distinct 和 count group by对 bigint, macaddr, text三种类型的字段做查询。


首先创建如下结构的表testmac

| Column | Type | Modifiers |
| mac_bigint | bigint |          |
| mac_macaddr | macaddr |          |
| mac_text | text |          |

并插入1000W条记录，并保证mac_bigint为mac_macaddr去掉冒号后的16进制转换而成的10进制bigint，而mac_text为mac_macaddr的文本形式，从而保证在这三个字段上查询的结果，也及复杂度相同。

count distinct SQL如下：

select
    count(distinct mac_macaddr)
from
    testmac
count group by SQL如下：

select
    count(*)
from
    (select
        mac_macaddr
    from
        testmac
    group by
        foo)
对于不同记录数较大的情景（1000万条记录中，有300多万条不同记录），查询时间（单位毫秒）如下表所示。

| query/字段类型 | macaddr   | bigint    | text       |
| -------------- | --------- | --------- | ---------- |
| count distinct | 24668.023 | 13890.051 | 149048.911 |
| count group by | 32152.808 | 25929.555 | 159212.700 |

对于不同记录数较小的情景（1000万条记录中，只有1万条不同记录），查询时间（单位毫秒）如下表所示。

| query/字段类型 | macaddr   | bigint   | text       |
| -------------- | --------- | -------- | ---------- |
| count distinct | 20006.681 | 9984.763 | 225208.133 |
| count group by | 2529.420  | 2554.720 | 3701.869   |

从上面两组实验可看出：

在不同记录数较小时，count group by性能普遍高于count distinct，尤其对于text类型表现的更明显。

而不同记录数较大时，count group by 的性能反而低于count distinct

count distinct 多个map 1个reduce

count  (group by 多个map 多个reduce) 1个reduce 数据量较大情况更合适

数据量重复较少 : 36864284 -> 35488015 , 时间相差不多

count(distinct) 1个reduce 在还查其他字段时,可能会出现数据倾斜问题