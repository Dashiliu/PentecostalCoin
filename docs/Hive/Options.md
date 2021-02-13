# 常用操作

## UDF

Hive进行UDF开发十分简单，此处所说UDF为Temporary的function，所以需要hive版本在0.4.0以上才可以。

1.定义：UDF(User-Defined-Function)，用户自定义函数对数据进行处理。

2.用法:

​	1、UDF函数可以直接应用于select语句，对查询结构做格式化处理后，再输出内容。
​	2、编写UDF函数的时候需要注意一下几点：
​		a）自定义UDF需要继承org.apache.hadoop.hive.ql.**UDF**。
​		b）需要实现**evaluate**函。
​		c）evaluate函数支持重载。

​	3、以下是两个数求和函数的UDF。evaluate函数代表两个整型数据相加，两个浮点型数据相加，可变长数据相加

​	Hive的UDF开发只需要重构UDF类的evaluate函数即可。例：



```java
package hive.connect;
import org.apache.hadoop.hive.ql.exec.UDF;

public final class Add extends UDF {
	public Integer evaluate(Integer a, Integer b) {
	    if (null == a || null == b) {
	          return null;
	    } 
	    return a + b;
	}
	
	public Double evaluate(Double a, Double b) {
	       if (a == null || b == null) return null;
	       return a + b;
	
	}
	
	public Integer evaluate(Integer... a) {
	       int total = 0;
	       for (int i = 0; i < a.length; i++)
	           if (a[i] != null) total += a[i];
	       return total;
	}
}
```



​	4、步骤

​	a）把程序打包放到目标机器上去； 

​	b）进入hive客户端，添加jar包：hive>add jar /run/jar/udf_test.jar; 

​	c）创建临时函数：hive>CREATE TEMPORARY FUNCTION add_example AS 'hive.udf.Add'; 

​	d）查询HQL语句： 

​		SELECT add_example(8, 9) FROM scores; 

​		SELECT add_example(scores.math, scores.art) FROM scores; 

​		SELECT add_example(6, 7, 8, 6.8) FROM scores; 

​	e）销毁临时函数：hive> DROP TEMPORARY FUNCTION add_example; 

​	5、细节在使用UDF的时候，会自动进行类型转换，例如： 

​	SELECT add_example(8,9.1) FROM scores; 

注： 

1. UDF只能实现一进一出的操作，如果需要实现多进一出，则需要实现UDAF 

 

## UDAF

1、Hive查询数据时，有些聚类函数在HQL没有自带，需要用户自定义实现。 

2、用户自定义聚合函数: Sum, Average…… n – 1 

UDAF（User- Defined Aggregation Funcation） 

1.用法 

​	1、一下两个包是必须的import org.apache.hadoop.hive.ql.exec.**UDAF**和 	org.apache.hadoop.hive.ql.exec.**UDAFEvaluator**。 

​	2、函数类需要继承UDAF类，内部类Evaluator实UDAFEvaluator接口。 

​	3、**Evaluator需要实现 init、iterate、terminatePartial、merge、terminate这几个函数**。 

​		a）init函数实现接口UDAFEvaluator的init函数。 

​		b）iterate接收传入的参数，并进行内部的轮转。其返回类型为boolean。 

​		c）terminatePartial无参数，其为iterate函数轮转结束后，返回轮转数据，terminatePartial类似于hadoop的Combiner。 

​		d）merge接收terminatePartial的返回结果，进行数据merge操作，其返回类型为boolean。 
​		e）terminate返回最终的聚集函数结果。

```java
package hive.udaf;
import org.apache.hadoop.hive.ql.exec.UDAF;
import org.apache.hadoop.hive.ql.exec.UDAFEvaluator;

public class Avg extends UDAF {
	public static class AvgState {
         private long mCount;
         private double mSum;
	}

	public static class AvgEvaluator implements UDAFEvaluator {
    	AvgState state;
    	public AvgEvaluator() {
            super();
            state = new AvgState();
            init();
	}

	/** * init函数类似于构造函数，用于UDAF的初始化 */
	public void init() {
	         state.mSum = 0;
	         state.mCount = 0;
	}

	/** * iterate接收传入的参数，并进行内部的轮转。其返回类型为boolean * * @param o * @return */
	public boolean iterate(Double o) {
	         if (o != null) {
	              state.mSum += o;
	              state.mCount++;
	         } return true;
	}

	/** * terminatePartial无参数，其为iterate函数轮转结束后，返回轮转数据， * terminatePartial类似于hadoop的Combiner * * @return */
	public AvgState terminatePartial() {
	         // combiner
	         return state.mCount == 0 ? null : state;
	}
	
	/** * merge接收terminatePartial的返回结果，进行数据merge操作，其返回类型为boolean * * @param o * 	@return */
	public boolean terminatePartial(Double o) {                
	         if (o != null) {
	            state.mCount += o.mCount;
	            state.mSum += o.mSum;
	         }
	         return true;
	}
	/** * terminate返回最终的聚集函数结果 * * @return */
	public Double terminate() {
	         return state.mCount == 0 ? null : Double.valueOf(state.mSum / state.mCount);
	}
}
```




