---
layout:     post
title:      "Yes24 책 추천 알고리즘, 어떻게 구현했나"
subtitle:   "알고리즘의 디자인과 구현"
category:   Data Analysis
date:       2016-02-02 11:15:00
author:     "Hyunje"
tags: [Yes24, Recommendation, Spark, Machine Learning]
image:
  background: white.jpg
  feature: bg1.jpg
  credit: Hyunje Jo
keywords:	"추천 알고리즘, 데이터 분석, Yes24 책 추천, Apache Spark, 머신 러닝, 추천, Recommendation"
---


얼마전 한국 정보화 진흥원이 관리하는 [개방형 문제해결 플랫폼](http://crowd.kbig.kr)에 올라온 Yes24 도서 추천 알고리즘 대회가 종료되었다.

총 230여명이 참여하였고, 25팀이 최종 결과물을 제출한 대회였다. 이 대회에서 친구와 같이 참여했으며, 입상은 아니지만 우수한 알고리즘 혹은 분석결과를 제시한 팀에 뽑혔다. [결과](http://crowd.kbig.kr/board/notice_view.php?srno=3)

대회 문제와 간략한 설명들은 [지난 포스트](http://hyunje.com/data%20analysis/2015/12/21/yes24-recommendation-1/)에 정리하였다. 이 포스트에서는 어떻게 추천 알고리즘을 구성하였고, 어떻게 Apache Spark를 이용하여 추천을 수행하였는지에 대해 설명할 것이다.

<br>
<br>

## 1. 레포지토리의 구성

현재 [Github Repository](https://github.com/RetrieverJo/yes24)에 업로드 되어 있는 코드는 크게 세 가지의 클래스로 구성되어 있으며, 대략적인 목적은 다음과 같다.

- org.herring.Comparison
	
	대조군을 먼저 형성하였었다. [이전에 작성하였던 포스트](http://hyunje.com/data%20analysis/2015/07/27/advanced-analytics-with-spark-ch3-2/)에서 사용한 AUC 값을 측정하는 것으로 하였으며, Apache Spark에서 제공하는 추천 알고리즘을 이용하여 추천을 수행했을 때에 대해 계산하였다. 여기서 사용된 방식을 구현한 추천 알고리즘에도 적용하여 성능을 평가할 계획이었다.

- org.herring.PreProcessing

	대회측에서 제공된 데이터파일을 전처리를 수행하는 과정에 대한 클래스이다. 현재 레포지토리의 `data`폴더 안의 **filters**, **bookWithId**, **uidIndex** 파일을 생성하는 과정이다. 대회측에서 제공한 파일을 필요로 하기 때문에 수행은 불가능하다. 따라서 레포지토리에 업로드된 파일을 HDFS의 적절한 경로(/yes24/data/)에 업로드 하는 것으로 대체한다.

- org.herring.LDAUserALSWholeProcess

	실제 추천 알고리즘을 수행하는 클래스이다. 코드 안에 주석으로 되어 있는 **spark-submit** 커맨드와 같이 수행을 시킬 수 있다.
	
Apache Maven 으로 구성되어 있으며, ``mvn package``를 수행 한 후에 생성되는 target 폴더의 `yes24-1.0-allinone.jar` 파일을 spark-submit 을 이용해 스파크에 제출하면 된다.
	
	
<br>
<br>

## 2. 소스코드의 흐름

소스코드는 크게 다음과 같은 과정으로 수행된다.

1. 전처리 결과를 불러오는 과정
2. LDA를 수행하기 위한 데이터 준비
3. LDA를 이용한 사용자 클러스터링
4. 각 클러스터별로 추천을 수행
5. 클러스터별 추천 결과를 이용한 최종 추천 결과 생성


<br>
<br>

## 3. 세부 수행 과정

전체 코드는 [Github Repository](https://github.com/RetrieverJo/yes24) 에서 확인할 수 있다.

### 3.1 전처리 결과를 불러오는 과정

전처리 결과는 `org.apache.spark.sql.Row`의 객체로 저장되어 있다. 현재는 모든 카테고리에 속하는 데이터들을 전체에 대해 LDA 클러스터링을 수행하고, 추천을 수행한다.

```scala
val filters = sc.objectFile[Row](filtersPath)
//                .filter(r => r.getAs[String]("category") == "인문")
//                .filter(r => r.getAs[String]("category") == "자기계발")
//                .filter(r => r.getAs[String]("category") == "국내문학")
//                .filter(r => r.getAs[String]("category") == "해외문학")
//                .filter(r => r.getAs[String]("category") == "종교")
```

위 코드에서 각 카테고리의 주석을 해제하면 해당 카테고리 데이터만 필터링 하여 수행하게 된다.

다른 전처리 결과 로드 과정은 생략하도록 한다.

<br>

### 3.2 LDA를 수행하기 위한 데이터 준비

이전 포스트에서 설명하였듯, 구성한 알고리즘은 각 사용자가 구매한 책들의 소개글 정보를 하나의 Document로 만들기 위해선 소개글에 대해 형태소 분석을 수행해야 한다.

형태소 분석에서는 Twitter에서 공개한 [형태소 분석기](https://github.com/twitter/twitter-korean-text)를 이용하여 다음과 같이 형태소 분석을 수행하고, 명사만 추출하였다.

```scala
val bookStemmed = bookData.map { case (id, intro) =>
    val normalized: CharSequence = TwitterKoreanProcessor.normalize(intro)
    val tokens: Seq[KoreanToken] = TwitterKoreanProcessor.tokenize(normalized)
    val stemmed: Seq[KoreanToken] = TwitterKoreanProcessor.stem(tokens)

    val nouns = stemmed
    	.filter(p => p.pos == KoreanPos.Noun)
    	.map(_.text)
    	.filter(_.length >= minLength)
    	
    (id, nouns) //(책 Id, Seq[책 소개에 등장한 명사])
}
```

그리고 다음과 같이 사용자가 구매한 모든 책의 명사를 합한 후에 Wordcount 와 합하여 사용자별 Document를 생성한다.

```scala
//사용자가 구매한 모든 책의 소개글의 명사를 합한 RDD 생성
val userNouns = userItem.groupByKey().mapValues { v =>
    val temp: Iterable[Seq[String]] = v.map(bStemmedMap.value.getOrElse(_, Seq[String]()))
    val result = temp.fold(Seq[String]()) { (a, b) => a ++ b }
    result //(사용자 ID, Seq[명사])
}   

//LDA와 클러스터링에 사용될 사용자별 Document 생성
val documents = userNouns.map { case (id, nouns) =>
    val counts = new mutable.HashMap[Int, Double]()
    nouns.foreach { term =>
        if (bWordMap.value.contains(term)) {
            val idx = bWordMap.value(term)
            counts(idx) = counts.getOrElse(idx, 0.0) + 1.0
        }
    }
    //Creates a sparse vector using unordered (index, value) pairs.
    //(사용자 ID, Vector[각 단어의 Count])
    (id.toLong, Vectors.sparse(bWordMap.value.size, counts.toSeq))
}
```

<br>

### 3.3 LDA를 이용한 사용자 클러스터링

다음 과정은 LDA를 수행하고, 그 결과를 이용해 클러스터링을 수행하는 과정이다. 우선, 다음과 같이 준비한 데이터를 이용해 LDA를 수행한다.

```scala
val lda = new LDA().setK(numTopics).setMaxIterations(maxLDAIters).setCheckpointInterval(10).setOptimizer("em")
```

수행한 LDA의 결과물을 이용해 클러스터링을 수행한다.

```scala
def ldaResultBasedClustering(ldaModel: DistributedLDAModel)
    : (RDD[(Long, Array[Int], Array[Double])], RDD[(Long, Int, Double)], RDD[(Long, Int)]) = {

    //LDA 수행 결과 기반의 클러스터링 수행
    //RDD[(사용자 ID, Array[Topic ID], Array[Topic과의 연관도])]
    val userTopicDistribution = ldaModel.topTopicsPerDocument(topClusterNum) 
    val clusteringResult = userTopicDistribution.flatMap { case (uid, tids, tweights) =>
        tids.map(t => (uid, t)).zip(tweights).map(t => (t._1._1, t._1._2, t._2))
    }   //RDD[(사용자 ID, Topic ID, Topic과의 연관도)]
    val userCluster = clusteringResult.map(i => (i._1, i._2))   //RDD[(사용자 ID, Topic ID)]
    (userTopicDistribution, clusteringResult, userCluster)
}
```

`topTopicsPerDocument(num)` 함수를 수행하면 LDA의 결과물인 Document - Topic Distribution Matrix 에서 각 Document 별로 가장 높은 확률값은 갖는 Topic을 num 개 만큼 추출해 준다.

이 함수의 결과물로 `RDD[(Long, Array[Int], Array[Double])]`이 반환되며, 이것은 **RDD[(사용자 ID, Array[Topic ID], Array[Topic과의 연관도])]**를 의미한다.

이 RDD를 세 가지 형태의 결과물의 형태로 변경하여 반환한다.

<br>

### 3.4 클러스터별 추천 수행

각 클러스터별로 추천을 수행하기 위해서는 Apache Spark에서 제공하는 추천 알고리즘의 입력 형태와 맞추어야 한다. 따라서 다음과 같이 각 클러스터별로 그 클러스터에 해당하는 사용자들이 구매한 책 정보를 모아 RDD로 생성한다.

```scala
def prepareRecommendation(userItem: RDD[(String, String)],
                          groupedUserCluster: RDD[(Long, Iterable[Int])])
: RDD[(Int, Array[Rating])] = {

    //각 클러스터별로 추천을 수행하기 위한 데이터 Filtering
    //ratingForEachCluster: 각 클러스터별로 (클러스터 Id, Array[Ratings])
    val ratingForEachCluster = userItem.map(i => (i._1.toLong, Rating(i._1.toInt, i._2.toInt, 1.0))).groupByKey()
            .join(groupedUserCluster) //RDD[(사용자 ID, Iter[Rating])] + RDD[(사용자 ID, Iter[Cluster ID])]
            .flatMap { uidRatingCluster =>
        val uid = uidRatingCluster._1
        val ratings = uidRatingCluster._2._1.toSeq
        val clusters = uidRatingCluster._2._2
        clusters.map(cnum => (cnum, ratings)) //(Cluster ID, Seq[Rating])을 FlatMap 으로 생성
    }.groupByKey() //(Cluster ID, Iter[Seq[Rating]])
            .mapValues(_.reduce((a, b) => a ++ b)) //(Cluster ID, Seq[Rating])
            .mapValues(_.toArray) //(Cluster ID, Array[Rating])
    ratingForEachCluster //각 클러스터에 해당된 사람들이 구매한 아이템을 이용한 Rating 정보
}
```

위 RDD를 바탕으로 각 클러스터별로 추천 알고리즘을 수행한다.

```scala
def runRecommendation(sc: SparkContext,
                      bUserCluster: Broadcast[Array[(Long, Int)]],
                      ratingForEachCluster: RDD[(Int, Array[Rating])])
: RDD[(Int, Int, Array[Rating])] = {

    val numOfClusters = ratingForEachCluster.count().toInt
    val recResult = new ArrayBuffer[(Int, Int, Array[Rating])]()

    for (cnum <- 0 until numOfClusters) {
        val ratings = ratingForEachCluster.filter(_._1 == cnum).take(1).head._2 //현재 클러스터에 해당하는 Rating만 추출
        val ratingsRdd = sc.parallelize(ratings)
        ratingsRdd.persist(StorageLevel.MEMORY_AND_DISK)

        //추천 수행
        val model: MatrixFactorizationModel = ALS.trainImplicit(ratingsRdd, rank, numRecIterations, lambda, alpha)
        val users = bUserCluster.value.filter(_._2 == cnum).map(_._1.toInt) //현재 클러스터에 해당하는 사용자 추출

        //각 사용자별로 추천 결과 생성
        for (uid <- users) {
            val rec = model.recommendProducts(uid, rank) //Array[Rating]
            recResult += ((uid, cnum, rec)) //Array[(사용자 ID, Cluster ID, 추천 결과 Array[Rating])]
        }
        ratingsRdd.unpersist()
    }
    val recResultRdd = sc.parallelize(recResult)
    recResultRdd
}
```

추천을 수행한 후에, 각 사용자별로 해당 클러스터에서 추천을 *rank*개씩 받고 그 결과를 반환한다. 이 결과를 이후에 수행할 최종 추천에서 활용한다.

<br>

### 3.5 클러스터별 추천 결과를 이용한 최종 추천 결과 생성

최종 추천 결과를 생성할 때는 다음 수식을 이용해 각 아이템별로 가중평균을 수행하였다.
<center>

\\(
{ R }\_{ u,i }=\frac { \sum\_{ c\in C }^{  }{ { w }\_{ u,c }{ r }\_{ u,i,c } }  }{ \sum\_{ c\in C }^{  }{ { w }\_{ u,c } }  } 
\\)

</center>

\\({R}\_{u,i}\\)는 사용자 \\(u\\)의 아이템 \\(i\\)에 대한 최종 예측 Rating 값이고, \\({w}\_{u,c}\\)는 사용자 \\(u\\)와 클러스터 \\(c\\) 사이의 연관도, \\({r}\_{u,i,c}\\)는 각 클러스터별 추천 알고리즘이 예측한 사용자 \\(u\\)의 클러스터 \\(c\\)에서 아이템 \\(i\\)에 대한 예측 Rating 값이다.

이 \\({R}\_{u,i}\\)를 각 사용자별 아이템들에 대해 계산하고, 그것을 내림차순으로 정렬하여 최종 추천으로 한다.

이 과정에 대한 코드는 다음과 같다.

```scala
//추천 결과와 클러스터별 가중치를 이용한 추천 계산
val userDistSum = userTopicDistribution.map { dist => (dist._1.toInt, dist._3.sum) }.collectAsMap() //가중평균의 분모, (사용자 ID, 가중치들의 합)
val recResultTuple = recResultRdd.map(l => ((l._1, l._2), l._3)) //((사용자 ID, Cluster ID), 아이템 Array[Rating])

//((사용자 ID, Cluster ID), 유사도) JOIN ((사용자 ID, Cluster ID), Array[Rating])
val userItemSim = clusteringResult.map(l => ((l._1.toInt, l._2), l._3)).join(recResultTuple)

val finalRecommendationResult = userItemSim.flatMap { case ((uid, cid), (sim, ratings)) => //((사용자 ID, Cluster ID), (유사도, 아이템))
    ratings.map(r => ((uid, r.product), r.rating * sim)) //((사용자 ID, 아이템 ID), 아이템에 대한 Rating 추정치 * Cluster와의 유사도))
}.groupByKey().map { case ((uid, iid), ratings) => //(사용자 ID, 아이템 ID)를 Key 로 하여 reduce
    val itemSum = ratings.sum //아이템에 대한 Rating 추정치 * 유사도의 합
    val distSum = userDistSum(uid) //모든 유사도의 합
    Rating(uid, iid, itemSum / distSum) //가중평균 계산 후 Rating 결과를 Rating 객체로 Wrapping
}.groupBy(_.user).map { case (uid, itemRatings) => //사용자 별로 추천 받은 아이템들을 reduce
    val sortedItems = itemRatings.toArray.sortBy(-_.rating).take(rank) //내림차순으로 정렬하여 상위 N개 추출
    (uid, sortedItems)
}
```

<br>
<br>

## 4. 추후 진행할 작업

아직 성능 평가가 이루어지지 않았다. 다음으로 진행할 작업은 성능평가를 진행하여 단순히 ALS를 이용한 추천만 수행했을 때 보다 과연 성능이 좋은지 판단해 봐야 할 것이다.

또한 알고리즘에 대한 설명이 부족한 것 같다. 다이어그램 혹은 예시 데이터를 이용하여 수행 가능하도록, 혹은 Follow-up 할 수 있도록 편집하는 것이 필요해보인다.