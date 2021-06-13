---
layout:     post
title:      "Spark & 머신 러닝 - Recommending Music - 1/2"
subtitle:   "Advanced Analytics with Spark, 2. Recommending Music - 1/2"
category:   Data Analysis
date:       2015-07-13 23:09:00
author:     "Hyunje"
tags: [Spark, Machine Learning, Recommendation]
image:
  background: white.jpg
  feature: bg2.jpg
  credit: Hyunje Jo
keywords:	"데이터 분석, Data Analysis, Spark, Apache Spark, 머신 러닝, Machine Learning"
---

이 글에서는 Spark를 이용하여 추천을 수행하는 과정에 대해 설명한다. [Audioscrobbler Dataset](http://www-etud.iro.umontreal.ca/~bergstrj/audioscrobbler_data.html) 를 이용하여 사용자가 좋아할 만한 음악을 추천해 주는 작업을 할 것이다.

이 포스트는 [Advanced Analytics with Spark](http://shop.oreilly.com/product/0636920035091.do)을 정리한 글이다.

Chapter 3를 두 개의 글로 나누었다. 첫 번째 글은 추천을 수행하고, 간단히 추천 수행 결과를 확인해 보는 정도로 마무리 하고, 두 번째 글은 생성한 추천 모델이 얼마나 효과적으로 추천을 수행해주는지 분석하는 과정이다.

<br>
<br>

## Introduction

추천 엔진은 사람들이 가장 쉽게 접할 수 있는 머신 러닝의 한 예라고 할 수 있다. Amazon, Youtube과 같은 사이트는 물론 대부분의 서비스는 자체적으로 추천 기능을 제공한다. 추천 시스템의 결과물은 현재 시스템을 사용하고 있는 사람이 좋아할만한 아이템이기 때문에, 다른 머신 러닝 알고리즘에 비해 좀 더 직관적으로 이해할 수 있다. 그만큼 추천 시스템은 많은 사람들에게 이미 널리 알려져 있고, 익숙한 머신 러닝 알고리즘이다.

이 챕터에서는 Spark에 정의되어 있는 핵심 머신 러닝 알고리즘 중 추천 시스템과 연관이 있는 것들에 대해 알아볼 것이고, 그것을 이용해 사용자에게 음악을 추천 해 줄 것이다. 이러한 과정들은 Spark와 MLlib의 실제 예시가 될것이며, 이어지는 다른 챕터들에서 사용하게 될 머신 러닝과 관련된 아이디어들에도 도움을 주게 될 것이다.

<br>
<br>

## Data Set

이 챕터에서 수행할 예시 데이터는 [Audtioscrobbler에서 제공하는 데이터셋](http://www-etud.iro.umontreal.ca/~bergstrj/audioscrobbler_data.html)이다. Audioscrobbler 는 Last.fm 에서 처음으로 활용한 음악 추천 시스템이다. Audioscrobbler는 사용자들이 듣는 음악 정보를 자동으로 서버로 전송하여 기록하는 API 를 제공하였는데, 이러한 API의 사용은 많은 사용자의 정보를 기록하는 것으로 이어졌고, Last.fm 에서는 이 정보를 강력한 음악 추천 엔진을 구성하는데 사용하였다.

그 당시의 대부분의 추천 엔진에 대한 연구는 평가기반(사용자 x가 아이템 a에 평점 5를 남겼다)의 데이터에 대한 것들이었다. 하지만 흥미롭게도 Audioscrobbler 데이터는 사용자들이 어떠한 음악을 플레이했다에 대한 정보(사용자 x는 음악 a를 플레이했다.)밖에 제공되지 않는다. 이러한 데이터는 기존 데이터에 비해 난이도가 있는데, 그 이유는 사용자가 음악을 재생했다는 정보가 그 음악을 좋아한다고는 볼 수 없기 때문이다. 이러한 형태의 데이터 셋을 **Implicit Feedback Dataset**이라 한다.

위 링크에서 데이터셋을 다운로드 받아 압축을 해제하면 몇 개의 파일이 나온다. 그 중 가장 핵심이 되는 데이터 파일은 `user_artist_data.txt` 파일이다. 이 파일은 141,000명의 사용자와 160만명의 아티스트에 대한 정보가 들어있으며 사용자의 아티스트에 대한 플레이 정보는 약 2400만 정도의 기록이 저장되어있다. `artist_data.txt`파일은 모든 아티스트에 대한 정보가 들어있지만, 이 데이터 안에는 같은 아티스트를 가리키지만 서로 다른 이름으로 저장되어 있는 경우가 있다. 때문에 이를 위해 같은 아티스트를 가리키고 있는 ID의 Map인 `artist_alias.txt` 파일이 존재한다.


<br>
<br>

## The Alternating Least Squares Recommender Algorithm

추천을 수행하기에 앞서, 설명한 데이터 셋의 형태에 맞는 추천 알고리즘을 선택해야 한다. 우리가 가지고 있는 데이터 셋은 Implicit feedback 형태이며, 사용자에 대한 정보(성별, 나이 등)라던가 아티스트에 대한 정보 역시 존재하지 않는다. 따라서 활용할 수 있는 정보는 **어떠한 사용자가 어떤 아티스트의 노래를 들었다** 라는 정보 뿐이고, 이러한 기록만 이용해서 추천을 수행해야 한다.

이러한 조건에 알맞는 추천 알고리즘은 [Collaborative Filtering, CF](https://en.wikipedia.org/wiki/Collaborative_filtering)[(협업 필터링)](https://ko.wikipedia.org/wiki/협업_필터링)이다. CF는 아이템이나 사용자의 속성을 사용하지 않고 단순히 둘 사이의 관계정보(이 데이터 셋에서는 음악 플레이 여부)만 이용하여 추천을 수행하는 알고리즘이다.

CF에는 여러 알고리즘들이 존재하는데, 여기선 Matrix Factorization 모델을 이용한 추천 알고리즘을 이용하여 추천을 수행한다. Matrix Factorization 계열의 추천 알고리즘은 \\( i \times j\\) 크기의 행렬을 생성하고, 사용자 \\(i\\)가 아티스트 \\(j\\)의 음악을 플레이 했다는 정보를 행렬의 데이터로 이용한다. 이 행렬을 \\(A\\)라 할때, 전체 데이터에 비해서 사용자-아티스트의 조합이 매우 적기 때문에 행렬 \\(A\\)의 데이터는 듬성듬성 존재한다(Sparse 하다고 한다).

Matrix Factorization 방식에서는 이 행렬 \\(A\\)를 두 개의 작은 행렬 \\(X\\)와 \\(Y\\)로 쪼갠다. 이 때 각 행렬의 크기는 \\(i \times k\\), \\(j \times k\\)로, 원래 행렬의 행과 열의 크기가 매우 크기 때문에 두 행렬의 행 역시 매우 크다. 그리고 \\(k\\)는 Latent factor로써, 사용자와 아티스트 사이의 연관을 표현하는데에 이용된다.

![Matrix Factorization] (https://dl.dropboxusercontent.com/u/97648427/blog-img/ch3-1.png)

위 그림 3-1[1]과 같이 행렬 \\(X, Y\\)를 계산한 후에, 사용자 \\(i\\)의 아티스트 \\(j\\)에 대한 평점을 계산하기 위해서는 행렬 \\(X\\)의 \\(i\\)번째 행과, 행렬 \\(Y^T\\)의 \\(j\\)번째 열을 곱하여 계산한다.

이러한 방법을 기반으로 한 추천 알고리즘이 많이 존재 하는데, 이 챕터에서 사용되는 알고리즘은 Alternating Least Squares 알고리즘이다. 이 알고리즘은 Netflix Prize 에서 우승한 논문인 "Collaborative Filtering for the Implicit Feedback Datasets"과, "Large-scale Parallel Collaborative Filtering for the Netflix Prize"에서 주로 사용된 방식이다. 또한 Spark의 MLlib 에는 이 두 논문의 구현체가 구현되어 있다. 이 것을 이용해 이번 챕터를 진행 할 것이다.

<br>
<br>

## Preparing the Data

다운로드 받은 [Audtioscrobbler에서 제공하는 데이터셋](http://www-etud.iro.umontreal.ca/~bergstrj/audioscrobbler_data.html)을 압축 해제하고, HDFS에 업로드한다. 다음과 같이 `/audio` 경로에 업로드하는것을 가정한다.

```
-rw-r--r--   1 hyunje supergroup    2932731 2015-07-12 04:19 /audio/artist_alias.txt
-rw-r--r--   1 hyunje supergroup   55963575 2015-07-12 04:19 /audio/artist_data.txt
-rw-r--r--   1 hyunje supergroup  426761761 2015-07-12 04:19 /audio/user_artist_data.txt
```

또한, 데이터의 크기가 크고, 계산량이 많기 때문에 Spark Shell 을 수행시킬 때 다음과 같이 드라이버의 메모리 용량을 **6GB**이상을 확보시켜야 한다.

```bash
spark-shell --driver-memory 6g --master local[2]
```

Spark의 MLlib 에 한 가지 제한이 있는데, 사용자와 아이템 아이디의 크기가 `Integer.MAX_VALUE` 보다 크면 안된다는 것이다. 즉 `2147483647`을 초과할 수 없다. 이를 다음과 같이 확인해 볼 수 있다.

```scala
val rawUserArtistData = sc.textFile("/audio/user_artist_data.txt")
rawUserArtistData.map(_.split(' ')(0).toDouble).stats()
rawUserArtistData.map(_.split(' ')(1).toDouble).stats()
```

위 명령어의 수행 결과는 다음과 같으며, 이는 데이터를 다른 변환 없이 그대로 사용해도 무방함을 나타낸다.

```scala
org.apache.spark.util.StatCounter = (count: 24296858, mean: 1947573.265353, stdev: 496000.544975, max: 2443548.000000, min: 90.000000)
org.apache.spark.util.StatCounter = (count: 24296858, mean: 1718704.093757, stdev: 2539389.040171, max: 10794401.000000, min: 1.000000)
```

그리고 추천을 수행하기 위해 아티스트의 데이터를 읽어 이를 기억해야 할 필요가 있다. 다음과 코드를 이용해 아티스트 데이터를 불러올 수 있다.

```scala
val rawArtistData = sc.textFile("/audio/artist_data.txt")
val artistById = rawArtistData.flatMap( line => {
	val (id, name) = line.span(_ != '\t')
	if (name.isEmpty) {
		None
	} else {
		try {
			Some((id.toInt, name.trim))
		} catch {
			case e: NumberFormatException => None
		}
	}
})
```


또한, 앞서 설명하였듯이 각 아티스트가 오타 등의 이유로 다른 텍스트로 표현될 수 있기 때문에 이를 하나로 통합시켜야 한다. `artist_alias.txt` 파일의 각 행은 두 개의 열 `badID \t good ID`로 이루어져 있으며, 해당 파일을 읽어 드라이버 에서 Map 형태로 기억하고 있는다. 이 작업은 다음 코드를 수행함으로써 이뤄진다.

```scala
val rawArtistAlias = sc.textFile("/audio/artist_alias.txt")
val artistAlias = rawArtistAlias.flatMap( line => {
	val tokens = line.split('\t')
	if(tokens(0).isEmpty) {
		None
	} else {
		Some((tokens(0).toInt, tokens(1).toInt))
	}
}).collectAsMap()
```

`artistAlias.get(6803336)`의 결과는 아이디 **1000010**이기 때문에, 다음과 같은 예시를 통해 정상적으로 데이터가 불러와졌는지 확인할 수 있다.

```scala
artistById.lookup(6803336)
artistById.lookup(1000010)
```

위 코드의 수행 결과는 각각 `Aerosmith (unplugged)`와 `Aerosmith`를 나타내며, 이는 정상적인 결과를 의미한다.


## Building a First Model

Spark 의 MLlib에 구현되어 있는 ALS를 사용하기 위해선 두 가지의 변환 과정이 필요하다. 첫번째는 기존에 구한 아티스트의 ID를 앞서 생성한 Map 을 이용하여 같은 ID끼리 묶어야 하며, 데이터를 MLlib의 ALS에서 사용하는 입력 형태인 **Rating** 객체로 변환해야 한다. **Rating**객체는 `사용자ID-ProductID-Value`형태를 갖는 객체인데, 이름은 Rating 이지만 Implicit 형태의 데이터에서도 사용 가능하다. 이 챕터에서는 Value를 ProductID 를 아티스트의 ID, Value를 사용자가 해당 아티스트의 노래를 재생한 횟수로 사용할 것이다. 다음과 같은 코드를 이용하여 추천을 수행하기 위한 데이터를 준비한다.


```scala
import org.apache.spark.mllib.recommendation._

val bArtistAlias = sc.broadcast(artistAlias)
val trainData = rawUserArtistData.map( line => {
	val Array(userId, artistId, count) = line.split(' ').map(_.toInt)
	val finalArtistId = bArtistAlias.value.getOrElse(artistId, artistId)
	Rating(userId, finalArtistId, count)
}).cache()
```

위 코드에서 중요한 부분은 기존에 생성하였던 `artistAlias` Map 을 **broadcast**하는 과정이다. Broadcast를 하지 않는다면 artistAlias 를 Spark가 생성하는 모든 Task 마다 복사하여 사용하게 된다. 하지만 이러한 작업은 큰 비용을 소비한다. 각각의 과정은 최소 몇 메가 바이트 에서 몇십 메가 바이트(크기에 따라 다르며, 이 예시에서의 크기임)를 소비하기 때문에, JVM에서 생성하는 모든 Task 에 이 데이터를 복사한다는 것은 매우 비효율적이다.

따라서 생성한 Map을 Broadcasting 함으로써 Spark Cluster의 각 Executer 가 단 하나의 Map만 유지할 수 있도록 한다. 때문에 Cluster에서 여러 Executer 가 수많은 Task 를 생성할 때 메모리를 효율적으로 관리할 수 있도록 해준다.

그리고 지금까지 계산한 결과를 **cache()**를 통해 메모리에 임시 저장함으로써, `trainData` 변수를 접근할 때마다 map 을 다시 수행하는 것을 막는다.

생성한 변수들을 이용해 다음과 같이 추천 모델을 생성할 수 있다.

```scala
val model = ALS.trainImplicit(trainData, 10, 5, 0.01, 1.0)
```

위 과정은 앞에서 설명한 MatrixFactoriation 방식을 이용해 추천 모델을 생성하는 과정이다. 이때 클러스터의 상태에 따라 수행시간은 몇 분 정도 수행될 수 있다. 그리고 다음 코드를 수행함으로써 내부 Feature 들이 정상적으로 계산되었는지 확인한다(정확한 값인지는 모르지만).

```scala
model.userFeatures.mapValues(_.mkString(", ")).first()
model.productFeatures.mapValues(_.mkString(", ")).first()

...

(Int, String) = (90,-0.8930547833442688, -0.7431690096855164, -0.6351532936096191, -0.28394362330436707, 0.14852239191532135, -0.37798216938972473, -0.923484742641449, -0.12640361487865448, 0.5575262308120728, -0.35868826508522034)
(Int, String) = (2,-0.08458994328975677, 0.027468876913189888, -0.16536176204681396, 0.08694511651992798, 0.019154658541083336, -0.12874850630760193, -0.04696394130587578, -0.0629991888999939, 0.15156564116477966, 0.0011008649598807096)
```

알고리즘이 랜덤성을 갖고 있기 때문에 수행 결과는 위와 다를 수 있다.

<br>
<br>

## Spot Checking Recommendations

이제 실제로 사용자들에게 추천을 잘 수행해 주었는가를 확인해 봐야 한다. 2093760 사용자에 대해 과연 추천을 잘 수행했는지 확인해 볼 것이다.

```scala
val rawArtistsForUser = rawUserArtistData.map(_.split(' ')).filter({
	case Array(user,_,_) => user.toInt == 2093760
})

val existingProducts = rawArtistsForUser.map({
	case Array(_,artist,_) => artist.toInt
}).collect.toSet

artistById.filter({
	case (id, name) => existingProducts.contains(id)
}).values.collect().foreach(println)
```

위 코드의 수행 결과는 다음과 같은 결과를 보이는데,

```
David Gray
Blackalicious
Jurassic 5
The Saw Doctors
Xzibit
```

이 결과는 2093760 사용자가 플레이한 아티스트의 목록이다. 플레이했던 아티스트로 보아, 주로 pop과 hip-hop 음악을 플레이했음을 알 수 있다. (물론 나를 포함한 이 글을 읽는 사람들은 한국인이기 때문에 잘 모를 것이다... 책에서 그렇다고 하니 일단 믿어 보자.) 이러한 정보를 갖고 있는 사용자에게는 어떤 아이템들을 추천 해 주었는가는 다음 코드를 이용해 확인할 수 있다.

```scala
val recommendations = model.recommendProducts(2093760, 5)
recommendations.foreach(println)
```

위 결과는 다음과 같이 상위 5개의 아이템을 추천해 준 결과를 출력한다.

```
Rating(2093760,1300642,0.027983077231064094)
Rating(2093760,2814,0.027609241365462805)
Rating(2093760,1001819,0.027584770801984716)
Rating(2093760,1037970,0.027400202899883735)
Rating(2093760,829,0.027248976510692982)
```

추천의 수행 결과는 앞서 생성하였던 `Rating` 객체를 이용하여 표현된다. Rating 객체에는 (사용자 ID, 아티스트 ID, 값) 형태의 데이터가 존재한다. 이름은 Rating 이지만 세번째 필드의 값이 그대로 평점 값을 나타내는 것은 아님을 주의해야한다. ALS 알고리즘에서는 이 값은 0 과 1 사이의 값을 가지며 값이 높을 수록 좋은 추천을 이야기한다.

다음 코드를 이용해 추천된 결과에서 각각의 아티스트 ID 가 어떤 아티스트인지 이름을 확인해 볼 수 있다.

```scala
val recommendedProductIDs = recommendations.map(_.product).toSet

artistById.filter({
	case (id, name) => recommendedProductIDs.contains(id)
}).values.collect().foreach(println)
```

수행 결과는 다음과 같다. 이 결과는 수행시마다 다를 수 있다.

```
50 Cent
Nas
Kanye West
2Pac
The Game
```

위 목록의 아티스트는 모두 hip-hop 관련 아티스트이다. 얼핏 보기엔 괜찮아 보이지만 너무 대중적인 가수들이며 사용자의 개인적인 성향을 파악하지는 못한 것 같은 결과를 보인다.


<br>
<br>

## Next Post

지금까지는 사용자들의 음악 플레이 기록을 이용하여, 아티스트를 추천해주는 과정을 수행하였다. 다음 포스트에서는 수행한 추천이 얼마나 잘 수행되었는지 평가하는 과정을 진행할 것이다.

<br>
<br>



## References
[1] : [Advanced Analytics with Spark](http://shop.oreilly.com/product/0636920035091.do)