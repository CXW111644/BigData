### spark概念

：是一种基于内存计算的大数据并行计算框架，也是一种快速、通用、可扩展的大数据分析引擎*

RDD： RDD（Resilient Distributed Dataset）叫做弹性分布式数据集，是Spark中最基本的数据抽象，它代表一个不可变、可分区、里面的元素可并行计算的集合。RDD是spark core的底层核心。

RDD是抽象概念，partition是具体概念

SparkRDD五大特性

- 每个RDD都是多个partition组成
- 可以对每个partition应用一个方法进行计算
- RDD是由上一个RDD转化来的，有依赖的
- 可以通过重新分区分配指定个数的partition来达到指定并行度的目的
  （有三个partition按理来说会有三个并行度去计算，但是我们可以把这三个partition指定成四个。用四个并行度去计算，增加计算效率。）
- 遵循数据本地行，数据在哪就在那加载到那台机器所在的内存
  （每个数据有三块，在计算时就把这三个位置封装到RDD中作为最佳计算位置）

### spark运行时

- spark自带的资源调度框架，主节点是master，从节点是worker。
- 如果是yarn，主节点是rm从节点是nm。

### spark on yarn模式

#### yarn的框架

resourcemanger是nodemanager的管理。负责资源调度，基本单位是container（封装有机器资源（内存，cpu，磁盘，网络）等），每一个分配到任务的节点都会被分配至少一个container，任务只能在cintainer里执行，并使用其中的资源，nodemanager负责启动所需要的container并且监控资源情况，上报给RM。applicationMaster与具体的application相关，负责同RM协商获取更合适的container，并且监控状态及进度。

- Yarn client

​    Yarn Client Dirver，spark的上下文向RM申请启动appmaster初始化spark context，RM分发给某一NM，拉起appMaster，appmaster向RM申请（container）资源，申请到之后，可以在申请到的container，启动excutor，可以向sparkcontext注册并申请task，sparkcontext分配task并监控excutor的各任务运行状态，excutor完成后向container汇报完成，全部成功后container申请注销并关闭自己。

<img src="C:\Users\陈宣玮\AppData\Roaming\Typora\typora-user-images\image-20250414145518310.png" alt="image-20250414145518310" style="zoom:67%;" />

- spark cluster

​    相比起yarnclient，sparkcluster区别在于Dirver部分在集群的节点中

<img src="C:\Users\陈宣玮\AppData\Roaming\Typora\typora-user-images\image-20250414145533830.png" alt="image-20250414145533830" style="zoom:67%;" />

#### 过程：

- 不管是哪种调度框架，每一个应用程序都会有一个驱动程序driver
- driver会把应用程序划分出多个task。
- 然后把这些task扔到集群的从节点（nm）去执行。
- 这些nm会把结果返回给driver

运行时调度

分布式文件系统（FileSystem）        --加载数据集

transformations延迟执行            --针对RDD的操作

action触发执行                     --碰到一个action就提交一个job（job切割）

![image-20250320225909318](C:\Users\陈宣玮\AppData\Roaming\Typora\typora-user-images\image-20250320225909318.png)

持久化策略--缓存优化

持久化策略：StorageLevel.MEMORY_ONLY

持久化到内存的数据又1t内存只有半t那么久之持久化半t然后剩下的需要用到这1t的时候缓存的半t直接拿来用，没缓存的继续找上一步的rdd，如果上一步也没有继续网上找直到找到数据源

StorageLevel.MEMORY_AND_DISK

内存装不下的那部分数据放到磁盘。
这种方式其实并不快。虽然不用重新计算了。但是结果要写磁盘，这很慢。用的时候，还要读磁盘。那我还不如直接去hdfs读原始数据，因为计算相对于写磁盘，还是很快的

StorageLevel.MEMORYAND_SER

将RDD系列化为Java对象，再进行持久化。这样比较节省内存，但是需要更多的CPU资源。因为序列化和反序列化占用 CPU,内存紧张的时候可以使用

