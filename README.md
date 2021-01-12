# 使用Docker一键搭建并实战大数据开发环境 Hadoop Hive Spark ES Kibana

## 背景

在学习大数据的时候，我发现以下痛点：

1. 环境搭建复杂；
2. 搭建完成不知道如何入手；
3. 大数据搭建、使用的资料散落各处且良莠不齐。

在收集、实验、总结了各方资料的基础上，现借助Docker一键搭建大数据开发环境，相当简单，可以极大的提高学习的便利性，希望对大家有帮助。

git地址： https://github.com/ArvinLiubaby/BigData_Practice_Via_Docker



## 准备

- git
- docker环境（docker & docker-compose）

## Steps

### 3.1 git clone repository

```shell
git clone https://github.com/ArvinLiubaby/docker-hadoop-spark-hive.git
```

*注：该repository fork自 [ibywind/docker-hadoop-spark-hive: docker-hadoop-spark-hive 快速构建你的大数据环境 (github.com)](https://github.com/ibywind/docker-hadoop-spark-hive) ，因本机测试过程中Hive始终无法启动，故自行修改了Hive相关的信息。*

### 3.2 一键搭建环境

clone完代码，直接运行`run.sh`即可，最好使用国内的docker源，比如163的。

```sh
./run.sh
```

经过一段时间等待之后，环境搭建完成。

### 3.3 环境验证

#### 3.3.1 browser访问spark hadoop es kibana等

本机访问如下几个地址：

```tex
localhost:8080	-- spark
localhost:50070 -- hadoop NN
localhost:9200	-- ES
localhost:5601	-- Kibana
```

如果可以正确打开则没问题。

#### 3.3.2 进入hive：

进入容器

bash前面的是你的容器ID，通过`docker ps`可以看到。

```shell
docker exec -it b002969ab690 bash
```

进入beeline

```shell
beeline -u jdbc:hive2://localhost:10000
```

![image-20210104160722380](C:\Users\l30009462\AppData\Roaming\Typora\typora-user-images\image-20210104160722380.png)

至此已经进入Hive环境，可以自行尝试hive的语句。



#### 3.3.3 使用Spark
##### 1 Python版本

前提（*注意：该前提仅是方便本地开发调试，不影响程序在集群的运行，即：就算你不具备这些条件，你也可以提交到集群运行你的代码*）：

- python3

- pyspark包
- pycharm

python版本很适合学习Spark使用。首先我们使用pycharm在本地写个简单的程序：

只要你本地`pip install pyspark`即可。

`test.py`

```python
# -*- coding: utf-8 -*-
from pyspark import SparkConf, SparkContext

conf = SparkConf().setAppName("spark_python")  # 不要setMaster('local')，否则是本地执行，不会调用集群，所以localhost:8080 看不到执行任务。
sc = SparkContext(conf=conf)

input_paths = ['test.txt', 'hdfs://namenode:8020/input/test.txt']
output_paths = ['result', 'hdfs://namenode:8020/output/result']


def replace_path(ind=0):
    return input_paths[ind], output_paths[ind]


input_path, output_path = replace_path()
# input_path, output_path = replace_path(1)
lines = sc.textFile(input_path)


def sp(ln):
    ss = ln.strip().split(',')
    name = str(ss[0])
    eng = int(ss[1])
    math = int(ss[2])
    lang = int(ss[3])
    # return name, eng, math, lang
    return name, eng + math + lang


rdd = lines.map(sp)
# rdd = lines.map(sp).collect()
for line in rdd.collect():
    print(line)

rdd.saveAsTextFile(output_path)
# rdd.repartition(1).saveAsTextFile(output_path)
```

其中`test.txt`如下：

```txt
zhangsan,77,88,99
lisi,56,78,89
wanger,78,77,67
```

直接运行，可以正常输出即可。

![image-20210112092555620](C:\Users\l30009462\AppData\Roaming\Typora\typora-user-images\image-20210112092555620.png)

下面在刚刚搭建的spark集群中运行我们的python代码。

记得取消刚刚的注释，test.py修改如下：

```python
# -*- coding: utf-8 -*-
from pyspark import SparkConf, SparkContext

conf = SparkConf().setAppName("spark_python")  # 不要setMaster('local')，否则是本地执行，不会调用集群，所以localhost:8080 看不到执行任务。
sc = SparkContext(conf=conf)

input_paths = ['test.txt', 'hdfs://namenode:8020/input/test.txt']
output_paths = ['result', 'hdfs://namenode:8020/output/result']


def replace_path(ind=0):
    return input_paths[ind], output_paths[ind]


# input_path, output_path = replace_path()
input_path, output_path = replace_path(1)
lines = sc.textFile(input_path)


def sp(ln):
    ss = ln.strip().split(',')
    name = str(ss[0])
    eng = int(ss[1])
    math = int(ss[2])
    lang = int(ss[3])
    # return name, eng, math, lang
    return name, eng + math + lang


rdd = lines.map(sp)
# rdd = lines.map(sp).collect()
for line in rdd.collect():
    print(line)

rdd.saveAsTextFile(output_path)
# rdd.repartition(1).saveAsTextFile(output_path) # 输出一个文件
```

首先在Ubuntu/WSL2中，新建两个文件，分别为刚才的测试代码和文件（`test.py`&`test.txt`）。

赋予权限

```shell
chmod 755 test.py
```

通过`docker ps`命令查看并找到`spark-master`。

![image-20210109182618102](C:\Users\l30009462\AppData\Roaming\Typora\typora-user-images\image-20210109182618102.png)

我们首先进入spark-master容器，查看目录

```shell
docker exec -it container_id bash
```

进入容器的以下路径，就可以看到熟悉的shell&cmd命令了。

```shell
root@spark-master:/# cd /spark/bin
```

![image-20210109182839892](C:\Users\l30009462\AppData\Roaming\Typora\typora-user-images\image-20210109182839892.png)

回到Linux中，分别上传test.py和test.txt。

```shell
docker cp test.py spark-master:/spark/bin/
docker cp test.txt spark-master:/spark/bin/
```

在spark-master中，把test.txt上传到HDFS中：

```shell
hdfs dfs -mkdir /input
hdfs dfs -mkdir /output
hdfs dfs -put test.txt /input
```

使用spark集群运行代码

```shell
./spark-submit --master spark://spark-master:7077 test.py
```

![image-20210112100415210](C:\Users\l30009462\AppData\Roaming\Typora\typora-user-images\image-20210112100415210.png)

观察控制台，可正常输出：

![image-20210109183535772](C:\Users\l30009462\AppData\Roaming\Typora\typora-user-images\image-20210109183535772.png)

同时登陆`localhost:8080`可以看到我们刚刚提交的任务信息：

![image-20210112100804928](C:\Users\l30009462\AppData\Roaming\Typora\typora-user-images\image-20210112100804928.png)

访问`http://localhost:50070/`，查看HDFS的内容：Utilities - Browse the file system

![image-20210112100908748](C:\Users\l30009462\AppData\Roaming\Typora\typora-user-images\image-20210112100908748.png)

继续访问HDFS的路径，可以看到我们的结果：

![image-20210112101119534](C:\Users\l30009462\AppData\Roaming\Typora\typora-user-images\image-20210112101119534.png)

如果你只想输出一个文件，请取消该行注释：

```python
rdd.repartition(1).saveAsTextFile(output_path)
```

##### 2 Java版本

python版本比较简单，但很少用，我们在提供一个WordCount的Java版本代码：

前提（*注意：该前提仅是方便本地开发调试，不影响程序在集群的运行，即：就算你不具备这些条件，你也可以提交到集群运行你的代码*）：

- IDEA
- Java8
- maven

`WordCount.java`

```java
import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaPairRDD;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;
import scala.Tuple2;

import java.text.SimpleDateFormat;
import java.util.Arrays;
import java.util.Date;
import java.util.List;


/**
 * @author arvin
 */
public class WordCount {

    public static void main(String[] args) {
//        String hdfsHost = args[0];
//        String hdfsPort = args[1];
//        String textFileName = args[2];
        String hdfsHost = "namenode";
        String hdfsPort = "8020";
        String textFileName = "test.txt";

//        SparkConf sparkConf = new SparkConf().setMaster("local").setAppName("Spark WordCount Application (java)");
        SparkConf sparkConf = new SparkConf().setAppName("Spark WordCount Application (java)");

        JavaSparkContext javaSparkContext = new JavaSparkContext(sparkConf);

        String hdfsBasePath = "hdfs://" + hdfsHost + ":" + hdfsPort;
        //文本文件的hdfs路径
        String inputPath = hdfsBasePath + "/input/" + textFileName;

        //输出结果文件的hdfs路径
        String outputPath = hdfsBasePath + "/output/"
                + new SimpleDateFormat("yyyyMMddHHmmss").format(new Date());

        System.out.println("input path : " + inputPath);

        System.out.println("output path : " + outputPath);

        //导入文件
        JavaRDD<String> textFile = javaSparkContext.textFile(inputPath);

        JavaPairRDD<String, Integer> counts = textFile
                //每一行都分割成单词，返回后组成一个大集合
                .flatMap(s -> Arrays.asList(s.split(" ")).iterator())
                //key是单词，value是1
                .mapToPair(word -> new Tuple2<>(word, 1))
                //基于key进行reduce，逻辑是将value累加
                .reduceByKey(Integer::sum);

        //先将key和value倒过来，再按照key排序
        JavaPairRDD<Integer, String> sorts = counts
                //key和value颠倒，生成新的map
                .mapToPair(tuple2 -> new Tuple2<>(tuple2._2(), tuple2._1()))
                //按照key倒排序
                .sortByKey(false);

        //取前10个
        List<Tuple2<Integer, String>> top10 = sorts.take(10);

        //打印出来
        for (Tuple2<Integer, String> tuple2 : top10) {
            System.out.println(tuple2._2() + "\t" + tuple2._1());
        }

        //分区合并成一个，再导出为一个txt保存在hdfs
        javaSparkContext.parallelize(top10).coalesce(1).saveAsTextFile(outputPath);

        //关闭context
        javaSparkContext.close();
    }
}
```

`pom.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.bolingcavalry</groupId>
    <artifactId>sparkwordcount</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-core_2.11</artifactId>
            <version>2.3.2</version>
        </dependency>
    </dependencies>

    <build>
        <sourceDirectory>src/main/java</sourceDirectory>
        <testSourceDirectory>src/test/java</testSourceDirectory>
        <plugins>
            <plugin>
                <artifactId>maven-assembly-plugin</artifactId>
                <configuration>
                    <descriptorRefs>
                        <descriptorRef>jar-with-dependencies</descriptorRef>
                    </descriptorRefs>
                    <archive>
                        <manifest>
                            <mainClass></mainClass>
                        </manifest>
                    </archive>
                </configuration>
                <executions>
                    <execution>
                        <id>make-assembly</id>
                        <phase>package</phase>
                        <goals>
                            <goal>single</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
            <plugin>
                <groupId>org.codehaus.mojo</groupId>
                <artifactId>exec-maven-plugin</artifactId>
                <version>1.2.1</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>exec</goal>
                        </goals>
                    </execution>
                </executions>
                <configuration>
                    <executable>java</executable>
                    <includeProjectDependencies>false</includeProjectDependencies>
                    <includePluginDependencies>false</includePluginDependencies>
                    <classpathScope>compile</classpathScope>
                    <mainClass>com.bolingcavalry.sparkwordcount.WordCount</mainClass>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

在maven中，clean，install，package之后，在target中会生成对应的jar包。

![image-20210112102044824](C:\Users\l30009462\AppData\Roaming\Typora\typora-user-images\image-20210112102044824.png)

![image-20210112102119899](C:\Users\l30009462\AppData\Roaming\Typora\typora-user-images\image-20210112102119899.png)

也可以通过命令打jar包：

```shell
mvn clean package -Dmaven.test.skip=true
```

把jar包复制到spark-master容器中，然后运行即可：

```shell
docker cp sparkwordcount-1.0-SNAPSHOT.jar spark-master:/spark/bin	-- 复制
./spark-submit --master spark://spark-master:7077 --class WordCount sparkwordcount-1.0-SNAPSHOT.jar  -- 运行
```

同样的，我们可以在`localhost:8080`和HDFS中查看到对应内容：

![image-20210112102348339](C:\Users\l30009462\AppData\Roaming\Typora\typora-user-images\image-20210112102348339.png)

##### 3 Scala 版本

一般生产都会采用Scala版本，毕竟Spark就是用Scala写的。





参考资料：

[ibywind/docker-hadoop-spark-hive: docker-hadoop-spark-hive 快速构建你的大数据环境 (github.com)](https://github.com/ibywind/docker-hadoop-spark-hive)

[入门spark的第一节课：docker spark镜像集群部署与spark java程序编写并运行_pingweicheng的博客-CSDN博客_spark 镜像](https://blog.csdn.net/pingweicheng/article/details/104886148)

