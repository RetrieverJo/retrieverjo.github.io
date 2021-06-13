---
layout:     post
title:      "Spark & 머신 러닝 - Predicting Forest Cover"
subtitle:   "Advanced Analytics with Spark, 3. Predicting Forest Cover"
category:   Data Analysis
date:       2015-09-09 21:09:00
author:     "Hyunje"
tags: [Spark, Machine Learning, Decision Tree, Random Forest]
image:
  background: white.jpg
  feature: bg4.jpg
  credit: Hyunje Jo
keywords:	"데이터 분석, Data Analysis, Spark, Apache Spark, 머신 러닝, Machine Learning"
---


이 포스트는 Decision Tree를 이용해서 미국 콜로라도의 한 지역을 덮고 있는 숲의 종류를 예측하는 과정에 대한 글이다. 

이 포스트는 [Advanced Analytics with Spark](http://shop.oreilly.com/product/0636920035091.do)을 정리한 글이다.
<br>
<br>

## Regression and Classification

Regression은 숫자 형태의 데이터를 이용하여 또 다른 값을 예측하는 것이고, Classification은 레이블이나 카테고리등을 예측하는 것을 이야기 한다. 두 가지의 공통점은 주어진 하나 이상의 값을 이용하여 새로운 값을 예측해 낸다는 것이고 예측을 수행하기 위한 데이터와, 어떤 값을 예측할 것인지에 대한 'known answers'가 필요하다. 그렇기 때문에 Regression과 Classification은 Machine learning 에서 `supervised learning`에 속한다.

두 가지 모두 predictive analytics 에서 가장 오래되고 연구가 많이 된 방법이다. 머신러닝 관련된 라이브러리에서 대부분의 분석 알고리즘은 regression 혹은 classification 방법이다. 예를 들면 [Support Vector Machine](https://en.wikipedia.org/wiki/Support_vector_machine), [Logistic Regression](https://en.wikipedia.org/wiki/Logistic_regression), [Naive Bayes Classifier](https://en.wikipedia.org/wiki/Naive_Bayes_classifier), [Neural Network](https://en.wikipedia.org/wiki/Artificial_neural_network), [Deep Learning](https://en.wikipedia.org/wiki/Deep_learning) 등이 있다.

이 챕터(포스트)에서는 이러한 여러 알고리즘 중 가장 간단하면서도 확장성이 좋은 Decision Tree와, 그것의 확장형인 Random Decision Forest(Random Forest라고도 한다.)를 이용할 것이다. 

<br>
<br>

## Decision Trees and Forest

앞서 설명한 **Decision Tree**는 숫자 형태의 데이터 뿐만 아니라 categorical 한 Feature에 대해서도 사용 가능하다. 이 Decision Tree가 좀 더 일반화되어 좀 더 강력한 성능을 제공하는 것이 **Random Decision Forest**라고 불린다. 이 두 가지가 Spark의 MLlib에 `DecisionTree`와 `RandomForest`로 구현되어 있으며 이것을 데이터 셋에 적용하여 테스트 할 것이다.

Decision Tree 알고리즘은 직관적으로 이해하기 비교적 쉽다는 장점이 있다. 실제로 우리는 Decision Tree에 내재되어 있는 방식과 똑같은 방식을 실제로 사용하고 있기도 하다. 예를 들면 아침에 커피와 우유를 섞기 전에 우유가 상했는지 예측을 할 때 우리는 일련의 과정을 거친다. 먼저, 유통기한이 지났는가? 지나지 았았다면 상하지 않았을 것이고 날짜가 3일이상 지났다면 우유가 상했을 것이라 예측한다. 그렇지 않다면 우유에 냄새를 맡고 냄새가 이상하면 상했고, 그렇지 않다면 상하지 않았다고 판단할 것이다.

![Decision Tree Example] (https://db.tt/HxhHze1F)

Decision Tree에는 위 다이어그램과 같은 일련의 Yes/No 로 구성된 선택들이 내재되어 있다. 각각의 질문은 예측 결과 혹은 또 다른 결정문제를 유도한다. 이 Decision Tree의 각각의 노드는 결정문제가 되고, 각 leaf는 최종 예측 결과가 된다.

<br>
<br>


## Preparing the Data

이 장에서 사용할 데이터셋은 Covtype이라는 데이터셋으로 [이 링크](https://archive.ics.uci.edu/ml/machine-learning-databases/covtype/covtype.data.gz)에서 다운로드 할 수 있다. 다운로드 후 압축을 해제하면 생성되는**covtype.data** 파일이 데이터파일이다.

데이터셋은 미국 콜로라도주의 특정 지역을 덮고 있는 숲의 종류에 대해 기록이 정리되어있다. 각 행은 해당 지역에 존재하는 여러 부분들을 여러 개의 Feature로 표현한다. 고도, 경사도, 토양의 종류 등등이 Feature에 해당된다. 이렇게 표현되는 지역들을 덮고 있는 숲은 Feature로부터 예측 가능하며 총 54 가지의 Feature가 존재한다. 그리고 약 580000개의 데이터가 존재한다. 비록 데이터가 크지 않아 빅 데이터라 할 수는 없지만 우리가 Decision Tree를 생성하고 그 결과를 확인해 보는 데에는 적당하다.

데이터는 다음과 같은 내용으로 구성되어 있다.

![Covtype Dataset Description] (https://db.tt/q4uI4Ua7)

고맙게도 이 데이터는 CSV형태로 되어있기 때문에 Spark MLlib을 이용하기 위해 데이터를 정제하는 과정이 거의 필요하지 않다. **covtype.data**파일을 HDFS에 업로드할 때, 이 포스트에서는 `/covtype/covtype.data`로 업로드 하는 것을 가정한다.

Spark의 Mllib은 Feature의 Vector를 `LabeledPoint` 객체로 추상화하여 사용한다. Vector 형태로 저장되어 있는 Feature 값과 구하고자 하는 target인 `Label`로 구성된 형태이다. LabeledPoint는 숫자 형태의 값만을 이용하는 것으로 제한되며 target은 Double 형태로 지정된다. 때문에 만약 Categorical한 Feature(숫자가 아닌 값들)를 사용하고자 할 때는 적절한 값으로 인코딩을 해 주어야 한다.

한 가지 인코딩 방법은 \\(N\\)개의 값은 갖는 Categorical한 Feature를 \\(N\\)개의 0과 1의 조합을 통해 서로 다른 값을 갖게하는 것이다. 예를 들어 날씨에 대한 Feature가 갖는 값의 종류는 **cloudy**, **rainy**, **clear** 일 때, cloudy는 `1,0,0`으로, rainy는 `0,1,0`과 같은 값은 갖는 형태로 인코딩을 할 수 있다. 이 포스트에서 사용하는 데이터 셋은 이 방법으로 구성되어있다. 다른 방법으로는 각각의 값에 특정 숫자(1, 2, ...)를 부여하는 방법이 있다. 이 챕터의 뒷 부분에서는 전자의 방식을 후자의 방식으로 바꿔서도 실험을 수행할 것이다.

HDFS에 업로드한 데이터를 MLlib에서 사용하기 위해 다음과 같이 데이터를 준비한다.

```scala
import org.apache.spark.mllib.linalg._
import org.apache.spark.mllib.regression._

val rawData = sc.textFile("/covtype/covtype.data")

val data = rawData.map { line =>
	val values = line.split(",").map(_.toDouble)
	val featureVector = Vectors.dense(values.init)
	val label = values.last - 1
	LabeledPoint(label, featureVector)
}
```

데이터의 구성을 보면 알 수 있듯이 각 행의 마지막 값이 우리가 최종적으로 알아내고자 하는 숲의 종류임을 알 수 있다. 따라서 위 코드에서처럼 `featrueVector`를 생성할 때 init()을 이용해 **가장 마지막 값은 제외한 벡터를 feature vector로** 한다. 그리고 MLlib에서의 Decition Tree의 Label은 값이 0 부터 시작하므로 데이터의 가장 마지막 값은 1을 뺀다.

또한 다음 코드를 이용해 이전 포스트에서 수행하였던 것처럼 모든 데이터를 Decition Tree를 생성하는데 사용하지 않고, 80% 데이터를 트레이닝에 사용하고, 10%를 Cross-validation, 나머지 10%를 테스트용으로 사용할 것이다.

```scala
val Array(trainData, cvData, testData) = data.randomSplit(Array(0.8, 0.1, 0.1))
trainData.cache()
cvData.cache()
testData.cache()
```

<br>
<br>

## A First Decision Tree

다음 코드를 이용해 Decision Tree를 트레이닝시킨다.

```scala
import org.apache.spark.mllib.tree._

val model = DecisionTree.trainClassifier(trainData, 7, Map[Int, Int](), "gini", 4, 100)
```

ALS를 이용한 추천 모델 생성에서도 그러하였듯이, `trainClassifier` 함수를 이용해 트레이닝을 할 때에도 몇 가지 필수 Hyperparameter가 사용된다. 첫번째 파라미터는 **어떤 데이터를 이용하여 Classifer를 훈련시킬 것인지에 대한 것**이고, 두 번째 파라미터는 **우리가 최종적으로 예측하고자 하는 값의 종류가 몇가지 인지**를 나타낸다. Covtype Dataset에서 우리가 예측하고자 하는 숲의 종류는 7가지 이므로 7이 파라미터로 이용된다. 다음 ```Map```인자는 categorical feature에 대한 정보를 갖고 있는데, 이것은 나중에 *gini*파라미터의 의미와 함께 설명할 것이다. 그리고 다음 인자는 4로 **Tree의 최대 깊이**를 의미하며 마지막 100 파라미터는 **Feature가 연속한 값을 가질 때 최대 몇 개의 bin 까지 나눌 수 있는가**에 대한 것이다.

생성한 Decision Tree 모델이 얼마만큼의 정확도를 갖는지는 다음과 같은 코드를 이용해 [Confusion Matrix](https://en.wikipedia.org/wiki/Confusion_matrix)를 계산함으로써 알 수 있다. 이 계산에서는 Cross-Validation 데이터를 이용하였다.

```scala
import org.apache.spark.mllib.evaluation._
import org.apache.spark.mllib.tree.model._
import org.apache.spark.rdd._

def getMetrics(model: DecisionTreeModel, data: RDD[LabeledPoint]): MulticlassMetrics = {
	val predictionsAndLabels = data.map( example =>
		(model.predict(example.features), example.label)
	)
	new MulticlassMetrics(predictionsAndLabels)
}

val matrics = getMetrics(model, cvData)
matrics.confusionMatrix
```

Confusion Matrix는 다음과 같다. (랜덤성을 갖고 있기 때문에 수행 결과가 다를 수 있다.)

```
15767.0  5223.0   3.0     0.0  0.0   0.0  380.0
6761.0   21073.0  447.0   0.0  8.0   0.0  31.0
0.0      714.0    2836.0  0.0  0.0   0.0  0.0
0.0      0.0      292.0   0.0  0.0   0.0  0.0
16.0     865.0    24.0    0.0  16.0  0.0  0.0
0.0      428.0    1287.0  0.0  0.0   0.0  0.0
1134.0   20.0     0.0     0.0  0.0   0.0  914.0
```

우리가 예측하고자 하는 목표 값이 7가지 값을 갖기 때문에 Confusion Matrix는 7 X 7 의 행렬을 갖는다. 또한 위 행렬의 **각 행은 실제 정답을 의미하고, 각 열은 Decision Tree가 예측한 값을 의미**한다. 따라서 행 \\(i\\), 열 \\(j\\)의 숫자는 Decision Tree가 \\(j\\)로 예측하였고 실제 그 값이 \\(i\\)인 개수를 나타낸다. 때문에 Decision Tree가 정확히 맞춘 것은 diagonal(행과 열의 인덱스가 같은 것)에 존재하는 것들이며 그것을 제외한 나머지들은 에러이다. 위 결과에서는 category 3과 5와 같은 경우에는 제대로 맞춘 것이 하나도 없는 결과를 보인다.

다음 코드를 수행하면 간단히 정확도를 계산할 수 있다.

```scala
metrics.precision

...

Double = 0.6972303782688576
```

약 0.7의 값을 보이는데 이 값은 실제 [Precision](https://en.wikipedia.org/wiki/Precision_and_recall#Definition_.28classification_context.29)라는 값을 계산한 것이다. 일반적으로 정확도라고 불리기도 하지만 *binary classification*에서 주로 사용되는 성능 측정 방법이다. `positive`와 `negative` 두 클래스로 데이터를 분류하는 문제에서 **Precision은 Classifier가 Positive라고 판단한 것 중에 실제 Poistive가 차지하는 비율**이다. 보통 이 Precision은 `Recall`과 함께 언급된다. **Recall은 실제 Positive 값 중에 Classifier가 Positive라 판단한 것의 비율**이다.

예를 들면 50개의 데이터 중에 20개가 Positive인 데이터셋이 있고, Classifier가 50개 중에 10개를 Positive라 판단하였는데 그 10개 중에 4개가 실제 Positive(제대로 Classify 한것을 의미) 데이터라 한다면 Precision은 \\( \frac { 4 }{ 10 } = 0.4\\) 이며, Recall은 \\( \frac { 4 }{ 20 } = 0.2\\) 이다. 이것을 Multi-class classification에 적용하여 각각의 카테고리를 Positive라 가정하고 각 수행에서 나머지 카테고리를 Negative라 하여 Precision Recall을 계산할 수 있다. 이런 계산 결과들이 위에서 생성한  `MulticlassMatrics`클래스에 들어있다. 각 카테고리별로 Recall과 Precision이 얼마나 되는지 다음 코드를 통해 확인할 수 있다.

```scala
(0 until 7).map( //Category 는 0부터 시작하기때문에
	category => (metrics.precision(category), metrics.recall(category))
).foreach(println)
```

수행 결과는 다음과 같다. (수행시마다 다를 수 있다.)

```
(0.665892389559929,0.7377064520656904)
(0.7440242912120891,0.7441031073446328)
(0.5800777255062385,0.7988732394366197)
(0.0,0.0)
(0.6666666666666666,0.01737242128121607)
(0.0,0.0)
(0.689811320754717,0.44197292069632493)
```

처음 계산한 정확도인 0.7은 꽤나 괜찮은 결과처럼 보이지만 0.7이라는 정확도가 얼마나 정확한 것인지 판단하기 어렵다. 따라서 그것을 구분하기 위한 기준선이 필요하다. 다음과 같은 함수를 이용하여 인자로 받는 데이터에서 Classifier가 무작위로 클래스를 선택한다고 가정후에 그 것에 대한 정확도를 계산할 수 있다.

```scala
def classProbabilities(data: RDD[LabeledPoint]) : Array[Double] = {
	val countsByCategory = data.map(_.label).countByValue()
	val counts = countsByCategory.toArray.sortBy(_._1).map(_._2)
	counts.map(_.toDouble / counts.sum)
}
```

Classifier가 무작위로 클래스를 선택하기 때문에 각 데이터에 대해서 Classifier가 정확한 클래스를 맞출 할 확률은 데이터에서 각 클래스가 차지하는 비율과 같다. 위 함수를 이용하여 다음과 같이 트레이닝 데이터와 Cross-validation 데이터에서 무작위로 클래스를 선택하는 Classifier의 정확도를 계산할 수 있다.

```scala
val trainPriorProbabilities = classProbabilities(trainData)
val cvPriorProbabilities = classProbabilities(cvData)
trainPriorProbabilities.zip(cvPriorProbabilities).map {
	case (trainProb, cvProb) => trainProb * cvProb
}.sum

...

Double = 0.37742100814227625
```

위 결과를 통해 Random Gessing 은 37%의 정확도를 가짐을 알 수 있다. 때문에 우리가 앞서 구한 70%의 정확도는 괜찮은 결과임을 알 수 있다. 하지만 이것은 `Tree.trainClassifier()`의 기본 파라미터만을 이용하여 계산한 것이기 때문에 Hyperparameter를 조절하여 더욱 성능을 끌어올릴 수 있다.


<br>
<br>

## Decision Tree Hyperparameters

앞서 설명하였듯이 Decision Tree에는 세 가지의 Hyperparameter로 *최대 깊이*, *최대 bin의 수*, *impurity measure*가 존재한다.

Decision Tree의 최대 깊이는 Tree의 높이를 이야기한다. Tree의 깊이가 깊다는 것은 데이터를 분류하는데 까지의 과정이 세밀함을 의미하며, 깊이가 깊을 때는 좀 더 정확하게 분류를 할 수 있지만 [Overfitting](http://sanghyukchun.github.io/59/)의 위험성이 있다.

Decision Tree는 각각의 질의 단계에서 각 단계에 맞는 decision을 내려야 한다. 이때 결정을 내리기 위한 질문들은 숫자 형태의 Feature인 경우에는 `feature >= value`와 같은 형태이며, Categorical한 Feature인 경우에는 `feature in (value1, value2, ...)`형태를 갖는다. 이러한 decision rule의 셋이 각 단계에서 decision을 내리는 데에 사용된다. 이것들을 Spark MLlib에서는 `bin`이라 부른다. 사용되는 bin의 수가 클수록 계산 시간은 길어지지만 좀 더 최적화된 결과를 낼 수 있다.

Decision Tree를 형성하는데에 있어 좋은 Decision Rule이라 함은 데이터를 명확하게 구분짓는 Rule 일것이다. 예를 들면 어떤 Rule이 Covtype Dataset을 정확히 클래스 1~3과 4~7로 나눈다면 그 Rule은 매우 좋은 Rule이라 할 수 있다. 결론적으로 좋은 Rule을 고른다는 것은 **데이터를 두 개의 서브셋으로 나눌 때, 그 사이의 불순물이 적도록 선택하는 것**이다. 때문에 어떤 방식으로 불순물이 많은지 판단할지가 매우 중요한데, 일반적으로 많이 사용하는 방식은 `Gini`방식과 `Entropy`방식이다.

Gini 방식은 앞서 설명하였던 무작위로 클래스의 레이블을 선택하는 과정과 비슷하다. Gini impurity(불순도) 측정 방식은 어떤 한 클래스를 골라서 무작위로 라벨을 추정하였을 때, 그 추정이 틀릴 확률을 이용하여 계산된다. 만약 이 불순도가 0이 된다면 현재 데이터는 완벽히 한 클래스만 존재하여 정확히 분류가 된 것을 의미한다. 그리고 데이터의 모든 클래스가 같은 비율로 존재할 때 불순도가 제일 높다. 이러한 Gini 불순도 \\(I\_G\\) 는 \\( i = 1, 2, ..., N \\) 클래스와, 각 클래스가 데이터 중에서 차지하는 비중인 \\(p\_i\\)를 이용해 다음과 같이 표현 가능하다.

\\( I\_G(p) = \sum \_{ i=1 }^{ N }{ { p }\_{ i }(1-{ p }\_{ i }) }  = 1 - \sum \_{ i=1 }^{ N }{ { { p }\_{ i } }^{ 2 } } \\) 

Entropy 방식은 Information Theory에서 나온 방법이다. 식이 유도되는 방식을 이야기 하기에는 어렵지만 **목표하는 클래스의 서브셋이 얼마만큼의 불확실성을 갖는가에 대한 것**이며, 다음과 같이 정의된다.

\\( { I }\_{ E }(p)=\sum \_{ i=1 }^{ N }{ { p }\_{ i }\log { \frac { 1 }{ p }  }  } = -\sum \_{ i=1 }^{ N }{ { p }\_{ i }log({ p }\_{ i }) } \\)

데이터셋에 따라 어느 측정 방식이 좋은지는 다르며, Spark의 MLlib 에서는 `Gini`방식이 기본값이다. 몇몇의 Decision Tree는 `Minimum Information Gain`이라는 방식을 이용하는데 아직 이 방식은 MLlib에 구현되어 있지 않다.


<br>
<br>

## Tuning Decision Trees

위에서 설명한 파라미터를 조절하여 처음 생성한 Decision Tree 모델보다 성능을 개선시킬 것이다. 영화 추천 모델에서와 마찬가지로 다음 코드와 같이 간단한 형태의 파라미터 조절 실험이 가능하다.

```scala

val evaluations = 
	for(impurity <- Array("gini", "entropy");
		depth <- Array(1, 20, 30);
		bins <- Array(10, 300))
	yield {
		val model =
			DecisionTree.trainClassifier(trainData, 7, Map[Int, Int](), impurity, depth, bins)
		val predictionsAndLabels = cvData.map(example =>
			(model.predict(example.features), example.label)
		)
		val accuracy = new MulticlassMetrics(predictionsAndLabels).precision
		((impurity, depth, bins), accuracy)
	}

evaluations.sortBy(_._2).reverse.foreach(println)
```

위 코드의 수행 결과는 다음과 같다. 물론 세부적인 수치는 다를 수 있다.

```
((entropy,30,300),0.938073512955152)
((gini,30,300),0.9331152621158647)
((entropy,30,10),0.9135060686924334)
((gini,30,10),0.9122320736851166)
((entropy,20,300),0.9077386588620125)
((gini,20,300),0.9051734526986314)
((entropy,20,10),0.895394680210037)
((gini,20,10),0.8903675647757596)
((gini,1,300),0.6380304725832832)
((gini,1,10),0.6375484204183525)
((entropy,1,300),0.48788843935611603)
((entropy,1,10),0.48788843935611603)
```

이 결과로부터 Decision Tree의 깊이를 1로 하는 것은 결과를 생성하는데 부족하다는 것을 알 수 있다. 또한, bin의 수는 높을 수록 좋은 정확도를 보이는 경향을 보였다. 그리고 불순도를 판단하는 방법은 Entropy 방식이 조금 더 좋은 결과를 보였다. 결론적으로 **Entropy방법**, **최대 깊이 30**, **300개의 bin**을 이용하여 Decition Tree를 생성하였을 때 가장 높은 성능인 **93.8%**의 성능을 보였다.

여기서 우리는 찾아낸 파라미터들이 Overfitting 된 것이 아닌지 판단을 해 보아야 한다. 이것은 다음과 같이 위에서 구한 파라미터를 이용하여 트레이닝 데이터와 Cross-Validation 데이터를 이용하여 Decision Tree를 생성하였을 때의 성능을 확인함으로써 판단 가능하다.

```scala
val model = DecisionTree.trainClassifier(
	trainData.union(cvData), 7, Map[Int, Int](), "entropy", 30, 300)
val accuracy = getMetrics(model, testData).precision

...

accuracy: Double = 0.9427713442521098
```

위 코드의 수행 결과는 94.2%의 정확도를 보인다. 만약 앞서 얻은 파라미터가 Overfitting된 파라미터였다면 Cross-Validation데이터를 union한 데이터에 대한 Decision Tree의 정확도는 더 떨어졌을 것이다. 하지만 앞서 수행하였던 결과보다 데이터를 추가하여 Decision Tree 트레이닝 하였고, 성능이 증가하였다. 따라서 앞에서 얻은 파라미터가 트레이닝 데이터에 Overfitting되지 않은 파라미터임을 알 수 있다.

<br>
<br>

## Categorical Features Revisited

지금까지 작성한 예제 코드에서는 `Map[Int, Int]()` 파라미터에 대한 설명을 거의 하지 않고 진행하였다. 이 파라미터는 7과 같이 각각의 Categorical한 Feature의 값의 가지 수에 대한 것이다. Map의 key는 **입력 벡터의 인덱스**를 나타내며, value들은 **벡터의 각 인덱스에 존재하는 Categorcal한 값의 가지 수**를 나타낸다. 때문에 비어있는 `Map()`이 파라미터로 전달되었을 때는 **Categorical한 값이 없음을 나타내며, 모든 값이 숫자 형태의 데이터임**을 의미한다.

다행히도 지금까지 이용한 covtype 데이터셋은 Categorical Feature를 여러 개의 Numeric Feature를 이용하여 표현하고 있다. 이때, 0과 1을 이용한 binary 형태의 값 표현이기 때문에 Decision Tree를 구성할 때 큰 문제가 생기지는 않는다. 하지만 당연히도, 이러한 형태의 Categorical Feature에 대한 표현은 Decision Tree를 구성할 때 하나의 Categorical Feature를 구분하기 위해 그 Feature를 표현하는데 사용되는 모든 Numeric Feature를 고려하여 Decision Rule을 구성하게된다. 이때, 메모리 사용량도 증가할 것이며 속도 역시 감소할 것이다.

이러한 것을 피하는 방법은 다음과 같이 여러 개의 Numeric Feature로 구성되어 있는 하나의 Categorical Feature를 통합하여 재구성하는 것이다.

```scala
val data = rawData.map { line =>
	val values = line.split(',').map(_.toDouble)
	val wilderness = values.slice(10, 14).indexOf(1.0).toDouble
	val soil = values.slice(14, 54).indexOf(1.0).toDouble
	val featureVector = Vectors.dense(values.slice(0, 10) :+ wilderness :+ soil)
	val label = values.last - 1
	LabeledPoint(label, featureVector)
}
```

위처럼 기존에 Categorical Feature를 여러 개의 Numeric Feature로 표현하던 것을 하나의 Feature로 변경한다. 그리고 다음의 코드에서처럼 DecisionTree를 생성할 때 Map()에 해당 Feature의 인덱스와 그 값의 가지 수를 함께 전달하여 Categorical Feature를 그대로 활용하도록 한다. 다만, bin의 수는 반드시 Categorical Feature의 가지 수 보다 커야 하기 때문에 bind르 40이상으로 설정하여 수행한다.


```scala
val Array(trainData, cvData, testData) = data.randomSplit(Array(0.8, 0.1, 0.1))
trainData.cache()
cvData.cache()
testData.cache()

val evaluations = 
	for(impurity <- Array("gini", "entropy");
		depth <- Array(1, 20, 30);
		bins <- Array(40, 300))
	yield {
		println((impurity, depth, bins))
		val model =
			DecisionTree.trainClassifier(trainData, 7, Map(10 -> 4, 11 -> 40), impurity, depth, bins)
		val predictionsAndLabels = cvData.map(example =>
			(model.predict(example.features), example.label)
		)
		val trainAccuracy = getMetrics(model, trainData).precision
		val cvAccuracy = getMetrics(model, cvData).precision
		((impurity, depth, bins), (trainAccuracy, cvAccuracy))
	}

evaluations.sortBy(_._2._2).reverse.foreach(println)
```

위 예제의 수행 결과는 다음과 같다.

```
((entropy,30,300),(0.9997400744976563,0.9424945022597011))
((entropy,30,40),(0.9995381489007944,0.9389967273293969))
((gini,30,300),(0.9997142967618867,0.9377153642361171))
((gini,30,40),(0.9996197783973981,0.9343734307631036))
((entropy,20,300),(0.9675436825214062,0.9258194663295873))
((gini,20,40),(0.9666049433104628,0.9238974216896677))
((gini,20,300),(0.9692987166983876,0.9238627902547142))
((entropy,20,40),(0.966656498782002,0.9227892157711555))
((gini,1,300),(0.6335093379847826,0.6374261917542553))
((gini,1,40),(0.6328541538673048,0.6369413516649063))
((entropy,1,300),(0.4876546127110015,0.4874201312531385))
((entropy,1,40),(0.4876546127110015,0.4874201312531385))
```

위 결과는 트레이닝 셋에 대한 수행 결과는 Overfitting 된 결과를 보이지만 Cross-Validation 데이터를 이용해 정확도를 판별하였을 때에는 데이터를 변환하지 않았을 때의 성능과 비슷한 성능을 보였다. 또한 다음과 같이 트레이닝 셋에 Cross-Validation 데이터를 포함한 것을 이용해 Decision Tree를 훈련시켜 그 결과도 확인하였다. 그 결과 역시 변경 전의 결과와 비슷한 성능을 보였다. 하지만 두 경우 모두 약간의 성능 증가는 존재하였다.


```scala
val model = DecisionTree.trainClassifier(trainData.union(cvData), 7, Map(10 -> 4, 11 -> 40), "entropy", 30, 300)
val accuracy = getMetrics(model, testData).precision
...

accuracy: Double = 0.9450667034844589
```
_(실제 책에서는 3% 차이로 더 큰 차이가 발생하였으나, 이 글 작성시 수행한 실험에서는 큰 차이가 나지 않았다.)_


<br>
<br>

## Random Decision Forests


만약 위 코드를 그대로 따라왔다면, 직접 수행한 결과와 이 포스트에 있는 수행 결과가 약간 다를 것이다. 그 이유는 Decision Tree가 가지고 있는 랜덤성 때문이다. Decision Tree 알고리즘이 각 단계에서 가능한 모든 경우의 수를 고려하게되면 수많은 시간이 필요하기 때문에 그렇게 할 수 없다. 만약 하나의 Categorical Feature가 \\(N\\)가지의 경우를 갖는다면 \\({ 2 }^{ N } - 2\\)의 Decision Rule이 존재한다. 따라서 \\(N\\)이 커진다면 가능한 Decision Rule의 수 역시 기하급수적으로 증가하게 될 것이다.

그래서 모든 경우의 수를 확인하는 방법 대신에, 몇몇 휴리스틱한 방법을 이용하여 성능을 증가시킨다. 그 중 하나는 `배깅(Bagging)`이라는 방식인데, **한 개의 Decision Tree가 아닌 여러 독립적인 Decision Tree를 이용하여 각 Tree가 특정 결과를 생성하고, 그 각 결과의 평균치 등을 이용하여 예측치를 결정하는 것**이다. 단순히 이야기 하면 Voting 방식이다. 이러한 방식은 단순히 하나의 Decision Tree만으로 결과를 내는 것보다 정확한 결과를 낼 확률이 높으며, 이때 여러 독립적인 Decision Tree를 생성하는 과정에서 랜덤성(Randomness)이 활용되고, 이것이 `Random Decision Tree`의 주요 아이디어가 된다.

다음과 같이 Spark MLlib의 `RandomForest`를 이용하여 쉽게 Random Decision Forest를 생성할 수 있다.

```scala
val forest = RandomForest.trainClassifier(
	trainData, 7, Map(10 -> 4, 11 -> 40), 20, "auto", "entropy", 30, 300)
```

Random Decision Forest는 새로운 두 개의 파라미터가 등장한다. 첫 번째 파라미터는 몇 개의 Decision Tree를 생성할 것인지에 대한 파라미터이며, 본 예시에서는 20으로 정하였다. 한 개의 Decision Tree를 생성하는 것이 아니라 20개의 Tree를 생성하기 때문에 지금까지 수행하였던 예시들보다 수행 시간이 오래 걸린다. 두 번째 파라미터는 트리의 각 레벨에서 어떤 feature를 선택할지에 대해 평가를 하는 방식인데, 여기서는 "auto"를 이용하였다. Random Decision Forest에서는 모든 feature를 전부 고려하지 않고 전체 중 일부만 선택하여 활용하는데, 어떤 방식으로 일부를 선택할 것인지에 대한 것이다. 예측 과정은 단순히 Decision Tree들의 가중평균으로 계산된다. Categorcal Feature는 결과 중 가장 많이 나온 값으로 선택하는 방식을 따른다.

다음과 같이 생성한 Random Decision Forest의 성능을 평가할수 있다.

```scala
def getForestMetrics(model: RandomForestModel, data: RDD[LabeledPoint]): MulticlassMetrics = {
	val predictionsAndLabels = data.map {
		example => (model.predict(example.features), example.label)
	}
	new MulticlassMetrics(predictionsAndLabels)
}

val accuracy = getForestMetrics(forest, testData).precision

...

accuracy: Double = 0.9632742522824155
```

트레이닝 데이터로만 생성한 Random Decision Forest의 정확성은 96.3%로 지금까지 생성하였던 단일 Decision Tree보다 성능이 높음을 알 수 있다.

Random Decision Forest는 Spark와 MapReduce와 같은 빅 데이터의 데이터 처리 방식과도 연관성이 있다. Random Decision Forest를 생성할 때는 각각의 Tree를 독립적으로 생성하는데, 이것은 언급한 빅데이터 기술들은 데이터를 독립적으로 처리하는 매커니즘을 포함하고 있기 때문이다.

<br>
<br>

## Making Predictions

Decision Tree와 Random Decision Forest를 생성하는 과정이 흥미로웠지만, 이것은 완전한 목표가 아니다. 최종 목표는 주어진 벡터를 생성한 Model을 이용해 결과를 예측하는 것이다. 근데, 지금까지의 과정을 제대로 따라왔다면 정말로 쉽다. 이미 앞선 과정에서 `getMetrics` 함수와 `getForestMetrics` 함수 안에서 해당 과정을 수행하고 있기 때문이다.

DecisionTree와 RandomForest의 훈련 결과는 각각 DecisionTreeModel과 RandomForestModel 객체인데, 두 객체는 모두 `predict()`함수를 포함하고 있다. 이 함수는 벡터를 인자로 받아 해당 데이터에 맞는 예측 결과를 반환하는 함수이다. 따라서 다음과 같이 특정 벡터에 대해 그 결과를 예측할 수 있다.

```scala
val input = "2709,125,28,67,23,3224,253,207,61,6094,0,29"
val vector = Vectors.dense(input.split(',').map(_.toDouble))
forest.predict(vector)

...

Double = 4.0
```

이 결과는 4가 나와야 하는데, 이것은 위 벡터 **"2709,125,28,67,23,3224,253,207,61,6094,0,29"**의 예측 결과가 숲의 종류 `5`임을 뜻한다(원래의 데이터는 클래스가 1 부터 시작하고, 트레이닝에서 활용한 데이터는 0부터 시작하는 것으로 변경하였으므로). 이것은 Covtype 데이터 셋에서 "Aspen" 임을 뜻한다.

위 예시에서는 단순히 하나의 벡터에 대해서만 수행했지만, RDD로 구성된 여러 개의 벡터를 한번에 예측할 수도 있다.