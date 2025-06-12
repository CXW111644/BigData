# hive

## hive与之前学习的关系

hive写hql（sql）查询采用hadoop的mr程序，然后数据真实存储到hdfs

## hive架构

![image-20250212160404998](C:\Users\陈宣玮\AppData\Roaming\Typora\typora-user-images\image-20250212160404998.png)

hive是基于hadoop的一个数据仓库工具

### hive的服务

#### hivserver2服务

hivserver2

提供jdbc/odbc接口，为用户远程访问Hive数据的功能，个人电脑访问远程服务中的hive数据，就用到hivserver2

在远程访问hive时客户端为直接访问集群，而是由hivserver2代理访问，若启用hivserver2会模拟客户端登录用户去

访问hadoop集群数据

#### metastore服务（独立服务模式）

hive的metastore服务的作用是为hive 客户端，或者hiveserver2提供元数据访问接口，metastore让元数据库减负，在metastore中可以进行缓存

嵌入模式：hive客户端或者hiveserver2直接访问元数据

### 数据类型

#### 基本数据类型

（mysql类似）

| hive      | 说明                     |
| --------- | ------------------------ |
| tinyint   | 1byte有符号整数          |
| smallint  | 2byte有符号整数          |
| int       | 4byte有符号整数          |
| bigint    | 8byte有符号整数          |
| boolean   | 布尔型                   |
| float     | 单精度浮点数             |
| double    | 双精度浮点数             |
| decimal   | 十进制精确数字类型       |
| varchar   | 字符序列需指定最大长度   |
| string    | 字符串，无需指定最大长度 |
| timestamp | 时间类型                 |
| xbinary   | 二进制数据               |

#### 复杂数据类型

| 类型   | 说明                                                     | 定义                       | 取值       |
| ------ | -------------------------------------------------------- | -------------------------- | ---------- |
| array  | 数组是一组相同类型的值的集合                             | array<string>              | arr[0]     |
| map    | map是一组相同类型的键-值对集合                           | map<string,int>            | map['key'] |
| struct | 结构体由多个属性组成，每个属性都有自己的属性名和数据类型 | struct<id:int,name:string> | struct.id  |

### 关键字说明：

1. TEMPORARY:临时表
2. EXTERNAL:内部表（管理表），管理表意味着hive会完全接管该表，包括元数据和HDFS中的数据；外部表，hive只接管元数据，而不完全接管hdfs中的数据
3. data type：数据类型：基本数据类型与复杂数据类型；可以显式转化或隐式转化
4. PARTITIONED BY(分区表)
5. CLUSTERED BY … SORTED BY … INTO … BUCKETS(分桶表)
6. ROW FORMAT：hive使用rializer and Deserializer（SERDE）序列化和反序列化：是数据处理中的两个基本概念，通常用于将数据结构或对象转换为可存储或传输的格式，或者从这种格式恢复原始数据结构或对象。
7. STORED AS：指定文件格式，常见文件格式textfile（文本格式），sequence file（），orc file,parquet file
8. LOCATION:指定表所对应的hdfs路径，不指定路径则为"user/hive/warehouse"(默认值)
9. TBLPROPERTIES：配置表的一些kv键值对参数

## 数据库

### 表

#### 内部表外部表

| **特性**               | **内部表**                                               | **外部表**                                             |
| ---------------------- | -------------------------------------------------------- | ------------------------------------------------------ |
| **数据管理**           | Hive 管理数据（数据存储在 Hive 默认目录）                | 数据由用户自己管理，存储路径由用户指定                 |
| **删除表时数据的处理** | 删除表时，数据和表结构都会被删除                         | 删除表时，只删除表结构，数据文件保留                   |
| **数据生命周期**       | 数据生命周期由 Hive 完全控制                             | 数据生命周期不由 Hive 控制（用户管理）                 |
| **使用场景**           | 用于完全由 Hive 管理的数据，适合数据完全属于 Hive 的场景 | 用于需要让 Hive 管理表结构，但数据不由 Hive 管理的场景 |

##### 查看表 



##### 创建表

- reateTable As Select（CTAS）建表 允许用户利用select出来的结果表作为新表

- Create Table Like 语法 创建一个新表，结构一样没有数据

