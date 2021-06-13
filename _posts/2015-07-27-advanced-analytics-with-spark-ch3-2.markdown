---
layout:     post
title:      "Spark & 머신 러닝 - Recommending Music - 2/2"
subtitle:   "Advanced Analytics with Spark, 2. Recommending Music - 2/2"
category:   Data Analysis
date:       2015-07-27 21:09:00
author:     "Hyunje"
tags: [Spark, Machine Learning, Recommendation]
image:
  background: white.jpg
  feature: bg3.jpg
  credit: Hyunje Jo
keywords: "데이터 분석, Data Analysis, Spark, Apache Spark, 머신 러닝, Machine Learning"
---

[지난 포스트](http://hyunje.com/data%20analysis/2015/07/13/advanced-analytics-with-spark-ch3-1/)에 이어 ALS를 이용한 추천 알고리즘의 성능을 평가하는 과정에 대한 글이다.

이 포스트는 [Advanced Analytics with Spark](http://shop.oreilly.com/product/0636920035091.do)을 정리한 글이다.
<br>
<br>

## Evaluating Recommendation Quality

추천의 수행 결과를 평가하는 것으로 가장 정확한 방법은, 각 사용자가 추천 결과를 보고 그것에 대해 평가를 내리는 것이 가장 정확한 방식이다. 하지만 이러한 과정은 몇몇의 사용자를 샘플링 하여 진행한다고 하더라도 실제적으로 불가능에 가까운 방식이다. 때문에 사용자들이 들었던 아티스트들은 끌리는 아티스트들이고, 사용자들이 듣지 않은 아티스트들은 그렇지 않은 아티스트라고 가정하여 평가를 수행하는 것이 납득할만한 방법이다. 이러한 가정은 문제를 위한 가정이긴 하지만, 다른 데이터를 추가적으로 사용하지 않고 적용시킬 수 있다는 장점이 있다.

이 방법을 이용하여 추천 모델을 평가하기 위해서는 데이터를 분리하여 분리된 데이터는 ALS 모델을 생성하는 과정에서 제외시키는 과정이 필요하다. 그러면 이 분리된 데이터는 사용자들에 대한 좋은 추천 결과들을 가지고 있는 것으로 해석될 수 있다. 결과적으로 추천 시스템은 분리된 데이터를 제외하고 추천 모델을 생성시킨 후에 추천을 수행할 것이고, 추천이 이상적이라면 추천 시스템이 생성한 추천 결과의 상위권에 이 분리된 데이터들이 존재해야 할 것이다.

추천 결과를 분리한 아티스트의 리스트와 비교하여 0.0에서 1.0의 범위를 갖는 값(높을 수록 좋은 추천 결과를 나타냄)으로 수치화시킬 수 있다. (모든 아티스트의 쌍과 비교할 수 있지만, 그렇게 되면 너무 많은 쌍이 발생할 수 있기 때문에 일부 샘플된 쌍만 비교하는 것으로 한다.) 그리고 여기서 0.5는 무작위로 추천을 수행하였을 때의 기대값이라 한다.

이 방식은 [ROC Curve](https://en.wikipedia.org/wiki/Receiver_operating_characteristic)와 직접적인 연관성을 갖는다. 앞서 얘기한 방식은 [AUC, Area Under the Curve](https://en.wikipedia.org/wiki/Receiver_operating_characteristic#Area_under_curve)을 나타내는데, 이것은 무작위로 생성된 추천 결과에 비해 좋은 추천 결과들이 얼마만큼의 좋은 추천을 수행했는가 판단하는데에 이용된다.

AUC는 일반적인 Binary Classifier 와 같은 일반적인 Classifier 에서도 평가 방법으로 많이 이용된다. Spark의 MLlib에서는 BinaryClassificationMetrics에 이것이 구현되어있다. 이 글에서는 **각 사용자별 AUC**를 계산하고, 그것을 **평균**낼 것이다.

<br>
<br>

## Computing AUC

이 절에서 수행되는 코드는 [지난 포스트](http://hyunje.com/data%20analysis/2015/07/13/advanced-analytics-with-spark-ch3-1/)의 코드까지 수행된 것을 가정한다.

```scala
val allData = rawUserArtistData.map{ line => 
    val Array(userId, artistId, count) = line.split(' ').map(_.toInt)
    val finalArtistId = bArtistAlias.value.getOrElse(artistId, artistId)
    Rating(userId, finalArtistId, count)
}.cache()

val Array(trainData, cvData) = allData.randomSplit(Array(0.9, 0.1))
trainData.cache()
cvData.cache()

val allItemIDs = allData.map(_.product).distinct().collect()
val bAllItemIDs = sc.broadcast(allItemIDs)

val model = ALS.trainImplicit(trainData, 20, 5, 0.01, 1.0)
```

위 코드는 데이터셋을 9:1의 비율로 나누어 90%의 데이터를 트레이닝 데이터로, 나머지 10%의 데이터를 Cross-Validation 데이터로 사용하여 추천 모델을 훈련시키는 과정을 나타낸다.

그리고 다음 코드는, 생성된 추천 모델의 `predict` 함수를 이용하여 AUC를 계산하는 것에 대한 함수이다. 이 함수를 그대로 shell 에 입력하거나, 따로 파일에 작성하여 이전 포스트 에서 수행하였던 방식처럼 불러와도 된다.

```scala
import org.apache.spark.rdd.RDD
import org.apache.spark.broadcast.Broadcast
import scala.collection.mutable.ArrayBuffer
import scala.util.Random

// 각 사용자별로 AUC를 계산하고, 평균 AUC를 반환하는 함수.
def areaUnderCurve(
      positiveData: RDD[Rating],
      bAllItemIDs: Broadcast[Array[Int]],
      predictFunction: (RDD[(Int,Int)] => RDD[Rating])) = {

    // Positive로 판단되는 결과들, 즉 전체 데이터에서 Cross-validation을 하기 위해 남겨둔
    // 10%의 데이터를 이용하여 Positive한 데이터로 저장한다.
    val positiveUserProducts = positiveData.map(r => (r.user, r.product))
    // Positive 데이터에서 (사용자, 아티스트ID)별로 각각의 쌍에 대한 예측치를 계산하고,
    // 그 결과를 사용자별로 그룹화한다.
    val positivePredictions = predictFunction(positiveUserProducts).groupBy(_.user)

    // 각 사용자에 대한 Negative 데이터(전체 데이터셋 - Positive 데이터)를 생성한다.
    // 전체 데이터 셋에서 Positive 데이터를 제외한 아이템 중 무작위로 선택한다.
    val negativeUserProducts = positiveUserProducts.groupByKey().mapPartitions {
      // 각 파티션에 대해서 수행한다.
      userIDAndPosItemIDs => {
        // 각 파티션 별로 난수 생성기를 초기화
        val random = new Random()
        val allItemIDs = bAllItemIDs.value
        
        userIDAndPosItemIDs.map { case (userID, posItemIDs) =>
          val posItemIDSet = posItemIDs.toSet
          val negative = new ArrayBuffer[Int]()
          var i = 0
          // Positive 아이템의 갯수를 벗어나지 않도록하는 범위 내에서
          // 모든 아이템 중 무작위로 아이템을 선택하여
          // Positive 아이템이 아니라면 Negative 아이템으로 간주한다.
          while (i < allItemIDs.size && negative.size < posItemIDSet.size) {
            val itemID = allItemIDs(random.nextInt(allItemIDs.size))
            if (!posItemIDSet.contains(itemID)) {
              negative += itemID
            }
            i += 1
          }
          // (사용자 아이디, Negative 아이템 아이디)의 쌍을 반환한다.
          negative.map(itemID => (userID, itemID))
        }
      }
    }.flatMap(t => t)
    // flatMap을 이용하여 묶여져 있는 셋을 하나의 큰 RDD로 쪼갠다.

    // Negative 아이템(아티스트)에 대한 예측치를 계산한다.
    val negativePredictions = predictFunction(negativeUserProducts).groupBy(_.user)

    // 각 사용자별로 Positive 아이템과 Negative 아이템을 Join 한다.
    positivePredictions.join(negativePredictions).values.map {
      case (positiveRatings, negativeRatings) =>
      	// AUC는 무작위로 선별된(처음에 10%를 무작위로 분리하였으므로) Positive 아이템의 Score가
      	// 무작위로 선별된(negativeUserProducts 를 구할 때 무작위로 선택하였으므로) Negative 아이템의 Score보다
      	// 높을 확률을 나타낸다. 이때, 모든 Postive 아이템과 Negative 아이템의 쌍을 비교하여 그 비율을 계산한다.

        var correct = 0L
        var total = 0L
        // 모든 Positive 아이템과 Negative 아이템의 쌍에 대해
        for (positive <- positiveRatings; negative <- negativeRatings) {
          // Positive 아이템의 예측치가 Negative 아이템의 예측치보다 높다면 옳은 추천 결과
          if (positive.rating > negative.rating) {
            correct += 1
          }
          total += 1
        }
        // 전체 쌍에서 옳은 추천 결과의 비율을 이용한 각 사용자별 AUC 계산
        correct.toDouble / total
    }.mean() // 전체 사용자의 AUC 평균을 계산하고 리턴한다.
  }
```

위 함수를 이용하여 다음과 같이 AUC를 계산할 수 있다. 함수의 동작 과정에 대한 설명은 코드에 포함되어 있는 주석으로 대신한다.

```scala
val auc = areaUnderCurve(cvData, bAllItemIDs, model.predict)
...
auc: Double = 0.9623184489901165
```

수행 결과는 조금 다를 수 있겠지만 거의 0.96에 가까운 수치가 나올 것이다. 이 수치는 무작위로 추천을 수행했을 때의 기대값인 0.5 보다 많이 높은 값이며, 최대값인 1.0에 매우 가까운 수치이다. 따라서 괜찮은 추천을 수행해 주었다고 할 수 있다.

이 과정을 전체 데이터 셋을 90%의 트레이닝 데이터와 나머지 10% 데이터로 구분하는 것부터 다시 수행함으로써 좀 더 최적화된 평가 수치를 얻을 수 있다. 실제로 전체 데이터 셋을 \\(k\\)개의 서브셋으로 분리하고, \\(k-1\\)개의 서브셋을 트레이닝 데이터로, 나머지 한 개의 서브셋을 평가용으로 사용하여 \\(k\\)번 반복하는 방식이 존재한다. 이것이 일반적으로 불리는 [K-fold Cross-validation](https://en.wikipedia.org/wiki/Cross-validation_(statistics)#k-fold_cross-validation) 방식이다.

앞서 계산한 결과가 어느정도의 결과를 갖는지 간단한 벤치마크 값을 계산하여 비교해 볼 수도 있다. 모든 사용자에게 가장 많이 플레이 된 아티스트를 똑같이 추천해 주는 것이다. 이런 추천은 개인화된 추천이 아니지만 간단하고, 빠른 방법이다. 이 경우의 AUC를 계산하여 앞서 계산한 결과와 어느정도 차이가 있는지 확인해 볼 수 있다.

다음과 같이 함수 `predictMostListened`함수를 정의하여 사용한다.

```scala
import org.apache.spark.SparkContext

def predictMostListened(sc: SparkContext, train: RDD[Rating])(allData: RDD[(Int, Int)]) = {
	val bListenCount = sc.broadcast(
		train.map(r => (r.product, r.rating)).reduceByKey(_ + _).collectAsMap()
	)
	allData.map { case (user, product) =>
		Rating(user, product, bListenCount.value.getOrElse(product, 0.0))
	}
}

val auc = areaUnderCurve(cvData, bAllItemIDs, predictMostListened(sc, trainData))
```

이 결과는 0.93 정도가 나온다. 앞서 우리가 추천 모델을 이용하여 수행한 추천의 결과가 더 높은 것을 알 수 있다. 하지만 좀 더 결과를 좋게 만들 수 없을까?
<br>
<br>

## Hyperparameter Selection

한 가지 간단한 방법은 추천 모델 형성에 사용된 몇 개의 [Hyperparameter](https://en.wikipedia.org/wiki/Hyperparameter)를 조절해보는 것이다. 지금까지의 추천 모델 형성 과정에서는 이 값에 대해 언급이 없었지만, 사용되었던 파라미터와 그 기본값은 다음과 같다.

#### rank = 10 
*rank* 파라미터는 *user-feature* 행렬과 *product-feature* 행렬을 구성할 때 column \\(k\\)의 크기를 의미한다.

#### iterations = 5
*iterations*는 Matrix Factorization 과정을 몇번 반복할 것인가에 대한 것이다. 횟수가 많아질 수록 추천의 성능은 좋아지지만, 수행 시간이 늘어난다.

#### lambda = 0.01
*Overfitting*을 막아주는 파라미터이다. 값이 높을수록 Overfitting 을 막아주지만, 너무 높다면 추천의 정확도를 저하시킨다.

#### alpha = 1.0
*Alpha*는 Implicit Feedback 방식에서 사용되는 파라미터로, user-product의 baseline confidence(값이 존재하는 데이터와 그렇지 않은 데이터 중 어떤것에 초점을 둘 것인지)를 조절하는 파라미터이다.


이 파라미터들을 조절하여 추천 모델의 성능을 증가시킬 수 있다. 파라미터를 조절하여 최적의 값을 찾는 방식에는 다양한 방법이 있지만, 여기선 간단하게만 변화를 주어 테스트를 할 것이다. 다음 코드와 같이 각 *rank*, *lambda*, *alpha* 에 두 개의 값으로 변화를 주어 그 결과로 계산되는 AUC를 비교할 것이다.

```scala
val evaluations =
	for(rank	<- Array(10, 50);
	    lambda	<- Array(1.0, 0.0001);
		alpha	<- Array(1.0, 40.0))
		yield {
		 val model = ALS.trainImplicit(trainData, rank, 10, lambda, alpha)
		 val auc = areaUnderCurve(cvData, bAllItemIDs, model.predict)
		 ((rank, lambda, alpha), auc)
	}

evaluations.sortBy(_._2).reverse.foreach(println)
```

수행 결과는 다음과 같다.

```scala
((10,1.0,40.0),0.9775933769125035)
((50,1.0,40.0),0.9775096131405069)
((10,1.0E-4,40.0),0.9767512207167729)
((50,1.0E-4,40.0),0.9761886422104153)
((10,1.0,1.0),0.9691674538720272)
((50,1.0,1.0),0.9670028532287775)
((10,1.0E-4,1.0),0.9648010615992904)
((50,1.0E-4,1.0),0.9545102924987607)
```

위 결과로 보아 rank는 10, lambda는 1.0, alpha를 40으로 하였을 때가 기본 설정으로 하였을 때보다 추천 성능이 좋음을 알 수 있다. 이런 방식으로 추천 모델을 최적화할 수 있다.

여기서 각 파라미터가 추천 결과에 어떤 영향을 미치는지 분석할 수 있다. *Alpha* 파라미터는 1일 때보다 40일때 추천의 성능이 증가되었다. 흥미로운 점은 이 40이라는 값이 지난 포스트에서 언급한 논문이 제안한 기본값이라는 것이다. 그리고 낮은 값인 1 보다 큰 값인 40일 때 성능이 좋은 것으로 보아 사용자가 특정 아티스트를 들었다는 정보가 듣지 않았다는 정보보다 추천 모델을 형성하는데에 있어 더욱 효과적이라는 것을 나타낸다.

*lambda*는 매우 적은 차이를 이끌어낸다. 하지만 높은 Lambda를 사용하였을 때 추천 성능이 더욱 좋은 것으로 보아 Overfitting을 효과적으로 방지하였음을 알 수 있다. Overfitting에 대해서는 다음 장에서 자세하게 살펴 볼 것이다.

column의 크기 \\(k\\)는 rank 파라미터의 값으로 보아 크게 중요하지 않음을 알 수 있다. 오히려 값이 50으로 클 때가 성능이 더 좋지 않았다. 따라서 너무 큰 \\(k\\)를 설정하게 되면 오히려 추천 성능이 감소함을 유추할 수 있다.

파라미터를 설정할 때 모든 파라미터에 대해 완벽하게 이해하고 있을 필요까지는 없다. 하지만 적어도 파라미터들이 어느 범위의 값을 갖는지 정도를 안다면, 여러 모델을 최적화하는데 많은 도움이 된다.