yarn集群模式

yarn集群模式-client模式

这种模式下，driver会在提交命令的这台机器运行，所以日志和结果可以直接在当前控制台查看

yarn集群模式-cluster模式

这种模式下，会在集群中找一个nm来选行driver，其他nm计算的结果都交给运行driver的nm节点。，因为driver运行的节点不是自己指定的，所以结果只能在系统控制页面查看

常用算子介绍

#### Transformations(转换)

1. map()对RDD中的每个元素都执行一个指定的函数来产生一个新的RDD，任何原RDD中的元素在新RDD中都有且只有一个元素与之对应

2. filter()过滤返回满足条件的集合

3. flatMap()一变多

4. sample()随机字符抽样

   ```java
           JavaRDD<Integer> source = jsc.parallelize(Arrays.asList(1,2,3,4,5));
           JavaRDD<Integer> sample = source.sample(true,0.5);
           sample.collect().forEach(ele -> System.out.println(ele));
   ```

5. groupByKey()根据key进行聚合

6. reduceByKey()根据key进行聚合

7. union()并集合，将

   ```java
           JavaRDD<Integer> source1 = jsc.parallelize(Arrays.asList(1,2,3,4,5));
           JavaRDD<Integer> source2 = jsc.parallelize(Arrays.asList(1,2,3,4,5));
           JavaRDD<Integer> source=source1.union(source2);
           source.collect().forEach(System.out::println);
   ```

8. join()联合

   ```java
           JavaPairRDD<String,Tuple2<Integer,Integer>> joinRdd =wordOne1.join(wordOne2);
           joinRdd.collect().forEach(ele->System.out.println(ele._1+"---"+ele._2._1+"---"+ele._2._2));
   ```

   ```java
   rdd1 = sc.parallelize([("a", 1), ("b", 2), ("c", 3)])
   rdd2 = sc.parallelize([("a", 4), ("b", 5), ("d", 6)])
   result = rdd1.cogroup(rdd2)
   result.collect()
   # 输出: [('a', ([1], [4])), ('b', ([2], [5])), ('c', ([3], [])), ('d', ([], [6]))]
   
   ```

9. cogroup()类似于多表联合查询

   ```java
   rdd1 = sc.parallelize([("a", 1), ("b", 2), ("c", 3)])
   rdd2 = sc.parallelize([("a", 4), ("b", 5), ("d", 6)])
   result = rdd1.cogroup(rdd2)
   result.collect()
   # 输出: [('a', ([1], [4])), ('b', ([2], [5])), ('c', ([3], [])), ('d', ([], [6]))]
   
   ```

11. mapValues()对数组里的所有value进行操作

12. sort()指定序列排序

13. partitionBy()分区操作

#### Actions（）

1. count（）计算

2. collect（）收集

3. reduce（）通过func函数聚集RDD中的所有元素，先聚合分区内数据，再聚合分区间数据。

   ```java
   输入
   1 2 3 4 5
   6 7 8 9 10
   ("a",1),("b",3),("c",3),("d",5)
   输出
   55
   ("abcd",12)
   ```

4. lookup（）用于(K,V)类型的RDD,指定K值，返回RDD中该K对应的所有V值。


### checkpoint

宽依赖与窄依赖对于一些计算代价比较大的rd，我们可以

persist 缓存，内存空间足够可以使用这个缓存

checkpoint 到分布式文件系统（如 hdfs）中，内存空间不足时可以用，或者计算本身就很大的

#### 宽依赖与窄依赖

父RDD中一个partition的数据只流向子RDD的一个partition，就是窄依赖

父RDD中一个partition的数据会流向子RDD的多个partition，就是宽依赖

（只要有shuffle就是宽依赖）

DAG-有向无环图

一个application（main函数）可以有多个job（action算子），每个job可以有多个stage。
只要碰到宽依赖就切分新的stage，窄依赖不切分（DAG优化）
室依赖不用切分stage，也不需要存储读取中间结果，提升效率。