- ```
  create external table student()
  row fromat delimited fields terminated by
  location ' '
  ```






##### 修改表

##### 清空表









#### join语句

- 等值join
- 表的别名
- 内连接
- 左外连接left join…on
- 右外连接right join…on
- 满外连接full join…on（将返回所有的符合条件的记录，没有符合条件的值用null替代）
- 多表连接join..on join..on
- 笛卡尔积省略所有连接条件，表中所有行相互连接；行数=两表行数之积
- 联合union（两个结构相同的表上下拼接）

#### 排序

- Order By “字段” asc（升序）desc（降序）
- 设置reduce数量内部排序（set mapreduce.job.reduces=-1;）
- 分区（设置多个reduces）（Distribute By）
- 分区排序（Cluster By“字段”=distribute by“字段”+sort by“字段”）
  先分区然后在reduce里排序

#### 函数

- 数值函数

- ```sql
  select round(3.3);  //四舍五入3
  select ceil(3.1);   //向上取整4
  select floor(4.8);  //向下取整4
  ```

- 字符串函数（截取字符串）

- ```sql
  select substring("abcdefgh",2);      bcdefgh
  select substring("abcdefgh",2,3);    bcd 
  ```

- 替换

  ```sql
  select replace('abcdefg','a','A');    Abcdefg
  ```

- 正则替换

  ```
  select regexp_replace('100-200','(\\d+)','num');将前边字段中符合中间正则表达式的替换为后边字段
  ```

- 正则匹配

  ```
  select 'deaaa' regexp 'de'前面字段匹配后边的正则表达式就返回true
  ```

- 重复字符串

  ```
  select repeat("123",3);       123123123
  ```

- 字符串切割

  ```
  select split('a-b-c-d','-');   ["a","b","c","d"]
  ```

- 替换null值

  ```
  select nvl(null,1);     将null替换为1
  ```

- 拼接字符串

  ```
  select concat（'a','b','c','d');    abcd 
  ```

- 指定分隔符拼接字符串

  ```
  select concat ws（'-','a','b','c','d');    a-b-c-d 
  ```

- 解析json具体数据

  ```sql
  select get_json_object('[{"name":"张三","sex":"男","age":"25"},{"name":"李四","sex":"男","age":"23"}]','$.[0].name'); "$"表示该数组
  ```

- …



#### 条件判断函数

```sql
select
	case
		when 条件 then '返回值'
		when 条件 then '返回值'
		else '返回值' end as 别名
from 表名
```

另一种形式色，

```sql
select
	case '字段'
		when 。。 then '返回值'
		when 。。 then '返回值'
		else '返回值' end as 别名
from 表名
```

if判断

三元运算符

```sql
select if（10<11,正确,错误）;
```

数组中的数量（size（））

```sql
select name,size(friends),friends from teacher;
```

map集合可以输出k，v

```sql
select `map`('xiaohai',1,'dahai',2);
select map_keys(`map`('xiaohai',1,'dahai',2));   ["xiaohai","dahai"]
select map_values(`map`('xiaohai',1,'dahai',2)); [1,2]
```

array_contains：查看array中是否包含某个元素返回true和false

```sql
select array_contains(array('log1','log2','log3','log4'),'log1')
```

sort_array:元素排序

```sql
select sort_array(`array`('a','d','c'));
```

named_struct声明属性

```sql
select named_struct('name','xiaoming','age',18,'weight',80);
```

#### 高级聚合

（多行传入一行输出）

collect_list收集并形成list集合，结果不去重

collect_set收集并形成list集合，结果去重

#### 炸裂函数

```hive
select
     movie,
     cate
 from (select
           movie,
           split(category,',') cates
       from movie_info
      )t1 lateral view explode(cates) tmp as cate
```



#### 窗口函数

partation by'字段'(用‘字段’来分窗口)



函数（）over（order by row| between。。and。。）

- [num] preceding:当前行的前num行
- current_row:当前行
- [num]following:当前行的后bnum行
- unbounded following：最后一行

以上所说的行数是mr计算时候的行数为了保证看到的与限定的行数是对应的前边要加个order by[column]

1. 聚合函数：max，min，sum，avg，count

