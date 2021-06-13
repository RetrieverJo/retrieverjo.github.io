---
layout:     post
title:      "Spark 기반의 추천 알고리즘 수행"
subtitle:   ""
category:	  Data Analysis
date:       2014-11-05 11:48:00
author:     "Hyunje"
tags: [Spark, Recommendation]
image:
  background: white.jpg
  feature: bg1.jpg
  credit: Hyunje Jo
---


이 글은 Spark를 기반으로 하여 추천 알고리즘을 수행하는 과정에 대한 것입니다.<br>
추천에 사용되는 데이터셋은 [MovieLens](http://grouplens.org/datasets/movielens/)를 기준으로 하였습니다.


### WARNING
이 글은 최신버전을 기준으로 설명된 글이 아닙니다. 최신 버전을 대상으로 하였을 때와 설치 과정 혹은 출력 결과가 다를 수 있습니다.

<br>


#### Environment
- Hadoop 2.5.1 - [설치과정](http://hyunje.com/post/os-xe-hadoop2-dot-5-1-seolcihagi/)
- Spark 1.1.0 - [설치과정](http://hyunje.com/post/spark-1-dot-1-0-seolci,-hadoop-2-dot-5-.1gwayi-yeondong/)
- IntelliJ IDEA
- Maven 
- JDK 1.7

<br>

#### Dependency
Spark의 sub-project인 MLlib 프로젝트에서 이미 추천 알고리즘에 대한 라이브러리를 구현해 놓았습니다. 이를 사용하기 위해서 pom.xml에 다음과 같은 dependency를 추가합니다.

```xml
<dependency>
    <groupId>org.apache.spark</groupId>
    <artifactId>spark-core_2.10</artifactId>
    <version>1.1.0</version>
</dependency>
<dependency>
    <groupId>org.apache.spark</groupId>
    <artifactId>spark-mllib_2.10</artifactId>
    <version>1.1.0</version>
</dependency>
```

<br>

#### Recommendation Module
다음과 같은 형태로 추천 알고리즘을 사용하고 그 결과를 저장합니다.<br>
JDK 1.7을 사용하였기 때문에 람다표현을 사용하지 않았습니다. Java 8을 사용하면 람다표현을 사용할 수 있으며, 좀 더 간략하게 Spark 프로그램을 작성할 수 있습니다.

```java
SparkConf conf = new SparkConf().setAppName("Spark-recommendation").setMaster("yarn-cluster");
JavaSparkContext context = new JavaSparkContext(conf);

JavaRDD<String> data = context.textFile(INPUT_PATH);
JavaRDD<Rating> ratings = data.map(
        new Function<String, Rating>() {
            public Rating call(String s) {
                String[] sarray = s.split(delimiter);
                return new Rating(Integer.parseInt(sarray[0]), Integer.parseInt(sarray[1]),
                        Double.parseDouble(sarray[2]));
            }
        }
);

// Build the recommendation model using ALS
int rank = 10;
int numIterations = 20;
MatrixFactorizationModel model = ALS.train(JavaRDD.toRDD(ratings), rank, numIterations, 0.01);

// Evaluate the model on rating data
JavaRDD<Tuple2<Object, Object>> userProducts = ratings.map(
        new Function<Rating, Tuple2<Object, Object>>() {
            public Tuple2<Object, Object> call(Rating r) {
                return new Tuple2<Object, Object>(r.user(), r.product());
            }
        }
);
JavaPairRDD<Tuple2<Integer, Integer>, Double> predictions = JavaPairRDD.fromJavaRDD(
        model.predict(JavaRDD.toRDD(userProducts)).toJavaRDD().map(
                new Function<Rating, Tuple2<Tuple2<Integer, Integer>, Double>>() {
                    public Tuple2<Tuple2<Integer, Integer>, Double> call(Rating r) {
                        return new Tuple2<Tuple2<Integer, Integer>, Double>(
                                new Tuple2<Integer, Integer>(r.user(), r.product()), r.rating());
                    }
                }
        ));

//<<Integer,Integer>,Double> to <Integer,<Integer,Double>>
JavaPairRDD<Integer, Tuple2<Integer, Double>> userPredictions = JavaPairRDD.fromJavaRDD(predictions.map(
        new Function<Tuple2<Tuple2<Integer, Integer>, Double>, Tuple2<Integer, Tuple2<Integer, Double>>>() {
            @Override
            public Tuple2<Integer, Tuple2<Integer, Double>> call(Tuple2<Tuple2<Integer, Integer>, Double> v1) throws Exception {
                return new Tuple2<Integer, Tuple2<Integer, Double>>(v1._1()._1(), new Tuple2<Integer, Double>(v1._1()._2(), v1._2()));
            }
        }
));

//Sort by key & Save
userPredictions.sortByKey(true).saveAsTextFile(OUTPUT_PATH);
context.stop();
```

전체 소스코드는 [github repository](https://github.com/RetrieverJo/Spark-Example)에 있습니다.

<br>

#### Preparation
Input file은 [Spark repository](https://github.com/apache/spark)의 [ALS 샘플 데이터](https://github.com/apache/spark/blob/master/data/mllib/sample_movielens_data.txt)를 이용하였습니다.<br>
해당 파일을 HDFS에 업로드 한 후,  Input File 로 사용합니다.<br>
위 코드에서는 INPUT_PATH로 정의되었지만, github repository 에서는 Apache Commons-cli 를 이용하여 입력받았습니다.
<br>
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
(github repository에는 Apache Commons-cli 를 이용하여 실제 수행 command는 뒤에 옵션이 추가로 붙습니다.)

<br>

#### Result
수행 결과로 다음과 같은 결과가 출력됩니다.

```
(0,(34,0.9846535656842613))
(0,(96,0.8178838683876802))
...
(1,(96,1.2547672185210839))
(1,(4,1.941481009392396))
...
(29,(86,1.0588376599353693))
(29,(68,3.3195965377284837))
```

위 결과는 각각의 사용자 0 ~ 29에 대해 영화별 평점을 예측한 수치입니다.