![image-20250320231146698](C:\Users\陈宣玮\AppData\Roaming\Typora\typora-user-images\image-20250320231146698.png)

计算到哪一部分有问题只需要计算那一部分的数据就行，避免大量重复计算

从上图可以知道，在一个pipeline中，数据在哪，计算就在哪，避免了数据传输的过程。比如在MR中，数据传递到下一步就是先写到磁盘，下一步再从磁盘读取到内存，这就效率很低。
而且只要不是action算子，都是延迟执行，这样就能从宏观上规划好task的封装，保证在一个task中，数据在哪，计算在哪，并且下一步直接在内存流程化执行，没有中间存储的资源和时间消耗



spark源码调度分析



SparkConf->def this()当前类 = this(true)将true传入类->class JavaSparkContext(val sc: SparkContext) extends Closeable {}进行一系列初始化sparkconf

setMaster，setAppName
->set("spark.master", master)->settings.put(key, value)-放到setting map里 

new JavaSparkContext(conf)
->def this(conf: SparkConf) = this(new SparkContext(conf))传给后边然后又用conf创建一个sparkcontext
->class JavaSparkContext(val sc: SparkContext) extends Closeable ,java代码封装一下方便java使用

SparkContext(conf)->class SparkContext(config: SparkConf) extends Logging

SparkEnv.set(_env)



只要不是def方法里面的代码都会被执行，只需要看try里的初始化部分即可

读取spark-master，appname

设置spark_env

读取配置文件内容，

启动spark.ul

ctreaetaskscheduler
不同模式下申请资源不同，所有不同的schedulerbackend（申请资源的）和taskschedule，创建dagscheduler不管什么模式都是一样的dag，   



action算子执行job过程



cach(缓存)



collect()

->runjob

->……

->dagScheduler.runJob(rdd, cleanedFunc, partitions, callSite, resultHandler, localProperties.get)（规划器）

->runjob()