2. 跨行取值函数 ：

   lead获取当前行下边的某行某字段值，

   lag获取当前行上边的某行某字段值

   first_value:获取窗口内某一列的第一个值

   last_value:获取窗口内某一列的最后一个值

3. 排名函数

   rank：并列时也占名次

   dense_rank:并列时不占名次

   row_number:并列时按照序号排序



```hive 
hive> add jar /home/had/myfiles/Hadoopc01-1.0-SNAPSHOT.jar;
Added [/home/had/myfiles/Hadoopc01-1.0-SNAPSHOT.jar] to class path
Added resources: [/home/had/myfiles/Hadoopc01-1.0-SNAPSHOT.jar]

创建临时函数
hive> create temporary function my_len as "com.bigdata.udf.MyLen";
OK
Time taken: 0.356 seconds
hive> select my_len("abc");
OK
3
Time taken: 0.847 seconds, Fetched: 1 row(s)
```

```hive
hive> select
    > ename,my_len(ename) ename_len
    > from emp;
OK
张三    2
李四    2
王五    2
赵六    2
陈七    2
刘八    2
孙九    2
周十    2
Time taken: 0.273 seconds, Fetched: 8 row
```

### 分区表与分桶表

#### 分区表

再添加数据时（partition (day='20220401');）

```hive

load data local inpath '/opt/module/hive/datas/dept_20220401.log'
into table dept_partiton
partition (day='20220401');

insert overwrite table dept_partiton partition (day='20220402')
select deptno,dname,loc
from dept_partiton where day='20220401';
```

创建两个分区表（表之间不能有符号）

```hive
alter table dept_partiton add partition (day='20220403') partition (day='20220404')
```

删除分区表（多个表之间需要空格）

```hive
alter table dept_partiton
    drop partition (day='20220404'),partition (day='20220403');
```

修复分区

分区信息都在元数据，如果改动名字或者修改，需要进行，同步hdfs路径和元数据分区信息

二级分区表

```hive
create table dept_partition2(
                                deptno int,
                                dname string,
                                loc string
)
    partitioned by (day string, hour string) -- 使用 '--' 进行单行注释
    row format delimited fields terminated by '\t';
    
    
 --查询   
select *
from dept_partition2
where day='20220401' and hour='12' ;
```

动态分区

```hive
--动态分区总开关
set hive.exec.dynamic.partition=true;
--严格模式与非严格模式
set hive.exec.dynamic.partition.mode=nonstrict;
--单个insert语句同时创建的最大分区个数
set hive.exec.max.dynamic.partitions=1000;
--单个Mapper或者Reducer可同时创建的最大的分区个
set hive.exec.max.dynamic.partitions.pernode=100;
--一条insert语句可以创建的最大的文件个
set hive.exec.max.created.files=100000;
--当查询结果为空时且进行动态分区时，是否抛出异常，默认false
set hive.error.on.empty.partition=false;[]()
```

#### 分桶表

为每一行计算一个指定字段的数据的hash值，然后模拟一个分桶数，最后取模运算结果相同的行，写入同一个文件中，这个文件就称为分桶；(通过id进行hash运算分成into  4 buckets四部分)

```hive
create table stu_buck_sort(
id int,
name string
    )
clustered by(id)sorted by(id)
into  4 buckets
row format delimited fields terminated by'\t';
```

### 文件格式和压缩

hadoop压缩：

hive中保持数据的压缩，对磁盘空间的有效利用和提高查询性能都是非常有益的。

文件格式





## 补充：

服务，接受请求，处理数据，执行操作

decimal：浮点型计算更加精确

​                                                                                                                                                                                                                                                                                                                              

介绍一下hive：

hive是一个基于hadoop生态系统的数据仓库工具，Hive 的应用场景：存储，查询和分析大规模结构化数据（如用户行为分析、日志分析、广告数据分析），提供了一种类似sql的hql语言来管理hadoop中hdfs上的数据，执行引擎可拓展，默认使用mr程序也可支持spark等，也允许用户编写自定义的函数来拓展hiveql功能，。hive的架构:客户端的执行语句通过hiveserver2或者直接执行mr程序来访问metastore，经metastore来查询hdfs上的数据，最后返回给客户端，hive主要用的数据模型“数据库，表，分区，桶（将数据进一步细分，类似数据库的哈希分区。）”













