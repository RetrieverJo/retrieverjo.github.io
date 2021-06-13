---
layout:     post
title:      "Spark RDD의 함수 동작 방식"
subtitle:   ""
category:	  Data Analysis
date:       2014-11-06 14:09:00
author:     "Hyunje"
tags: [Spark, Spark RDD]
image:
  background: white.jpg
  feature: bg2.jpg
  credit: Hyunje Jo
---


이 글은 Spark의 RDD에 존재하는 함수들이 어떤 방식으로 동작되는지에 대한 글입니다. <br>
[Spark 의 공식 Documentation](http://spark.apache.org/docs/latest/programming-guide.html#resilient-distributed-datasets-rdds)을 일부 번역하였습니다.<br>
또한, Java 기준으로 설명 할 것이며, Scala 로 된 버전은 추후에 추가 작성할 계획입니다.<br>
번역하기에 적절치 않은 용어들은 영문 단어 그대로 남겨놓았습니다.
<hr>
<br>

### WARNING
이 글은 최신버전을 기준으로 설명된 글이 아닙니다. 최신 버전을 대상으로 하였을 때와 설치 과정 혹은 출력 결과가 다를 수 있습니다.

<br>

### Spark의 RDD
Spark는 Resilient Distributed Dataset, RDD 로 구성되어 있습니다. 이 RDD는 분산 형태로 처리 가능한 fault-tolerant collection 입니다. Spark에서 RDD를 생성하는 방법은 두 가지가 있습니다. 첫 번째로는 Driver 프로그램에서 이미 존재하는 colleciton을 *parallelizing* 시키는 방법이 있고, 다른 방법으로는 HDFS나 HBase혹은 Hadoop InputFormat 으로 수행 가능한 어떠한 데이터 소스를 *referencing*하는 방법이 있습니다.

<br>

##### Parallelized Collections
Java에서 Parallelized Collection을 생성하기 위해서는 `JavaSparkContext`클래스에 존재하는 `parallelize` 함수에 Driver 프로그램에서 사용한 Collection을 파라미터로 넘겨주면 됩니다. Collection에 존재하는 엘리먼트들은 Distributed Dataset으로 복사되고, 분산 형태로 연산됩니다. 1 에서 5 까지 값을 갖는 리스트를 parallelized collection으로 생성하는 방법은 다음 예와 같습니다.

```java
List<Integer> data = Arrays.asList(1, 2, 3, 4, 5);
JavaRDD<Integer> distData = sc.parallelize(data);
```
한번 생성된 distributed dataset(`distData`)는 분산 형태로 연산될 수 있습니다. 예를들면, 다음과 같은 코드를 이용하여 리스트에 존재하는 모든 값을 더할 수 있습니다.

```java
distData.reduce((a, b) -> a + b);
```
이러한 분산 형태의 연산에 대해서는 아래 Section 에서 설명할 것입니다.

```
이 Documentation에서는 Java 8 에서 제공하는 Lamda 문법을 사용하고 있습니다. Java 8을 사용할 수 없는 상황이어서 lambda 표현식을 사용하지 못할 때는, org.apache.spark.api.java.function 패키지를 구현하여 사용할 수 있습니다. 이에 대해서는 아래 Section에서 설명할 것입니다.
```
parallel collection에서 중요한 파라메터중 하나는 데이터셋을 몇 개의 `slices`로 나눌 것인지에 대한 것입니다. Spark는 각 cluster의 조각마다 하나의 태스크를 수행 하게 됩니다. 일반적으로 클러스터에서 각각의 CPU 마다 2 ~ 4 개의 slice를 합니다. Spark는 클러스터를 기반으로 slice의 수를 자동적으로 생성을 시도하지만, `parallelize`를 수행할 때 그 개수를 수동으로 지정해 줄 수 있습니다. (e.g. `sc.parallelize(data, 10)`)

<br>

#### External Datasets
Spark에서는 Hadoop과 연관되는 모든 데이터 소스(Local File System, HDFS, Cassandra, HBase, Amazon S3, etc.)를 이용하여 distributed dataset을 생성할 수 있습니다. Spark는 텍스트파일, [Sequence File](http://hadoop.apache.org/common/docs/current/api/org/apache/hadoop/mapred/SequenceFileInputFormat.html)과 모든 Hadoop [InputFormat](http://hadoop.apache.org/docs/stable/api/org/apache/hadoop/mapred/InputFormat.html)을 지원합니다.

텍스트 파일의 RDD는 `SparkContext`의 `textFile` 함수를 사용하여 생성될 수 있으며, 이 함수는 파일의 URI를 이용하여 파일에 접근합니다. 그리고 텍스트 파일의 한 라인의 collection으로 읽고, 다음과 같은 형태로 파일을 불러옵니다.

```java
JavaRDD<String> distFile = sc.textFile("data.txt");
```
한번 생성이 되면, `distFile`은 dataset 연산들을 수행할 수 있게 됩니다. 예를들면 `map`과 `reduce`를 이용하여 모든 라인의 길이를 더한 갚을 구할때는 다음과 같이 명령어를 수행하면 됩니다.

```java
distFile.map(s -> s.length()).reduce(a, b) -> a + b)
```
Spark를 이용하여 파일을 읽을 때, 다음 사항들을 참고할 수 있습니다.

* Local File System 에 있는 파일에 접근할 때, 반드시 Worker node에서 접근 가능한 경로에 파일이 있어야 한다. 모든 Worker에게 파일을 복사하거나, network-mounted 로 공유된 파일 시스템을 사용해야 합니다.<br><br>
* Spark의 file-based 입력 함수는 폴더, 압축파일과 와일드카드 표현을 포함하여 사용될 수 있습니다. 예를 들면 `textFile("/my/directory")`, `textFile("/my/directory/*.txt")`, `textFile("/my/directory/*/gz")`가 모두 가능합니다.<br><br>

Spark는 텍스트 파일 이외에도 많은 데이터 포맷을 제공합니다.

* `JavaSaprkContext.wholeTextFiles`는 하나의 폴더 안에 존재하는 텍스트 파일을 읽습니다. 그리고 그것을 <파일이름, 내용> 형태의 쌍으로 리턴합니다. 이것은 한 파일을 읽어 각각의 줄에 대해 처리하는 `textFile`과는 다른 형태입니다.<br><br>
* [SequenceFiles](http://hadoop.apache.org/common/docs/current/api/org/apache/hadoop/mapred/SequenceFileInputFormat.html)를 읽기 위해서는 SparkContext의 `sequenceFile[K,V]` 함수를 사용해야합니다. K 와 V는 해당 파일에서 사용하고 있는 Key와 Value의 타입입니다. 이것들은 Hadoop의 IntWritable 과 Text 클래스와 같이 [Writable](http://hadoop.apache.org/common/docs/current/api/org/apache/hadoop/io/Writable.html)클래스의 subclass여야 합니다. Spark에서는 이러한 과정을 돕기 위해 몇 개의 일반적인 Writable 타입에 대해 native type을 제공합니다. 예를들어 `sequenceFile[Int,String]`을 사용하면, 이것은 `IntWritable`과 `Text`로 인식됩니다.<br><br>
* 다른 Hadoop InputFormat을 사용하기 위해서는 `JavaSparkContext.HadoopRDD`함수를 사용해야 합니다. 이 함수는 임의의 JobConf와 InputFormat 클래스, Key 클래스, Value 클래스를 받습니다. Hadoop 에서 입력 소스를 지정하는 과정과 같은 방식으로 지정을 해야 합니다. 또한 새 맵리듀스 API(org.apache.hadoop.mapreduce 패키지에 존재하는 API)를 사용하기 위해서는`JavaSparkContext.newHadoopRDD`를 사용해야합니다.<br><br>
* `JavaRDD.saveAsObjectFile`과 `JavaSparkContext.objectFile`은 Serialized된 자바 객체 형태로 출력하는 것을 지원합니다. 이것은 Avro와 같이 특수화 된 형태보다는 효율적이지 않지만 간단한 형태로 어떠한 RDD도 저장 가능합니다.

<br>

#### RDD Operations
추후 번역