->val waiter = submitJob(rdd, func, partitions, callSite, resultHandler, properties)(提交一个job，运行什么提交什么）

->submitJob

->eventProcessLoop(消息队列：采用异步事件驱动机制，确保了调度系统的高并发处理能力和模块解耦性。)

->private[spark] val eventProcessLoop = new DAGSchedulerEventProcessLoop(this)



### 共享变量

#### TopN

```java
package spark.p4;
import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaPairRDD;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;
import org.apache.spark.api.java.function.Function;
import org.apache.spark.api.java.function.Function2;
import org.apache.spark.api.java.function.PairFunction;
import org.apache.tools.ant.util.JavaEnvUtils;
import scala.Tuple2;
import java.util.List;
public class TopN {
    public static void main(String[] args) {
        int n=3;
        SparkConf sparkconf = new SparkConf().setMaster("local[2]").setAppName("TopN");
        JavaSparkContext jsc = new JavaSparkContext(sparkconf);
        JavaRDD<String> lineRdd = jsc.textFile("E:\\xiangmu\\Hadoopc01\\file\\input\\TopN\\in\\log.txt");

        //数据多可以加并行
//        lineRdd.repartition(10);

        JavaPairRDD<String,Integer> pairRdd = lineRdd.mapToPair((PairFunction<String, String, Integer>) line->new Tuple2<>(line,1));
        //将每一行数据转化为                                                     输入类型  输出的k   输出的v         将每个url后都跟一个1
        JavaPairRDD<String,Integer> urlNumRdd =pairRdd.reduceByKey((Function2<Integer, Integer, Integer>) (v1,v2) ->v1+v2);
        //                          汇总过的rdd           reduce计算根据k进行汇总，相同url的聚一块，v相加
        JavaPairRDD<Integer,String> numUrlRdd = urlNumRdd.mapToPair((PairFunction<Tuple2<String, Integer>, Integer, String>) tuple -> new Tuple2<>(tuple._2, tuple._1));
        //将格式url 2 改成2 url  替换（原因不能对value排序，只能对k排序）
        List<String> topN =numUrlRdd.sortByKey(false).map((Function<Tuple2<Integer, String>,String>) tuple -> tuple._2+"--->"+tuple._1).take(n);
        //先降序排序                                              对所有数据规范格式（tuple._2+"--->"+tuple._1）                                        拿取n个
        topN.forEach(System.out::println);

        //优化,直接拿前n个然后遍历出来
//        List<Tuple2<Integer, String>> topN =numUrlRdd.sortByKey(false).take(n);
//        topN.forEach(ele->System.out.println(ele._2+"--->"+ele._1));

        jsc.close();
    }
}
```

#### 分组求tap

```java
package spark.p4;

import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaPairRDD;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;
import org.apache.spark.api.java.function.PairFunction;
import org.apache.spark.api.java.function.VoidFunction;
import scala.Tuple2;
import java.util.ArrayList;
import java.util.Iterator;
import java.util.List;

public class GroupTopN {
    public static void main(String[] args) {
        int n = 3;
        SparkConf sparkConf = new SparkConf().setMaster("local[2]").setAppName("TopN");
        JavaSparkContext jsc = new JavaSparkContext(sparkConf);

        JavaRDD<String> lineRdd = jsc.textFile("E:\\xiangmu\\Hadoopc01\\file\\input\\TopN\\in\\groupTopN.txt");

        JavaPairRDD<String, Integer> uidNumRdd = lineRdd.mapToPair((PairFunction<String, String, Integer>) line -> {
            String[] arr = line.split(",");
            return new Tuple2<String, Integer>(arr[1], Integer.parseInt(arr[2]));
        });//将每一行根据“，”切分 url-1,uid-C,7   uid-C            7
        JavaPairRDD<String, Iterable<Integer>> uidNumIterableRdd = uidNumRdd.groupByKey();
        //根据uid进行分组，排序相同arr[1]的arr[2]

        //当数组很大时这个以下方法可以提高速度
        //根据排序可以取前n个数据，不需要进行全排序
        JavaPairRDD<String, Iterable<Integer>> topUidNumIterableRdd = uidNumIterableRdd.mapToPair(
            (PairFunction<Tuple2<String, Iterable<Integer>>, String, Iterable<Integer>>) tuple -> {
                List<Integer> list = new ArrayList<>(n);
                Iterator<Integer> numIterator = tuple._2.iterator();
                while (numIterator.hasNext()) {
                    int num = numIterator.next();
                    int i = 0;
                    while (i < n) {
                        if (i >= list.size()) {
                            list.add(num);
                            break;
                        } else {
                            if (list.get(i) < num) {//
                                if (list.size() >= n) {
                                    int j = list.size() - 1;
                                    for (; j > i; j--) {
                                        list.set(j, list.get(j - 1));
                                    }
                                    list.set(i, num);
                                } else {
                                    list.add(i, num);
                                }
                                break;
                            }
                        }
                        i++;
                    }
                }
                return new Tuple2<>(tuple._1, list);
            }
        );
        topUidNumIterableRdd.foreach((VoidFunction<Tuple2<String, Iterable<Integer>>>) tuple -> System.out.println(tuple._1 + "--->" + tuple._2));
        jsc.close();
    }
}

```

#### 二次查询

```java
package spark.p4;


import lombok.Getter;
import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaPairRDD;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;
import org.apache.spark.api.java.function.Function;
import org.apache.spark.api.java.function.PairFunction;
import scala.Serializable;
import scala.Tuple2;
import scala.math.Ordered;



public class SecondSort {
    public static void main(String[] args) {
        SparkConf sparkConf =new SparkConf().setMaster("local").setAppName("TopN");
        JavaSparkContext  jsc = new JavaSparkContext(sparkConf);
        JavaRDD<String> lineRdd = jsc.textFile("E:\\xiangmu\\Hadoopc01\\file\\input\\TopN\\in\\sort.txt");
        JavaPairRDD<SecondSortKey,String> pairs =lineRdd.mapToPair(new PairFunction<String,SecondSortKey,String>(){
        @Override
        public Tuple2<SecondSortKey, String> call(String line) throws Exception{
            String[] arr =line.split(" ");
            SecondSortKey key = new SecondSortKey(Integer.valueOf(arr[0]), Integer.valueOf(arr[2]));
            return new Tuple2<SecondSortKey,String>(key,line);
        }
    });
    JavaPairRDD<SecondSortKey,String> sortedPairs =pairs.sortByKey(false);
        JavaRDD<String> results = sortedPairs.map(new Function<Tuple2<SecondSortKey,String>,String>(){
            @Override
            public String call(Tuple2<SecondSortKey,String> tuple) throws Exception {
                return tuple._2;
            }
        });
        results.collect().forEach(System.out::println);
        jsc.close();
    }
}
@Getter
class  SecondSortKey implements Serializable, Ordered<SecondSortKey>{

    private  int first;
    private int second;

    public SecondSortKey(int first,int second){
        super();
        this.first = first;
        this.second = second;
    }

    public void setFirst(int first){this.first =first;}

    public void setSecond(int second) { this.second =second;}

    @Override
    public int compare(SecondSortKey that) {
        if(this.first -that.getFirst()!=0) {
            return this.first - that.getFirst();
        }else{
            return this.second -that.getSecond();
        }
    }

    @Override
    public boolean $less(SecondSortKey that) {
        if(this.first<that.getFirst()){
            return true;
        }else if(this.first==that.getFirst()
                && this.second<that.getSecond()){
            return true;
        }
        return false;
    }

    @Override
    public boolean $greater(SecondSortKey that) {
        if(this.first> that.getFirst()){
            return true;
        }else if(this.first==that.getFirst()
                &&this.second>that.getSecond()) {
            return true;
        }
        return false;
    }

    @Override
    public boolean $less$eq(SecondSortKey that) {
        if(this.$less(that)) {
            return true;
        }else if(this.first==that.getFirst() && this.second==that.getSecond()){
                return true;
        }
        return false;
    }

    @Override
    public boolean $greater$eq(SecondSortKey that) {
        if(this.$greater(that)){
            return true;
        }else if(this.first==that.getFirst()
                && this.second==that.getSecond()){
            return true;
        }
        return false;
    }

    @Override
    public int compareTo(SecondSortKey that) {
        if(this.first-that.getFirst()!=0){
            return this.first -that.getFirst();
        }else{
        return this.second -that.getSecond();}

    }

    @Override
    public String toString(){
        return "SecondSortKey{"+
                "first="+first+",second="+second+"}";
    }
}
```

广播变量

在 Spark 中，如果你在一个集群的多个任务中使用同一个变量（如查找表、配置数据），默认情况下这个变量会被每个任务单独拷贝一份，这会浪费网络带宽和内存。

共享的是：只读的、需要被多个任务使用的变量值。

若变量大小超过executor后引起内存溢出

广播变量 = 把“小数据”广播给“大数据”的每个执行任务，节省网络 IO，提升性能。

```java
Broadcast<int[]>broadcastvar=sc.broadcast(newint[] {1,2,3}）；
broadcastvar.value（）；                                                                                    
```

累加器

累加器是 Spark 提供的一种在多个任务中做聚合统计的机制，适用于 driver 端想统计 task 执行中的一些信息（但 task 之间不需要通信）的场景。

根据实际机器自行调节需要的数

只能并行的增加。Task的call不能读取

```java
import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;
import org.apache.spark.util.LongAccumulator;
import java.util.Arrays;
public class Leijiaqi {
    public static void main(String[] args) {
        SparkConf sparkConf =new SparkConf().setMaster("local[3]").setAppName("leijiaqi");
        JavaSparkContext jsc =new JavaSparkContext(sparkConf);
        JavaRDD<Integer> numRdd=jsc.parallelize(Arrays.asList(1,2,3,4,6));
        LongAccumulator accum=jsc.sc().longAccumulator();
        numRdd.foreach(x->accum.add(x));
        Long value=accum.value();
        System.out.println(value);

    }
}
```



### sparkSQL

DataFrame是一个分布式数据容器，更像传统的二维表格（映射为一张表的格式）

#### 支持数据源

json，hive，jdbc，parquet，mysql，hdfs，csv，hbsae等

#### 执行优化

如两个表需要join和过滤，系统会先进行过率再join使得数据量小，计算方便

#### SQLContext与SparkSession

Sparksql提供了两种数据流转化为视图窗的方法

 	SQLContext，用于Spark自己提供的SQL查询

​	 HiveContext，用于连接Hive的查询。

在新的版本中，可以使用 SparkSesson来统一，内部封装有sparkcontext

#### DataSet与DataFrame

```java
package spark.p6;

import org.apache.spark.SparkConf;
import org.apache.spark.sql.*;

public class DatasetOperation{
    public static void main(String[] args) {
        SparkConf sparkConf=new SparkConf().setAppName("DataFrameCreate").setMaster("local[2]");

        SparkSession sqlContext=SparkSession.builder().config(sparkConf).getOrCreate();
         //SparkSession直接创建sqlcontext，不需要再分情况
        Dataset<Row> df=sqlContext.read().json("E:\\xiangmu\\Hadoopc01\\file111\\json\\student.json");
        //文件地址
//        df.show();//show就是打印

////        df.printSchema();//printschema打印表字段信息
//
////        df.select(df.col("name"),df.col("score").plus(1).as("score1")).show();
				//查询name字段，				成绩字段 +1  		重命名			打印
//
////        df.filter(df.col("score").gt(80)).show();//过滤
//									 gt大于lt小于e等于
////        df.groupBy("score").count().show();
//				分组字段分数      计数信息  
//        df.createOrReplaceTempView("stuu");//创建一个stuu的虚拟表，可以直接进行sql操作
//        Dataset<Row> df1=sqlContext.sql("select * from stuu where score>=90");//直接进行sql操作
////        df1.show();
//
//        df1.write()
//                .mode(SaveMode.Append)//(追加)
////                .mode(SaveMode.ErrorIfExists)(有文件就报错)
//                .format("parquet")(压缩i格式)
//                .format("json")
//                .save("E:\\xiangmu\\Hadoopc01\\file111\\out");
//
//        sqlContext.close();
    }
}
```



#### 自定义udf

```java
package spark.p6;

import org.apache.spark.SparkConf;
import org.apache.spark.sql.Dataset;
import org.apache.spark.sql.Row;
import org.apache.spark.sql.SparkSession;
import org.apache.spark.sql.api.java.UDF1;
import org.apache.spark.sql.expressions.UserDefinedFunction;
import org.apache.spark.sql.types.DataTypes;

import java.util.Random;

import static org.apache.spark.sql.functions.udf;

public class UDFAndUDAFTest {
    public static void main(String[] args) {
        SparkConf conf = new SparkConf().setAppName("DataFrameCreate").setMaster("local[2]");
        SparkSession sqlContext = SparkSession.builder().config(conf).getOrCreate();
        Dataset<Row> rowDataset = sqlContext.read().json("E:\\xiangmu\\Hadoopc01\\file111\\json\\student.json");
        rowDataset.createOrReplaceTempView("stuu");

        Random random = new Random();
        sqlContext.udf().register("addScore", new UDF1<Long, Long>() {
            //            注册    名为addscore的udf 
            @Override
            public Long call(Long score) throws Exception {
                if (score == null) return null;
                return score + random.nextInt(20);
                //将成绩加1-19的随机数
            }
        }, DataTypes.LongType);
        //Spark 在执行时需要知道返回数据的类型
        sqlContext.sql("SELECT  addScore(score) AS score2 FROM stuu").show();
        //上面注册过所有可以直接用addscore
        sqlContext.close();
    }
}

```



读写csv

```java
package spark.p6;
import org.apache.spark.SparkConf;
import org.apache.spark.api.java.function.MapFunction;
import org.apache.spark.sql.*;

public class InOutCsv {
    public static void main(String[] args) {
        SparkConf sparkConf=new SparkConf().setAppName("sparksql").setMaster("local[2]");
        SparkSession spark = SparkSession.builder().config(sparkConf).getOrCreate();
        DataFrameReader reader=spark.read();
        Dataset<Row> studentDS= reader
                .option("header","true")
                .option("sep",",")
                .csv("E:\\xiangmu\\Hadoopc01\\file111\\csv\\111.csv");
        Dataset<StudentCSV> studentDS1=studentDS.map(new MapFunction<Row, StudentCSV>() {
            @Override
            public StudentCSV call(Row value) throws Exception {
                    return new StudentCSV(value.getString(0),Long.valueOf(value.getString(1)));
            }
        },Encoders.bean(StudentCSV.class));
        studentDS1.show();
//        DataFrameWriter<StudentCSV> writer =studentDS1.write();
//        writer.option("sep",",")
//            .option("header","true")
//                .mode(SaveMode.Append)
//                .csv("E:\\xiangmu\\Hadoopc01\\file111\\out\\csv");
//        spark.stop();
    }

}

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;
@Data
@AllArgsConstructor
@NoArgsConstructor
public class StudentCSV {
    private String name;
    private long score;
}
```



parquet读写

```java
import org.apache.spark.SparkConf;
import org.apache.spark.sql.Dataset;
import org.apache.spark.sql.Row;
import org.apache.spark.sql.SparkSession;

public class InOutParquet {
    public static void main(String[] args) {
        SparkConf sparkConf=new SparkConf().setAppName("sparksql").setMaster("local[*]");
        SparkSession spark = SparkSession.builder().config(sparkConf).getOrCreate();

        //写入
        Dataset<Row> rowDataset = spark.read().json("E:\\xiangmu\\Hadoopc01\\file111\\parquet\\student.json");
        rowDataset.write().parquet("E:\\xiangmu\\Hadoopc01\\file111\\out\\parquet");
        //读入
        Dataset<Row> parquet =spark.read().parquet("E:\\xiangmu\\Hadoopc01\\file111\\out\\parquet");
        parquet.printSchema();//打印parquet文件
        parquet.show();//展示
        spark.close();
    }
}
```

mysql读取写入

```JAVA
package spark.p6;
import org.apache.spark.SparkConf;
import org.apache.spark.sql.Dataset;
import org.apache.spark.sql.Row;
import org.apache.spark.sql.SparkSession;
import java.util.Properties;
public class InOutMysql {
    public static void main(String[] args) {
        SparkConf sparkConf=new SparkConf().setAppName("sparksql").setMaster("local[2]");
        SparkSession sparkSession= SparkSession.builder().config(sparkConf).getOrCreate();
        Properties properties =new Properties();
        properties.setProperty("user","root");
        properties.setProperty("password","123456");

        //写入mysql
//        Dataset<Row> rowDataset = sparkSession.read().json("E:\\xiangmu\\Hadoopc01\\file111\\json\\student.json");
//        rowDataset
//        	 .write()
//           .mode(SaveMode.Append)
//           .jdbc("jdbc:mysql://localhost:3306/cxw?serverTimezone=GMT","student2",properties);
        
        //读取mysql数据
        Dataset<Row> mysqlData=sparkSession
            .read()
            .jdbc("jdbc:mysql://localhost:3306/cxw","student",properties);
        mysqlData.show();
        
        sparkSession.close();
    }
}
```

 