​	4、执行求平均数函数的步骤 
​		a）将java文件编译成Avg_test.jar。 
​		b）进入hive客户端添加jar包： 
​		hive>add jar /run/jar/Avg_test.jar。 
​		c）创建临时函数： 
​		hive>create temporary function avg_test 'hive.udaf.Avg'; 
​		d）查询语句： 
​		hive>select avg_test(scores.math) from scores; 
​		e）销毁临时函数： 
​		hive>drop temporary function avg_test;



## 总结

1、重载evaluate函数。
2、UDF函数中参数类型可以为Writable，也可为java中的基本数据对象。
3、UDF支持变长的参数。
4、Hive支持隐式类型转换。
5、客户端退出时，创建的临时函数自动销毁。
6、evaluate函数必须要返回类型值，空的话返回null，不能为void类型。
7、UDF是基于单条记录的列进行的计算操作，而UDFA则是用户自定义的聚类函数，是基于表的所有记录进行的计算操作。
8、UDF和UDAF都可以重载。
9、查看函数 SHOW FUNCTIONS;

## UDTF

1.定义 : UDTF(User-Defined Table-Generating Functions) 用来解决 输入一行输出多行(On-to-many maping) 的需求。

2.编写自己需要的UDTF
 (1) 继承org.apache.hadoop.hive.ql.udf.generic.GenericUDTF。

 (2)实现**initialize, process, close三个方法**。

UDTF首先会调用initialize方法，此方法返回UDTF的返回行的信息（返回个数，类型）。初始化完成后，会调用process方法，对传入的参数进行处理，可以通过forword()方法把结果返回。最后close()方法调用，对需要清理的方法进行清理。

下面是我写的一个用来切分”key:value;key:value;”这种字符串，返回结果为key, value两个字段。供参考：

```java
import java.util.ArrayList;
import org.apache.hadoop.hive.ql.udf.generic.GenericUDTF;
import org.apache.hadoop.hive.ql.exec.UDFArgumentException;
import org.apache.hadoop.hive.ql.exec.UDFArgumentLengthException;
import org.apache.hadoop.hive.ql.metadata.HiveException;
import org.apache.hadoop.hive.serde2.objectinspector.ObjectInspector;
import org.apache.hadoop.hive.serde2.objectinspector.ObjectInspectorFactory;
import org.apache.hadoop.hive.serde2.objectinspector.StructObjectInspector;
import org.apache.hadoop.hive.serde2.objectinspector.primitive.PrimitiveObjectInspectorFactory;

   public class ExplodeMap extends GenericUDTF{
       @Override
       public void close() throws HiveException {
           // TODO Auto-generated method stub    
       }
       @Override
       public StructObjectInspector initialize(ObjectInspector[] args)
               throws UDFArgumentException {
           if (args.length != 1) {
               throw new UDFArgumentLengthException("ExplodeMap takes only one argument");
           }
           if (args[0].getCategory() != ObjectInspector.Category.PRIMITIVE) {
               throw new UDFArgumentException("ExplodeMap takes string as a parameter");
           }
           ArrayList<String> fieldNames = new ArrayList<String>();
           ArrayList<ObjectInspector> fieldOIs = new ArrayList<ObjectInspector>();
           fieldNames.add("col1");
           fieldOIs.add(PrimitiveObjectInspectorFactory.javaStringObjectInspector);
           fieldNames.add("col2");
           fieldOIs.add(PrimitiveObjectInspectorFactory.javaStringObjectInspector);
           return ObjectInspectorFactory.getStandardStructObjectInspector(fieldNames,fieldOIs);
       }
      @Override
       public void process(Object[] args) throws HiveException {
           String input = args[0].toString();
           String[] test = input.split(";");
           for(int i=0; i<test.length; i++) {
               try {
                   String[] result = test[i].split(":");
                   forward(result);
               } catch (Exception e) {
                  continue;
              }
         }
       }
   }
```

3. 使用方法 : UDTF有两种使用方法，一种直接放到select后面，一种和lateral view一起使用。

​		1：直接select中使用

```
select explode_map(properties) as (col1,col2) from src;
```

不可以添加其他字段使用

```
select a, explode_map(properties) as (col1,col2) from src
```

不可以嵌套调用

```
select explode_map(explode_map(properties)) from src
```

不可以和group by/cluster by/distribute by/sort by一起使用

```
select explode_map(properties) as (col1,col2) from src group by col1, col2
```


​		2：和lateral view一起使用

```
select src.id, mytable.col1, mytable.col2 from src lateral view explode_map(properties) mytable as col1, col2;
```

此方法更为方便日常使用。执行过程相当于单独执行了两次抽取，然后union到一个表里。