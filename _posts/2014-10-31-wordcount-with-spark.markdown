---
layout:     post
title:      "Spark 기반의 Wordcount 예제 수행하기"
subtitle:   ""
category:	  Data Analysis
date:       2014-10-31 11:18:00
author:     "Hyunje"
tags: [Spark]
image:
  background: white.jpg
  feature: bg4.jpg
  credit: Hyunje Jo
---


이 글은 Spark 기반의 Wordcount Application을 작성하고, 그것을 YARN을 통해 deploy 하는 과정에 대한 설명입니다.
<br>

### WARNING
이 글은 최신버전을 기준으로 설명된 글이 아닙니다. 최신 버전을 대상으로 하였을 때와 설치 과정 혹은 출력 결과가 다를 수 있습니다.

<br>

#### Environment

- Hadoop 2.5.1 - [설치과정](http://hyunje.com/post/os-xe-hadoop2-dot-5-1-seolcihagi/)
- Spark 1.1.0 - [설치과정](http://hyunje.com/post/spark-1-dot-1-0-seolci,-hadoop-2-dot-5-.1gwayi-yeondong/)
- IntelliJ IDEA
- Maven 

<br>

#### Dependency
pom.xml에 다음과 같은 dependency를 추가합니다.

```xml
<dependency>
    <groupId>org.apache.spark</groupId>
    <artifactId>spark-core_2.10</artifactId>
    <version>1.1.0</version>
</dependency>
```

<br>

#### Wordcount Example
다음과 같은 형태로 wordcount 프로그램을 작성합니다.

```java
//Create spark context
SparkConf conf = new SparkConf().setAppName("Spark Word-count").setMaster("yarn-cluster");
JavaSparkContext context = new JavaSparkContext(conf);

//Split using space
JavaRDD<String> lines = context.textFile(INPUT_PATH);
JavaRDD<String> words = lines.flatMap(new FlatMapFunction<String, String>() {
    @Override
    public Iterable<String> call(String s) throws Exception {
        return Arrays.asList(s.split(" "));
    }
});

//Generate count of word
JavaPairRDD<String, Integer> onesOfWord = words.mapToPair(new PairFunction<String, String, Integer>() {
    @Override
    public Tuple2<String, Integer> call(String s) throws Exception {
        return new Tuple2<String, Integer>(s, 1);
    }
});

//Combine the count.
JavaPairRDD<String, Integer> wordCount = onesOfWord.reduceByKey(new Function2<Integer, Integer, Integer>() {
    @Override
    public Integer call(Integer integer, Integer integer2) throws Exception {
        return integer + integer2;
    }
});

//Save as text file.
wordCount.saveAsTextFile(OUTPUT_PATH);

context.stop();
```

전체 소스코드는 [github page](https://github.com/RetrieverJo/Spark-Example)에 있습니다.

<br>

#### Run Spark Application
maven을 이용하여 프로젝트를 패키징 합니다.

```bash
$ mvn package
```
다음과 같은 명령어를 이용하여 작성한 Wordcount application을 YARN에 submit 합니다.

```bash
$ spark-submit --class 패키지명.클래스명 --master yarn-cluster Package된Jar파일.jar
```
