---
layout:     post
title:      "Spark & 머신 러닝 - Anomaly Detection"
subtitle:   "Advanced Analytics with Spark, 4. Anomaly Detection in Network Traffic"
category:   Data Analysis
date:       2015-10-24 17:15:00
author:     "Hyunje"
tags: [Spark, Machine Learning, Clustering, K-means, K-means++]
keywords:	"데이터 분석, Data Analysis, Spark, Apache Spark, 머신 러닝, Machine Learning"
usemathjax: true
---

이 포스트는 K-means Clustering을 이용하여 네트워크 트래픽에서의 비정상 트래픽을 감지해 내는 과정에 대한 내용을 담고 있다. 이 장에서 수행한 결과는 수행시마다 바뀌기 때문에, 수행 결과가 이 문서에서 제시하는 결과와 완벽히 일치하지 않는다.

이 포스트는 [Advanced Analytics with Spark](http://shop.oreilly.com/product/0636920035091.do)을 정리한 글이다.
<br>
<br>

$$E=mc^2$$

## Unsupervised Learning and Anomaly Detection

앞선 챕터에서 살펴본 Classification과 Regression은  강력하기도 하며 많은 연구가 진행된 머신 러닝 테크닉중 하나이다. 하지만 Decision Tree나, Random Dicision Forest등을 이용하여 새롭게 들어온 데이터를 이용해 어떤 값을 예측을 할 때는 그 값이 이전 훈련 과정에서 이미 알고 있는 값 중 하나여야만 했다. 그리고 이러한 과정이 Supervised Learning의 일반적인 방식이라 설명하였다.

하지만 다음과 같은 경우가 있을 수 있다. 어떤 인터넷 쇼핑몰 사이트의 회원을 그들의 쇼핑 습관과 관심 상품등을 기준으로 구분하고자 한다. 이 때의 입력 Feature는 사용자들의 구매 목록, 클릭한 내용들, 통계학적 데이터와 같은 것들이 될 수 있다. 이러한 입력 데이터들의 결과로 아웃풋은 사용자들의 그룹이 되어야 하는데, 예를 들면 한 그룹은 패션에 민감한 사람들의 그룹 또 다른 그룹은 가격에 민감한 사람들의 그룹과 같은 형태를 나타내어야 한다.

만약 Decision Tree와 같은 Supervised Learning 테크닉만 사용하여 위 문제를 해결하려 한다면 새로운 데이터를 각각 Classifier에 적용시켜야 하는데  어떤 데이터가 어떤 값을 가져야 하는지에 대한 사전정보가 전혀 없기 때문에 불가능에 가깝다. 떄문에 이런 문제를 해결할 때에는 Supervised Learning이 아닌 [Unsupervised Learning](https://en.wikipedia.org/wiki/Unsupervised_learning)을 이용하여 해결해야 한다. Unsupervised Learning은 Supervised Learning과 다르게 어떤 값을 예측해야 한다고 하여 사전에 그 값이 어떤 것인지 트레이닝 시킬 필요가 없다. 왜냐하면 Unsupervised Learning에 속하는 방법들은 **주어진 데이터들 사이에서 비슷한 것들끼리 그룹을 만들고, 새로운 데이터가 어떤 그룹에 속하는지 판단하는 역할을 하기 때문이다.**

Anomaly Detection은 이름에서도 알 수 있듯이 비정상적인 것을 찾는 것을 의미한다. Anomaly Detection은 네트워크 공격 혹은 서버에서의 문제 발생등을 검출하는데에 이용된다. 여기서 중요한 것은 새로운 형태의 공격이라던가 그동안 발생하지 않았던 서버 문제등을 발견할 수 있다는 것이다. Unsupervised Learning은 이 경우에 일반적인 형태의 데이터를 이용해 훈련하고, 기존에 있던 것과 다른 것이 입력으로 들어왔을 때 그것을 감지하는 형태로 Anomaly Detection 문제를 해결할 수 있다.

<br>
<br>

## K-means Clustering

클러스터링 방법은 Unsupervised Learning 중에서 가장 잘 알려진 방법 중 하나이다. 클러스터링은 **주어진 데이터를 이용하여 가장 자연스러운 그룹을 찾아내는 것**을 시도하는 알고리즘이다. K-means Clustering은 이러한 클러스터링 알고리즘 중에 가장 널리 알려진 알고리즘이다. 데이터셋에서 \\(k\\)개의 클러스터를 찾는데 이 때 \\(k\\)는 데이터를 분석하는 사람으로부터 주어진 것이다. 또한 이 \\(k\\)는 Hyperparameter이며, 각각의 데이터마다 최적 값이 다르다. 실제로 이 챕터에서 적절한 \\(k\\)값을 찾는 과정이 중요한 부분이 될 것이다.

사용자들의 행동에 대한 데이터 혹은 거래 기록에 대한 데이터에서 `비슷하다`라는 것은  어떤 것인가? 이러한 질문에 대답하기 위하여 K-means Clustering 방식은 데이터 사이의 거리에 대한 정의를 필요로한다. 가장 간단한 방법중 하나는 Spark MLlib에도 구현되어 있는 Eculidean Distance를 이용하는 것이다. 이것을 이용하면 `비슷한`데이터는 거리가 가깝게 나올 것이다.

K-means Clustering에서의 각 클러스터는 하나의 데이터 포인트로 표현된다. 각 클러스터에 속하는 데이터 포인트들의 평균값이 그 클러스터의 중심이 되며, 각 클러스터의 중심을 평균으로 구하기 때문에 K-means라는 이름이 붙었다. 이 때의 가정은 각 Feature의 값들이 숫자 형태의 값임을 가정으로 하며, 각 클러스터의 중심은 **centroid**라고 부른다. K-means Clustering의 동작 과정은 매우 간단하다. 제일 먼저 알고리즘은 \\(k\\)개의 데이터를 선택함으로써 각 클러스터의 centroid를 초기화한다. 그리고 각 데이터는 가장 가까운 centroid의 클러스터로 할당된다. 그리고 각각의 클러스터별로 새로운 데이터의 평균을 구해 새로운 centorid로 지정한다. 이 과정이 반복된다.

<br>
<br>

## Network Intrusion

사이버 공격이라는 형태의 해킹이 뉴스에서도 심심치 않게 등장하고있다. 몇몇 공격들은 네트워크 트래픽을 점령하여 정상적인 트래픽을 밀어내기도 한다. 하지만 네트워킹 소프트웨어의 결점 등을 이용해 컴퓨터에 대한 비정상적인 권한을 탈취하는 공격도 존재한다. 이 때 공격받는 컴퓨터에서는 공격받는지 알아채기가 매우 어렵다. 이러한 공격([Exploit](https://en.wikipedia.org/wiki/Exploit_(computer_security)))을 찾아내기 위해선 매우 많은 네트워크 요청들 사이에 비정상적인 공격을 찾아내야 하기 때문이다.

몇몇 공격들은 특정 알려진 패턴을 따른다. 예를 들면 가능한 모든 포트를 빠른 시간안에 접근하는데, 이것은 일반 소프트웨어들은 일반적으로 하지 않는 패턴이다. 하지만 이것은 일반적으로 Expolit을 하기 위한 컴퓨터를 찾는 공격자들이 일반적으로 제일 첫단계로 수행하는 과정이다.

만약 짧은 시간 내애 몇개의 포트에 대해 접속 시도가 발생한다면 몇 개의 접속 시도는 일반적인 것으로 간주 될 수 있지만 대부분의 시도들은 비정상 적인 것일 것이므로 우리는 이것을 port-scanning 공격이 들어온 것으로 판단할 수 있다. 이러한 이미 알려진 형태의 공격들은 미리 알고 감지할 수 있다. 하지만 알려져 있지 않은 형태의 공격이 들어온다면 어떻게 해야될까? 가장 큰 문제점은 어떤 형태의 공격일지 모른다는 것이다. 그동안과의 다른 형태의 접근 시도를 잠재적인 공격 트래픽으로 간주하여 감시해야 할 것이다.

여기에서 Unsupervised Learning이 사용될 수 있다. K-means Clustering과 같은 방법을 이용하면 비정상적인 네트워크 연결을 탐지해 낼 수 있다. K-means Clustering을 이용하여 네트워크 연결을 클러스터링 하고 기존의 정상적인 네트워크 연결의 클러스터와는 다른 연결이 요청되었을 때, 이것을 비정상적인 연결이라고 판단할 수 있다.

<br>
<br>

## KDD Cup 1999 Data Set

KDD Cup은 ACM에서 매년 열리는 데이터 마이닝 대회이다. 각 해마다 머신 러닝 문제가 주어지고, 그것을 얼마만큼의 정확도로 해결하는가에 대한 대회이다. 1999년에 열렸던 대회는 네트워크 침입에 대한 대회였는데 데이터셋은 지금도 접근 가능하다. 이 포스트에서는 이 데이터를 Spark을 통해 분석하여 비정상적인 접근을 찾아내도록 할 것이다.

다행히도 대회 개최자들이 이미 Raw 네트워크 데이터를 전처리하여 요약한 데이터를 제공하며 약 743MB의 490만개의 네트워크 연결에 대한 데이터이다. 데이터가 엄청 많은 것은 아니지만 이 챕터에서 수행하고자 하는 것들을 수행해보기에는 적절하다. 데이터 셋은 [이 링크](http://kdd.ics.uci.edu/databases/kddcup99/kddcup.data.gz)에서 다운로드 할 수 있다.

데이터의 각 행은 전달된 바이트 수, 로그인 시도, TCP 에러 등과 같은 정보를 포함한다. 각 데이터는 CSV 형태로 존재하며 제일 마지막 레이블을 제외한 41개의 Feature로 구성되어 있다.

```
0,tcp,http,SF,215,45076,0,0,0,0,0,1,0,0,0,0,0,0,0,0,0,0,1,1,0.00,0.00,0.00,0.00,1.00,0.00,0.00,0,0,0.00,0.00,0.00,0.00,0.00,0.00,0.00,0.00,normal.
```

위와 같은 형태로 존재하는데, TCP 연결, HTTP 서비스, 215바이트 송신, 45076바이트 수신과 같은 정보를 나타내며 각 Feature의 의미는 다음과 같다.

<center><img src="https://db.tt/DO8fq1YF" width="300"></center>

데이터를 살펴보면 많은 값이 15번째 컬럼과 같이 0 혹은 1임을 알 수 있다. 이것은 해당 값이 있는지 없는지 여부를 나타내며 이전 포트스에서처럼 특정 값을 비트로 표현한 것이 아니라 각각의 Feature가 특정 의미를 갖고 있는 것이다. 그리고 *dst\_host\_srv\_rerror\_rate* 이후의 컬럼은 0.0에서 1.0까지의 값을 갖는다.

가장 마지막 필드에는 해당 네트워크 연결의 레이블이 주어져있다. 대부분의 레이블이 `normal`이지만 몇몇 값들은 다음과 같이 특정 공격에 대한 타입이 기입되어 있다.

```
0,icmp,ecr_i,SF,1032,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,511,511,0.00,0.00,0.00,0.00,1.00,0.00,0.00,255,255,1.00,0.00,1.00,0.00,0.00,0.00,0.00,0.00,smurf.
```

이러한 정보들은 훈련 과정에서 유용하게 사용될 수 있지만, 이 장에서 하고자 하는 것은 비정상적인 접근 트래픽을 감지하는 것이기 때문에 잠재적으로 새롭고 알려지지 않은 접근을 찾아낼 것이다. 따라서 이 레이블 정보는 거의 제외된 상태로 사용될 것이다.

<br>
<br>

## A First Take on Clustering

kddcup.data.gz파일의 압축을 해제하고, **kddcup.data.corrected** 파일을 HDFS로 업로드한다. 이 포스트에서는 **/kdd/kddcup.data.corrected**경로에 업로드 한 것을 가정한다.

Spark-shell에서 다음과 같이 파일을 로드하고, 각 레이블별로 어느정도의 양이 있는지 확인함으로써 간단히 데이터를 확인한다.

```scala
val rawData = sc.textFile("/kdd/kddcup.data.corrected")
rawData.map(_.split(',').last).countByValue().toSeq.sortBy(_._2).reverse.foreach(println)
```

다음과 같이 23개의 레이블이 존재하며, 가장 많은 공격 형태는 smurf 이고, neptune 순서이다.

```
(smurf.,2807886)
(neptune.,1072017)
(normal.,972781)
...
```

K-means Clustering을 수행하기 전에 주의해야 할 것이 있다. KDD 데이터셋은 숫자 형태의 데이터가 아닌 Feature(nonnumeric feature)가 있다. 예를 들면 두 번째 열과 같은 경우에는 그 값이 **tcp, udp, icmp**와 같은 값들이다. 하지만 K-means Clustering에서는 숫자 형태의 Feature만 사용할 수 있다. 처음에는 이 값을들 무시하고 진행을 할 것이다. 그리고 뒷 부분에서 이 Categorical Feature를 포함하여 클러스터링을 수행할 것이다.

다음의 코드는 데이터를 파싱하여 K-means Clustering에 필요한 데이터로 필터링 하는 과정이다. 이 과정에서 제일 마지막의 레이블을 포함한 Categorical Feature를 제외한다. 그리고 이 데이터를 이용하여 K-means Clustering을 수행한다.

```scala
import org.apache.spark.mllib.linalg._
import org.apache.spark.mllib.clustering._

val labelsAndData = rawData.map { line =>
	val buffer = line.split(',').toBuffer
	buffer.remove(1, 3)
	val label = buffer.remove(buffer.length - 1)
	val vector = Vectors.dense(buffer.map(_.toDouble).toArray)
	(label, vector)
}

val data = labelsAndData.values.cache()

val kmeans = new KMeans()
val model = kmeans.run(data)

model.clusterCenters.foreach(println)

```

위 코드의 수행 결과로 다음과 같이 두 개의 클러스터의 중심 벡터가 출력될 것이다. 그것은 K-means Clustering을 통해 \\(k=2\\)로 데이터가 맞춰졌다는 것이다.

```
[48.34019491959669,1834.6215497618625,826.2031900016945,5.7161172049003456E-6,6.487793027561892E-4,7.961734678254053E-6,0.012437658596734055,3.205108575604837E-5,0.14352904910348827,0.00808830584493399,6.818511237273984E-5,3.6746467745787934E-5,0.012934960793560386,0.0011887482315762398,7.430952366370449E-5,0.0010211435092468404,0.0,4.082940860643104E-7,8.351655530445469E-4,334.9735084506668,295.26714620807076,0.17797031701994342,0.1780369894027253,0.05766489875327374,0.05772990937912739,0.7898841322630883,0.021179610609908736,0.02826081009629284,232.98107822302248,189.21428335201279,0.7537133898006421,0.030710978823798966,0.6050519309248854,0.006464107887636004,0.1780911843182601,0.17788589813474293,0.05792761150001131,0.05765922142400886]

[10999.0,0.0,1.309937401E9,0.0,0.0,0.0,0.0,0.0,0.0,0.0,0.0,0.0,0.0,0.0,0.0,0.0,0.0,0.0,0.0,1.0,1.0,0.0,0.0,1.0,1.0,1.0,0.0,0.0,255.0,1.0,0.0,0.65,1.0,0.0,0.0,0.0,1.0,1.0]
```

하지만 앞서 데이터를 살펴보았던 것과 마찬가지로 총 23개의 레이블이 존재하는데, 단순히 두 개의 클러스터로 구분하는 것은 적절치 않다는 것을 직관적으로 알 수 있다. 때문에 이미 알고 있는 레이블 정보를 이용하여 클러스터링 결과가 실제 데이터와 어떤 차이를 보이는지 확인 해 보는것은 많은 도움이 된다. 다음 코드를 이용하면 각 클러스터별로 어떤 형태의 공격에 대한 레이블이 포함되었는지 확인할 수 있다.

```scala
val clusterLabelCount = labelsAndData.map { case (label, datum) =>
	val cluster = model.predict(datum)
	(cluster, label)
}.countByValue

clusterLabelCount.toSeq.sorted.foreach {
	case ((cluster, label), count) =>
	println(f"$cluster%1s$label%18s$count%8s")
}
```

그 결과는 다음과 같다.

```
0             back.    2203
0  buffer_overflow.      30
0        ftp_write.       8
0     guess_passwd.      53
0             imap.      12
0          ipsweep.   12481
0             land.      21
0       loadmodule.       9
0         multihop.       7
0          neptune. 1072017
0             nmap.    2316
0           normal.  972781
0             perl.       3
0              phf.       4
0              pod.     264
0        portsweep.   10412
0          rootkit.      10
0            satan.   15892
0            smurf. 2807886
0              spy.       2
0         teardrop.     979
0      warezclient.    1020
0      warezmaster.      20
1        portsweep.       1
```

위 결과는 **portsweep** 공격에 대한 데이터를 제외한 모든 데이터들이 **0번 클러스터**에 할당되어 있음을 알 수 있다.

<br>
<br>

## Choosing K

앞선 결과를 통해 단순히 두개의 클러스터는 부족하다는 것을 알 수 있다. 그렇다면 몇 개의 클러스터가 이 데이터셋에 적절할것인가? 이 데이터에 존재하는 레이블의 종류가 23개 이므로 \\(k\\)는 최소 23과 비슷하거나 큰 것이 결과가 좋을 것이다. 일반적으로 \\(k\\)를 결정할 때는 많은 값들이 고려된다. 근데, 어떤 것이 과연 "좋은" \\(k\\)일까?

클러스터링이 잘 되었다는 것은 각 데이터와 그 데이터가 속하는 클러스터의 중심과 거리가 가까운 것을 이야기한다. 그래서 그것을 계산하기 위해 Eculidean distance를 계산하는 함수를 정의하고, 그 함수를 이용해 각 데이터가 가장 가까운 클러스터의 중심과 얼마만큼 떨어져 있는지 계산할 것이다.

```scala
def distance(a: Vector, b: Vector) = {
	math.sqrt(a.toArray.zip(b.toArray).map(p => math.pow(p._1 - p._2, 2)).sum)
}

def distToCentroid(datum: Vector, model: KMeansModel) = {
	val cluster = model.predict(datum)
	val centroid = model.clusterCenters(cluster)
	distance(centroid, datum)
}
```

이 함수를 이용하여 클러스터의 갯수 \\(k\\)가 주어졌을 때, 각 데이터와 가장 가까운 클러스터 사이의 거리에 대한 평균값을 계산할 수 있다.

```scala
import org.apache.spark.rdd._

def clusteringScore(data: RDD[Vector], k: Int) = {
	val kmeans = new KMeans()
	kmeans.setK(k)
	val model = kmeans.run(data)
	data.map(datum => distToCentroid(datum, model)).mean()
}

(5 to 40 by 5).map(k => (k, clusteringScore(data, k))).foreach(println)
```

다음과 같은 결과를 볼 수 있는데, 클러스터의 수가 늘어날수록 각 데이터와 클러스터의 중심점과의 거리 평균이 줄어드는 경향을 보인다는 것을 알 수 있다.

```python
(5,1938.8583418059188)
(10,1661.2533261157496)
(15,1405.536523064836)
(20,1111.7423030526104)
(25,946.2578661660172)
(30,597.6141598314152)
(35,748.4808532423143)
(40,513.382773134806)
```


하지만, 이러한 결과는 너무 뻔한 것이다. 많은 클러스터 중심을 이용 할 수록 데이터들이 각 클러스터의 중심과 가까울 것은 명백하고, 만약 데이터의 수와 클러스터의 수를 동일하게 설정한다면 각 데이터가 하나의 클러스터가 될 것이므로 각 데이터와 클러스터의 중심 사이의 거리는 0일 것이다. 또한 이상하게도 클러스터 개수가 35개일 때의 거리의 평균이 클러스터 개수가 30개일 때보다도 높다.
이것은 반드시 \\(k\\)가 높을 때에도 적은 \\(k\\)로 좋은 클러스터링을 수행하는 것을 허용하기 때문이다.

이것은 K-means Clustering은 꼭 주어진 \\(k\\)만을 이용해서 클러스터링을 수행해야만 하는 것은 아님을 이야기한다. 이것은 반복 과정에서  좋긴 하지만 최적은 아닌 local minimum 등에서 클러스터링이 멈출 수 있음을 의미한다. 이러한 것은 좀 더 지능적인 K-means Clustering을 이용하여 좋은 initial centroid(초기 중심점)을 선택할 때에도 마찬가지로 존재하는 문제이다.

예를 들면 K-means++ 과 같은 방법들은 여러 선택 알고리즘을 이용하여 다양하고, 분산된 중심점들을 선택하여 가장 기초적인 K-means Clustering 방식보다 좋은 결과를 이끌어낸다. 하지만 이러한 방법들도 마찬가지로 무작위(Random)적인 선택을 기반으로 하기 때문에 최적의 클러스터링을 보장하지는 않는다.

이 결과를 반복 횟수를 증가시킴으로써 성능을 증가시킬수 있다. `setRuns()`함수는 하나의 \\(k\\)마다 클러스터링을 수행하는 횟수를 설정하는 것이며, `setEpsilon()`함수는  각 반복마다  확인하는 클러스터의 중심의 이동한 차이에 대한 Threshold이다. Epsilon을 조절함으로써 각 클러스터링의 수행마다 얼마만큼의 반복이 수행될 지를 결정하는데 영향을 미친다. Epsilon을 큰 값으로 설정하면 클러스터 중심점의 변경에 민감하지 않을 것이고 작은 값으로 설정한다면 작은 클러스터 중심점의 변경에도 민감하게 반응할 것이다.(민감하다는 것은 적은 차이에도 새로운 반복을 수행한다는 것)

다음과 같이 해당 부분들을 변경시켜 수행할 수 있다.

```scala
kmeans.setRuns(10)
kmeans.setEpsilon(1.0e-6) //Epsilon의 기본값은 1.0e-4

(30 to 100 by 10).par.map(k => (k, clusteringScore(data, k))).toList.foreach(println)
```

위 코드의 수행 결과로 이제는 평가 수치가 점점 감소하는 결과를 보인다.

```
(30,654.6479150668039)
(40,833.771092804366)
(50,408.29802592345465)
(60,298.43382866843206)
(70,256.6636429518882)
(80,163.15198293023954)
(90,135.78737972772348)
(100,118.73064012152163)
```

이 결과를 이용하여 최적의 \\(k\\)를 선택해야 하는데, [Elbow를 선택하는 방법](https://en.wikipedia.org/wiki/Determining_the_number_of_clusters_in_a_data_set#The_Elbow_Method)을 이용하면 \\(k=50, k=80\\) 혹은 가장 작은 \\(k=100\\) 등의 값이 적절해보인다.

<br>
<br>

## Visualization in R

이 시점에서 데이터들이 어떤 형태로 구성되어 있는가 확인해보는 것은 많은 도움이 된다. 하지만  Spark에는 자체적인 시각화 도구를 갖고 있지 않기 때문에 다른 도구의 도움을 받아야 한다. 하지만 Spark에서 수행하는 결과들은 쉽게 HDFS로 저장될 수 있고, 그것은 쉽게 다른 통계 툴에서 사용할 수 있다. 이 챕터에서는 R을 이용하여 데이터를 시각화 해 볼 것이다.

R은 2차원 혹은 3차원의 데이터를 시각화 하는 반면 우리가 지금까지 사용해 온 데이터는 38차원이다. 때문에 이 데이터를 최대 3차원을 갖는 데이터로 줄여야(Project)할 필요성이 있다. 게다가 R에서는 많은 데이터를 처리하기에는 무리가 있다. 따라서 데이터의 수 역시 샘플링을 통해 줄여야 한다.

시작에 앞서 \\(k=100\\)을 이용해 모델을 생성하고, 각 데이터가 속하는 클러스터의 번호를 각 데이터에 매핑한다. 그리고 이것을 CSV 형태로 HDFS에 출력한다.

```scala
val kmeans = new KMeans()
kmeans.setK(100)
kmeans.setRuns(5)
kmeans.setEpsilon(1.0e-6)

val model = kmeans.run(data)

val sample = data.map(datum =>
	model.predict(datum) + "," + datum.toArray.mkString(",")
).sample(false, 0.05)

sample.saveAsTextFile("/kdd/sample")
```

`sample` 함수는 전체 데이터에서 원하는 만큼의 데이터를 샘플링 할 수 있도록 해 준다. 이 예시에서는 전체 데이터 중 5%를 반복 없이 샘플링한다.

다음의 R 코드는 HDFS에서 CSV파일을 읽어서 활용하는 코드이다. 이 코드에서는 `rgl`이라는 패키지를 이용하여 데이터를 그래프에 나타낸다. 이 과정에서 38차원의 데이터 중 무작위로 세 개의 Unit Vector를 선택하여 그 값을 이용해 3차원 벡터를 생성하고, 그것을 그래프에 그린다. 이 과정은 간단한 과정의 Dimension Reduction(차원 감소)이다. 물론 이 과정보다 훨씬 정한 알고리즘들(ex: [PCA](https://en.wikipedia.org/wiki/Principal_component_analysis), [SVD](https://en.wikipedia.org/wiki/Singular_value_decomposition))이 존재하고, 이것을 Spark에서나 R에서 수행이 가능하지만 이것을 추가로 수행하는데에 추가적인 시간이 소요되고, 이 챕터의 내용을 벗어나는 것이어서 사용하지는 않는다.

(다음 코드를 수행하기 위해서는 `rgl`패키지를 설치할 수 있는 환경이 구성되어야 한다.)

```r
install.packages("rgl") # 처음 한번만 수행하면 된다.
library(rgl)

# HDFS로부터 CSV형태의 데이터 읽기
clusters_data <- read.csv(pipe("hdfs dfs -cat /kdd/sample/*"))
clusters <- clusters_data[1]
data <- data.matrix(clusters_data[-c(1)])
rm(clusters_data)

# Random Unit Vector 생성
random_projection <- matrix(data = rnorm(3*ncol(data)), ncol = 3)
random_projection_norm <-
	random_projection / sqrt(rowSums(random_projection * random_projection))
	
projected_data <- data.frame(data %*% random_projection_norm)

num_clusters <- nrow(unique(clusters))
palette <- rainbow(num_clusters)
colors = sapply(clusters, function(c) palette[c])
plot3d(projected_data, col = colors, size = 10)
```

위 코드의 수행 결과로는 다음과 같은 형태의 3D 그래프가 출력된다.

![Random 3D Projection](https://db.tt/NLkpVFIZ)

이 결과는 각 클러스터의 번호가 색깔로 구분된 데이터들의 좌표(위 그림에서는 하나의 클러스터가 대부분을 차지하는 것으로 나오지만 확대하면 다양한 클러스터의 색깔을 볼 수 있다.)로 표현된 것이다. 데이터를 어떻게 분석해야 할지 막막하지만, 데이터의 분포가 `"L"`자 형태인 것만은 분명하다. 그리고 그 L의 한 쪽은 길고, 한쪽은 짧다.

이것은 Feature중에 그 값의 범위가 다른 Feature에 비해 큰 것들이 존재한다는 것이다. 예를 들면 대부분의 Feature의 값은 0과 1 사이인 것에 비해 주고받은 Byte의 수에 대한 Feature는 그 범위가 매우 크다. 때문에 각 클러스터로부터의 거리를 계산하여 클러스터링의 성능을 평가할 때 주고받은 Byte 수에 대한 Feature가 영향령이 크다는 것이 된다. 따라서 앞서 계산한 Euclidean Distance에서는 다른 Feature들이 거의 무시되고 주고받은 Byte 수에 대한 것이 대부분이라는 것이다. 이것을 방지하기 위해 각 Feature 값을 Normalize 할 필요가 있다.

<br>
<br>

## Feature Normalization

우리는 다음과 같은 식을 이용하여 각 Feature를 Normalize할 수 있다.

\\(
{ normalized }\_{ i }=\frac { { feature }\_{ i }-{ \mu  }\_{ i } }{ { \sigma  }\_{ i } } 
\\)

위 식은 각 Feature의 값에서 평균을 뺀 후에, 그것을 표준편차로 나누어 정규화 시키는 것이다. 사실 평균을 각 데이터에서 빼는 것은 클러스터링 과정에 아무 영향을 미치지 않는다. 모든 값을 같은 양만큼 같은 뱡향으로 움직이는 것이기 때문이다. 그렇지만, 일반적으로 사용하는 정규화 식을 그대로 이용하기 위하여 평균을 빼는 과정을 제외하지는 않았다.

표준값은 각 Feature별로 갯수, 합, 제곱합을 계산함으로써 구할 수 있다. 다음과 같은 코드를 이용해 각 Feature의 값을 Normalization할 수 있다.

```scala
val dataAsArray = data.map(_.toArray)
val numCols = dataAsArray.first().length
val n = dataAsArray.count()
val sums = dataAsArray.reduce {
	(a, b) => a.zip(b).map(t => t._1 + t._2)
}

val sumSquares = dataAsArray.aggregate(
	new Array[Double](numCols)
	)(
		(a, b) => a.zip(b).map(t => t._1 + t._2 * t._2),
		(a, b) => a.zip(b).map(t => t._1 + t._2)
	)	

val stdevs = sumSquares.zip(sums).map {
	case (sumSq, sum) => math.sqrt(n * sumSq - sum * sum) / n
}

val means = sums.map(_ / n)
```

위 코드에서 `aggregate` 함수에 대해 어려움을 느낄 수 있는데, Python API를 이용해 설명하고 있지만, [이 블로그](http://atlantageek.com/2015/05/30/python-aggregate-rdd/)에서 잘 설명을 하고 있다. 간단히 얘기하면 aggregate 함수의 파라미터로는 초기값(0 Value)이 사용되고, 첫번째 람다함수는 결과 Object를 생성하는 과정에서 어떻게 그 결과를 만들어 낼 것인지(위 코드의 예시에서는 제곱의 합을 구하는 과정)에 대한 것이고, 두번째 람다함수는 여러개의 결과 Object를 Combine할 때 어떻게 할 것인지에 대한 함수이다. 결론적으로 위 과정에서 제곱의 합을 구하는 과정은 각 Partition에서는 zip 된 두 개의 데이터 중 Feature별로 다음 것을 제곱하여 이전의 값에 더해나가고, 그 Partition을 합할 때는 덧셈을 수행함으로써 모든 데이터의 Feature별 제곱의 합을 구한다.

앞서 구한 값들을 이용해 데이터를 정규화하고, 그 데이터를 이용해 K-means 클러스터링을 수행시킨 뒤 다시 Euclidean Distance를 계산한다.

```scala
def normalize(datum: Vector) = {
	val normalizedArray = (datum.toArray, means, stdevs).zipped.map {
	(value, mean, stdev) =>
		if (stdev <= 0) (value - mean) else (value - mean) / stdev
	}
	Vectors.dense(normalizedArray)
}

val normalizedData = data.map(normalize).cache()

(60 to 120 by 10).par.map {
	k => (k, clusteringScore(normalizedData, k))
}.toList.foreach(println)
```

위 코드의 수행 결과는 다음과 같다.

```
(60,0.35788542308188337)
(70,0.33739539223932735)
(80,0.35734816618118775)
(90,0.3333896914476046)
(100,0.2866857023435722)
(110,0.27457555082113716)
(120,0.25987997589819595)
```

위 결과를 통해 elbow인 \\(k=100\\)은 괜찮은 선택이었음을 알 수 있다. 앞서 정규화한 데이터를 다시 R을 통해 시각화하여 어떤 차이가 있는지 확인해 볼 수 있다. 다음 코드를 이용해 Normalize된 데이터를 이용하여 K-means Clustering을 수행시키고, 그것을 HDFS에 저장한다. (확실한 결과를 위해 setRun()을 통해 옵션을 정하였다.)

```scala
val kmeans = new KMeans()
kmeans.setK(100)
kmeans.setRuns(5)
kmeans.setEpsilon(1.0e-6)

val model = kmeans.run(normalizedData)

val sample = normalizedData.map(datum =>
	model.predict(datum) + "," + datum.toArray.mkString(",")
).sample(false, 0.05)

sample.saveAsTextFile("/kdd/normalized_sample")
```

그리고 R에서 다시 그래프를 생성하였다.

```r
library(rgl)

# HDFS로부터 CSV형태의 데이터 읽기
clusters_data <- read.csv(pipe("hdfs dfs -cat /kdd/normalized_sample/*"))
clusters <- clusters_data[1]
data <- data.matrix(clusters_data[-c(1)])
rm(clusters_data)

# Random Unit Vector 생성
random_projection <- matrix(data = rnorm(3*ncol(data)), ncol = 3)
random_projection_norm <-
	random_projection / sqrt(rowSums(random_projection * random_projection))
	
projected_data <- data.frame(data %*% random_projection_norm)

num_clusters <- nrow(unique(clusters))
palette <- rainbow(num_clusters)
colors = sapply(clusters, function(c) palette[c])
plot3d(projected_data, col = colors, size = 10)
```

생성된 그래프는 다음과 같으며, 정규화 이전의 데이터에 비해 좀 더 풍부한(richer) 구조를 띄고 있음을 알 수 있다.

![Random 3D Projection of Normalized Data] (https://db.tt/jZ8WlR1x)

<br>
<br>

## Using Labels with Entropy

지금까지 우리는 클러스터링의 성능을 판단하기 위해, 그리고 그것을 이용해 적절한 \\(k\\)를 찾는 과정에서 매우 간단한 것을 이용하였다. 여기에 지난 Chapter에서 사용한 방식을 적용할 수 있다. `Gini inputiry`와 `Entropy`방식이 그것인데, 여기서는 Entropy 방식을 이용하여 설명할 것이다.

좋은 클러스터링이라는 것은 각각의 클러스터가 하나의 label에 대한 데이터만 가지고 있고, 결과적으로 낮은 Entropy를 갖는 것을 의미한다. 때문에 여기에 가중평균을 적용하여 클러스터링의 스코어를 계산할 수 있다.*

```scala
val entropy(counts: Iterable[Int]) = {
	val values = counts.filter(_ > 0)
	val n: Double = values.sum
	values.map { v =>
		val p = v / n
		-p * math.log(p)
	}.sum
}

def clusteringScore(normLabelAndData: RDD[(String, Vector)], k:Int) = {
	val kmeans = new KMeans()
	kmeans.setK(k)
	kmeans.setRuns(5)
	kmeans.setEpsilon(1.0e-6)

	val model = kmeans.run(normLabelAndData.values)

	val labelsAndClusters = normLabelAndData.mapValues(model.predict)
	val clustersAndLabels = labelsAndClusters.map(_.swap)
	val labelsInCluster = clustersAndLabels.groupByKey().values
	val labelCounts = labelsInCluster.map(_.groupBy(l => l).map(_._2.size))
	val n = normLabelAndData.count()

	labelCounts.map(m => m.sum * entropy(m)).sum / n
}

val normalizedLabelsAndData = labelsAndData.mapValues(normalize).cache()

(80 to 160 by 10).par.map {
	k => (k, clusteringScore(normalizedLabelsAndData, k))
}.toList.foreach(println)

normalizedLabelsAndData.unpersist()
```

<br>
위 코드의 수행 결과는 다음과 같으며, \\(k=150\\)일때 가장 좋은 성능을 나타냄을 알 수 있다.\**
<br>

```
(80,0.01633056783505308)
(90,0.014086821003939093)
(100,0.013287591809429072)
(110,0.011314751300005676)
(120,0.012290307370115603)
(130,0.00949651182021723)
(140,0.008943810114864363)
(150,0.007499306029722229)
(160,0.008704684195176402)
(170,0.008691369298104417)
(180,0.008559061207118177)
```

## Categorical Variables

지금까지 수행한 클러스터링은 세 개의 Categorical Feature를 제외하고 수행하였다. 왜냐면, MLlib에서의 K-means Clustering은 숫자 형태의 데이터가 아닌 것에 대해서는 클러스터링을 수행할 수 없기 때문이다. Categorical Feature를 클러스터링에 포함시키기 위하여 지난 챕터에서 사용한 데이터베이스가 활용하고 있는 방법인, Feature의 값의 범위를 하나의 비트로 표현하는 방법을 이용할 것이다.

예를 들면 두 번째 Feature의 경우에는 어떤 형태의 프로토콜이 사용되었는지에 대한 값이다. 이것은 **tcp, udp, icmp**의 값을 갖는데, 이 것을 바이너리의 값을 갖는 세 개의 Feature, 예를들어 **is_tcp, is_udp, is_icmp**와 같은 Feature로 나누어 저장하는 것이다. 그리고 어떤 데이터가 원래 데이터에서 **udp** 프로토콜을 사용한 데이터였다면, **0.0, 1.0, 0.0**으로 표현하는 것이다.

이 방법을 데이터에 적용하면 다시 Normalize와, 클러스터링을 수행시켜야 한다. 책에는 이 과정에 대한 코드가 있지 않지만, [Github Repository](https://github.com/sryza/aas)에 해당 코드가 있어 사용하였다.

```scala
import org.apache.spark.mllib.clustering._
import org.apache.spark.mllib.linalg._
import org.apache.spark.rdd._

def buildCategoricalAndLabelFunction(rawData: RDD[String]): (String => (String,Vector)) = {
  val splitData = rawData.map(_.split(','))
  val protocols = splitData.map(_(1)).distinct().collect().zipWithIndex.toMap
  val services = splitData.map(_(2)).distinct().collect().zipWithIndex.toMap
  val tcpStates = splitData.map(_(3)).distinct().collect().zipWithIndex.toMap
  (line: String) => {
    val buffer = line.split(',').toBuffer
    val protocol = buffer.remove(1)
    val service = buffer.remove(1)
    val tcpState = buffer.remove(1)
    val label = buffer.remove(buffer.length - 1)
    val vector = buffer.map(_.toDouble)

    val newProtocolFeatures = new Array[Double](protocols.size)
    newProtocolFeatures(protocols(protocol)) = 1.0
    val newServiceFeatures = new Array[Double](services.size)
    newServiceFeatures(services(service)) = 1.0
    val newTcpStateFeatures = new Array[Double](tcpStates.size)
    newTcpStateFeatures(tcpStates(tcpState)) = 1.0

    vector.insertAll(1, newTcpStateFeatures)
    vector.insertAll(1, newServiceFeatures)
    vector.insertAll(1, newProtocolFeatures)

    (label, Vectors.dense(vector.toArray))
  }
}

def buildNormalizationFunction(data: RDD[Vector]): (Vector => Vector) = {
  val dataAsArray = data.map(_.toArray)
  val numCols = dataAsArray.first().length
  val n = dataAsArray.count()
  val sums = dataAsArray.reduce(
    (a, b) => a.zip(b).map(t => t._1 + t._2))
  val sumSquares = dataAsArray.aggregate(
    new Array[Double](numCols)
    )(
    (a, b) => a.zip(b).map(t => t._1 + t._2 * t._2),
    (a, b) => a.zip(b).map(t => t._1 + t._2)
    )
    val stdevs = sumSquares.zip(sums).map {
      case (sumSq, sum) => math.sqrt(n * sumSq - sum * sum) / n
    }
    val means = sums.map(_ / n)

    (datum: Vector) => {
      val normalizedArray = (datum.toArray, means, stdevs).zipped.map(
        (value, mean, stdev) =>
        if (stdev <= 0)  (value - mean) else  (value - mean) / stdev
        )
      Vectors.dense(normalizedArray)
    }
}

def entropy(counts: Iterable[Int]) = {
  val values = counts.filter(_ > 0)
  val n: Double = values.sum
  values.map { v =>
    val p = v / n
    -p * math.log(p)
    }.sum
}

def buildNormalizationFunction(data: RDD[Vector]): (Vector => Vector) = {
  val dataAsArray = data.map(_.toArray)
  val numCols = dataAsArray.first().length
  val n = dataAsArray.count()
  val sums = dataAsArray.reduce(
    (a, b) => a.zip(b).map(t => t._1 + t._2))
  val sumSquares = dataAsArray.aggregate(
    new Array[Double](numCols)
    )(
      (a, b) => a.zip(b).map(t => t._1 + t._2 * t._2),
      (a, b) => a.zip(b).map(t => t._1 + t._2)
    )
  val stdevs = sumSquares.zip(sums).map {
    case (sumSq, sum) => math.sqrt(n * sumSq - sum * sum) / n
  }
  val means = sums.map(_ / n)

  (datum: Vector) => {
    val normalizedArray = (datum.toArray, means, stdevs).zipped.map(
      (value, mean, stdev) =>
      if (stdev <= 0)  (value - mean) else  (value - mean) / stdev
      )
    Vectors.dense(normalizedArray)
  }
}

def clusteringScore(normalizedLabelsAndData: RDD[(String,Vector)], k: Int) = {
  val kmeans = new KMeans()
  kmeans.setK(k)
  kmeans.setRuns(10)
  kmeans.setEpsilon(1.0e-6)

  val model = kmeans.run(normalizedLabelsAndData.values)
  val labelsAndClusters = normalizedLabelsAndData.mapValues(model.predict)
  val clustersAndLabels = labelsAndClusters.map(_.swap)
  val labelsInCluster = clustersAndLabels.groupByKey().values
  val labelCounts = labelsInCluster.map(_.groupBy(l => l).map(_._2.size))
  val n = normalizedLabelsAndData.count()

  labelCounts.map(m => m.sum * entropy(m)).sum / n
}

val rawData = sc.textFile("/kdd/kddcup.data.corrected")

val parseFunction = buildCategoricalAndLabelFunction(rawData)
val labelsAndData = rawData.map(parseFunction)
val normalizedLabelsAndData =
	labelsAndData.mapValues(buildNormalizationFunction(labelsAndData.values)).cache()

val result = (80 to 160 by 10).map(k =>
  (k, clusteringScore(normalizedLabelsAndData, k))).toList
result.foreach(println)

normalizedLabelsAndData.unpersist()
```

위 코드의 수행 결과는 다음과 같다. \***

```
(80,0.02779435579758084)
(90,0.05653844879893403)
(100,0.029429111090986747)
(110,0.022128545398091923)
(120,0.022724916386673424)
(130,0.02103069110661259)
(140,0.01920591910565662)
(150,0.019929832533142584)
(160,0.019637563306766435)
```

이로부터 \\(k=140\\)일때가 최적의 \\(k\\)임을 알 수 있다.

<br>
<br>

## Clustering in Action

이제, 남은 과정은 네트워크 트래픽 중에서 비정상적인 트래픽을 감지해 내는 것이다.

이 장의 처음 부분에서 했던 것과 마찬가지로 현재 \\(k=140\\)으로 클러스터링을 진행하였을 때의 각 클러스터별 데이터 레이블의 상태를 확인해 볼 수 있다.

```scala
val rawData = sc.textFile("/kdd/kddcup.data.corrected")

val parseFunction = buildCategoricalAndLabelFunction(rawData)
val labelsAndData = rawData.map(parseFunction)
val normalizedLabelsAndData =
	labelsAndData.mapValues(buildNormalizationFunction(labelsAndData.values))
val normalizedData = normalizedLabelsAndData.values.cache()

val kmeans = new KMeans()
kmeans.setK(140)
kmeans.setRuns(10)
kmeans.setEpsilon(1.0e-6)

val model = kmeans.run(normalizedData)
val labelsAndClusters = normalizedLabelsAndData.mapValues(model.predict)
val clustersAndLabelsCount = labelsAndClusters.map(_.swap).countByValue

clustersAndLabelsCount.toSeq.sorted.foreach {
    case ((cluster, label), count) =>
    println(f"$cluster%1s$label%18s$count%8s")
}

normalizedData.unpersist()
```

위 코드를 이용하면 다음과 같은 결과를 볼 수 있다. 처음 확인하였던 결과와는 다르게, 각 클러스터별로 최소 하나 이상의 레이블 데이터가 있으며, 한 레이블이 높은 비율을 차지하고 있다는 것을 알 수 있다.

```
0          neptune.  362825
0        portsweep.       9
1           normal.      13
1        portsweep.     645
2          neptune.     200
2        portsweep.       6
2            satan.       1
3          neptune.    1037
3        portsweep.      13
3            satan.       3
4             back.      43
4           normal.   63268
...
138           normal.    1005
139  buffer_overflow.       6
139        ftp_write.       4
139          ipsweep.      13
139         multihop.       3
139           normal.   35974
139        portsweep.       1
139          rootkit.       1
139      warezclient.     701
139      warezmaster.      18
```

이런 결과를 갖는 클러스터링 모델에서 비정상적인 트래픽을 찾아낼 것인데, 그 과정은 다음과 같다.

트레이닝한 데이터별로 각 그 데이터가 속하는 클러스터의 중심점과의 거리를 계산하고(이 과정은 앞서 진행하였다.), 그 길이들은 내림차순으로 정렬하여 100번째의 거리값을 Threshold로 정한다. 그리고 전체 데이터중 중심점과의 Threshold를 초과하는 것들을 비정상적인 데이터로 간주할 것이다. 그러기 위해선 데이터를 다시 normalize해야 한다.(앞선 과정에서 진행하였지만, 앞서 진행한 코드를 그대로 적용할 수 없어서 다시 계산한다.) 이어서 Threshold를 계산한 다음에 Threshold를 이용하여 각 데이터를 필터링한다.

```scala
val distances = normalizedData.map(datum => distToCentroid(datum, model))
val threshold = distances.top(100).last
val originalAndParsed = rawData.map(line => (line, parseFunction(line)._2))

val dataAsArray = originalAndParsed.map(line => line._2.toArray)
val numCols = dataAsArray.first().length
val n = dataAsArray.count()
val sums = dataAsArray.reduce {
	(a, b) => a.zip(b).map(t => t._1 + t._2)
}

val sumSquares = dataAsArray.aggregate(
	new Array[Double](numCols)
	)(
		(a, b) => a.zip(b).map(t => t._1 + t._2 * t._2),
		(a, b) => a.zip(b).map(t => t._1 + t._2)
	)	

val stdevs = sumSquares.zip(sums).map {
	case (sumSq, sum) => math.sqrt(n * sumSq - sum * sum) / n
}

val means = sums.map(_ / n)
def normalize(datum: Vector) = {
	val normalizedArray = (datum.toArray, means, stdevs).zipped.map {
	(value, mean, stdev) =>
		if (stdev <= 0) (value - mean) else (value - mean) / stdev
	}
	Vectors.dense(normalizedArray)
}

val originalAndNormalized = originalAndParsed.mapValues(normalize)
val anormalies = originalAndNormalized.filter{
  case (original, normalized) => distToCentroid(normalized, model) > threshold
}.keys
anormalies.take(10).foreach(println)
```

재미있게도 다음 결과가 제일 처음으로 나왔는데, 레이블은 정상적인 트래픽이라는 레이블이다. 하지만 이 정상적이라는 것은 보안에 영향을 미치는지에 대한 여부를 뜻하기 때문에 일반적인 네트워크 트래픽과 비교해 봤을때는 정상 트래픽과는 거리가 있다. 연결은 성공하였지만 그 이후로 아무것도 데이터가 전송되지 않은 것을 의미하는 S1 플래그이며, 짧은 순간 동안 30번 가량의 연결이 존재하였다. 이것으로부터 이 데이터는 악의적인 연결은 아니지만 충분히 정상적이지 않은 연결에 대한 것임을 알 수 있다.

```
0,tcp,telnet,S1,145,13236,0,0,0,0,0,1,31,1,2,38,0,0,0,0,0,0,1,1,1.00,1.00,0.00,0.00,1.00,0.00,0.00,29,10,0.28,0.10,0.03,0.20,0.07,0.20,0.00,0.00,normal.
```

<br>
<br>
<br>
--

\* 이 과정부터 맥북에서 수행시켰을 때 7시간 이상이 걸려, Amazon EMR을 이용하여 계산하였다.

\**, *** 이 결과는 책의 결과와 많이 다르다. 이것 때문에 책의 저자와 이메일을 주고받았는데, 책의 Spark 버전과 이 글을 쓸 때의 Spark 버전과의 차이가 있어서 인것으로 잠정 결론지었다.