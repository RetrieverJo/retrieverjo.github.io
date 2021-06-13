---
layout:     post
title:      "Spark & 머신 러닝 - Introduction to Spark"
subtitle:   "Advanced Analytics with Spark, 1. Introduction to Spark"
category:   Data Analysis
date:       2015-07-04 15:09:00
author:     "Hyunje"
tags: [Spark, Machine Learning]
image:
  background: white.jpg
  feature: bg1.jpg
  credit: Hyunje Jo
keywords:	"데이터 분석, Data Analysis, Spark, Apache Spark, 머신 러닝, Machine Learning"
---


이 글에서는 Spark가 어떤 형태로 동작하는지 설명하고 간단한 예시를 통해 Spark에 적응하는 과정을 설명한다.

이 포스트는 [Advanced Analytics with Spark](http://shop.oreilly.com/product/0636920035091.do)을 정리한 글이다.

이 글에서 다루고자 하는 내용은 Chapter 2이다. Chapter 1은 빅데이터에 대한 개략적인 얘기와, 왜 Spark가 뜨고 있는지, Spark 가 데이터 분석에서 어떠한 역할을 하고 있는지에 대한 설명이 있었다. 그 내용들은 다른 자료들에 많이 있으므로 따로 정리는 하지 않았다.

<br>
<br>

## Record Linkage

chapter 2에서는 Record Linkage와 비슷한 작업(ex : [ETL](https://en.wikipedia.org/wiki/Extract,_transform,_load))들을 Spark 로 어떻게 수행하는지에 대해 설명하고, Follow-up 할 수 있도록 하고있다.

실제 데이터를 이용해 분석을 수행할 때 대부분은 수집한 데이터를 그대로 이용하지 않는다. 수집한 데이터를 그대로 이용한다면 잘못된 데이터(값이 비어있거나, 필요없는 데이터가 섞여있거나, 같은 데이터가 중복되어 들어있거나 등등)들이 분석에 그대로 활용되기 때문에 이들을 잘 필터링 해야 한다.

이 책에서 Recoed Linkage는 위의 문제 중 같은 데이터가 다른 형태로 들어있을 때, 그것을 하나의 데이터로 간주하도록 하는 것이라 얘기하고 있다.

이러한 내용들을 수행하기 위해 Record Linkage Comparison Patterns Dataset을 이용한다.

<br>
<br>

## The Spark Shell and SparkContext

Spark를 이용하는 방법은 크게 두 가지가 있다. 하나는 Spark Shell 을 통해서 REPL(read-eval-print loop) 형태로 Spark를 이용하는 방법이고, 나머지 방법은 Spark Application 을 IDE를 이용해 작성한 후, 그것을 패키징 하여 Spark Cluster로 Submit하여 수행하는 방법이다.

REPL을 이용하려면 다음과 같은 명령어를 이용해야한다.

(여기서, 개인적인 공부이기 때문에 로컬 클러스터에서 수행함을 가정한다. 또한, Shell은 Scala를 기반으로 작성해야 하기 때문에 Scala 에 대한 이해가 필요하다.)

```bash
$ spark-shell --master local[2]
```
만약 Spark 를 YARN을 이용해 수행시키고 싶다면 다음과 같이 입력한다.

```bash
$ spark-shell --master yarn-client
```
<br>

이 챕터에서는 [http://bit.ly/1Aoywaq](http://bit.ly/1Aoywaq) 링크의 데이터를 이용하고 있다.
이 데이터를 HDFS로 업로드해야 하기 때문에, 다음 과정을 이용해 데이터를 HDFS로 업로드 한다.

```bash
$ wget http://bit.ly/1Aoywaq -O donation.zip
$ unzip donation.zip
$ unzip 'block_*.zip'
$ hdfs dfs -mkdir /linkage
$ hdfs dfs -put block_*.csv /linkage
```

책에서는 (아직은) Spark-shell을 기준으로 설명하고 있다. 본 글 역시 다른 언급이 있지 않는 이상 Spark-shell 을 기준으로 설명할 것이다.

우선, 다음 명령어를 이용해 데이터가 HDFS에 정상적으로 업로드 되고, 그것을 잘 읽어 오는지 확인한다.

```scala
val rawblocks = sc.textFile("/linkage")
...
rawblocks: org.apache.spark.rdd.RDD[String] = MapPartitionsRDD[1] at textFile at <console>:21
```

Spark Shell 에서는 기본적인 SparkContext 객체에 대한 인스턴스를 하나 제공한다. 그 인스턴스에 대한 접근은 ```sc```로 할 수 있으며, **textFile** 함수를 이용해 HDFS에 저장되어 있는 파일을 읽어올 수 있다.

다음 명령어를 이용해 데이터의 상위 10 줄에 어떤 데이터가 들어있는지 확인한다.

```scala
val head = rawblocks.take(10)
head.foreach(println)
```

그러면 다음과 같은 값이 들어있음을 알 수 있다.

```
"id_1","id_2","cmp_fname_c1","cmp_fname_c2","cmp_lname_c1","cmp_lname_c2","cmp_sex","cmp_bd","cmp_bm","cmp_by","cmp_plz","is_match"
37291,53113,0.833333333333333,?,1,?,1,1,1,1,0,TRUE
39086,47614,1,?,1,?,1,1,1,1,1,TRUE
70031,70237,1,?,1,?,1,1,1,1,1,TRUE
84795,97439,1,?,1,?,1,1,1,1,1,TRUE
36950,42116,1,?,1,1,1,1,1,1,1,TRUE
42413,48491,1,?,1,?,1,1,1,1,1,TRUE
25965,64753,1,?,1,?,1,1,1,1,1,TRUE
49451,90407,1,?,1,?,1,1,1,1,0,TRUE
39932,40902,1,?,1,?,1,1,1,1,1,TRUE
```

이 결과를 보면, csv 데이터의 제일 첫 줄에 각 컬럼이 나타내는 값이 어떤 것인지에 대한 정보가 있다. 이것을 다음과 같은 함수와 명령어를 이용하여 필터링한다.

```scala
def isHeader(line:String) : Boolean = {
	line.contains("id_1")
}

val noheader = rawblocks.filter(x => !isHeader(x))
```

그 결과로, 다음과 같이 컬럼명이 제외된 데이터 10개를 볼 수 있다.

```
37291,53113,0.833333333333333,?,1,?,1,1,1,1,0,TRUE
39086,47614,1,?,1,?,1,1,1,1,1,TRUE
70031,70237,1,?,1,?,1,1,1,1,1,TRUE
84795,97439,1,?,1,?,1,1,1,1,1,TRUE
36950,42116,1,?,1,1,1,1,1,1,1,TRUE
42413,48491,1,?,1,?,1,1,1,1,1,TRUE
25965,64753,1,?,1,?,1,1,1,1,1,TRUE
49451,90407,1,?,1,?,1,1,1,1,0,TRUE
39932,40902,1,?,1,?,1,1,1,1,1,TRUE
46626,47940,1,?,1,?,1,1,1,1,1,TRUE
```
<br>
<br>

## Structuring Data with Tuples and Case Classes

지금까지 읽은 데이터를 그대로 읽어서 분석에 활용할 수 있지만, 데이터를 파싱하여 활용하면 더욱 쉽게 활용할 수 있다.

데이터는 다음과 같은 형태를 띄고 있다.

* 처음 2개의 Integer 값 : 레코드에서 매칭되는 환자의 ID
* 9개의 Double 값 : 9가지의 필드에 대한 매칭 스코어(없을 수 있음)
* Boolean 값 : 매치 되는지 여부에 대한 판별결과

이를 파싱하기 위해 다음과 같은 [Case Class](http://docs.scala-lang.org/ko/tutorials/tour/case-classes.html)를 정의하고 필요한 서브 함수를 작성하여 활용한다.

```scala
case class MatchData(id1: Int, id2: Int, scores: Array[Double], matched: Boolean)

def toDouble(s: String) = {
	if("?".equals(s)) Double.NaN else s.toDouble
}

def parse(line: String) = {
	val pieces = line.split(',');
	val id1 = pieces.apply(0).toInt
	val id2 = pieces.apply(1).toInt
	val scores = pieces.slice(2, 11).map(toDouble)
	val matched = pieces.apply(11).toBoolean
	MatchData(id1, id2, scores, matched)
}
```

다음 명령어를 통해 데이터를 파싱하고, 그 결과를 확인한다.

```scala
val parsed = noheader.map(line => parse(line))

parsed.take(10).foreach(println)
```

그리고, 지금까지 파싱한 결과를 메모리에 cache()시켜놓는다.

```scala
parsed.cache()
```
<br>
<br>

## Creating Histograms

[Histogram](https://en.wikipedia.org/wiki/Histogram) 은 간단히 말하면, 항목별로 개수를 센 결과를 나타낸다고 할 수 있다. 이 절에서는 지금까지 파싱한 데이터가 `matched` 필드 값의 종류(true, false)별로 얼마나 존재하는지에 대한 히스토그램을 생성할 것이다. 다행히도 Spark의 RDD에서 기본적으로 제공하는 **countByValue**를 이용하면 쉽게 해결할 수 있다.

다음 명령어를 통해 각 값 별로 얼마만큼의 레코드가 존재하는지 쉽게 카운팅 할 수 있다.

```scala
val matchCounts = parsed.map(md => md.matched).countByValue()
```

다음과 같은 수행 결과로, 손쉽게 `matched` 필드의 각 값 별 히스토그램을 생성할 수 있다.

```scala
matchCounts: scala.collection.Map[Boolean,Long] = Map(true -> 20931, false -> 5728201)
```

<br>
<br>

## Summary Statistics for Continuous Variables

앞서 설명된 **countByValue**는 값의 종류에 따라 Histogram 을 생성하는 좋은 방법 중 하나이다. 하지만 Boolean 형태의 값처럼 적은 범위를 갖는 값이 아니라 Continuous Variable, 즉 연속변수와 같은 경우에 사용하기에는 적절하지 않다.

연속변수들에 대해서는 모든 값의 Histogram 을 구하는 것보다 분포에 대한 확률적인 통계 수치(평균, 표준편차, 최대 or 최솟값 등)를 보는 것이 좀 더 간결하게 데이터를 파악할 수 있다.

Spark에서는 이를 위해 **stats**라는 함수를 제공한다. 이 함수를 이용함으로써 손쉽게 특정 변수에 대한 통계적 수치를 출력할 수 있다. 파싱한 값중에 NaN이 들어가 있을 수 있기 때문에, 해당 레코드는 필터링을 수행한 후에 수행시키도록 한다.

```scala
import java.lang.Double.isNaN
parsed.map(md => md.scores(0)).filter(!isNaN(_)).stats()
```

위 코드의 수행 결과로 다음과 같은 결과를 볼 수 있다.

```
org.apache.spark.util.StatCounter = (count: 5748125, mean: 0.712902, stdev: 0.388758, max: 1.000000, min: 0.000000)
```

또한 Scala의 **Range**를 이용하여 scores 배열에 들어있는 모든 변수에 대한 수치값들에 대한 통계치를 구할 수 있다.

```scala
val stats = (0 until 9).map(i => {
	parsed.map(md => md.scores(i)).filter(!isNaN(_)).stats()
})
stats.foreach(println)
```

그 결과는 다음과 같다.

```
(count: 5748125, mean: 0.712902, stdev: 0.388758, max: 1.000000, min: 0.000000)
(count: 103698, mean: 0.900018, stdev: 0.271316, max: 1.000000, min: 0.000000)
(count: 5749132, mean: 0.315628, stdev: 0.334234, max: 1.000000, min: 0.000000)
(count: 2464, mean: 0.318413, stdev: 0.368492, max: 1.000000, min: 0.000000)
(count: 5749132, mean: 0.955001, stdev: 0.207301, max: 1.000000, min: 0.000000)
(count: 5748337, mean: 0.224465, stdev: 0.417230, max: 1.000000, min: 0.000000)
(count: 5748337, mean: 0.488855, stdev: 0.499876, max: 1.000000, min: 0.000000)
(count: 5748337, mean: 0.222749, stdev: 0.416091, max: 1.000000, min: 0.000000)
(count: 5736289, mean: 0.005529, stdev: 0.074149, max: 1.000000, min: 0.000000)
```

<br>
<br>

## Creating Reusable Code for Computing Summary Statistics

하지만, 위와 같은 작업은 매우 비효율적이다. scores 배열에 존재하는 값에 대한 각각의 통계치를 계산하기 위해서 `parsed RDD`를 매번 다시 계산해야 한다. 물론 앞선 과정에서 `parsed RDD` 를 **cache** 해 놓긴 하였지만, 데이터가 많아지면 많아질 수록 이 작업의 소요시간은 급속도로 증가할 것이다.

이러한 경우에, 어떠한 ```RDD[Array[Double]]```를 인자로 받아, 값이 정상적으로 들어있는 레코드들에 대한 각 인덱스별 `StatCounter` 를 갖는 클래스 혹은 함수를 작성하는 것을 생각해 볼 수 있다.

또한, 이러한 작업이 분석 과정에서 반복될 때, 매번 해당 코드를 새롭게 작성하는 것보다 다른 파일에 작성하여 그것을 재사용하는 것이 적절한 방식이다. 때문에 다른 파일에 스칼라 코드를 작성하고, Spark에서 그 파일을 불러와 사용하도록 할 것이다. 다음 소스코드를 다른 파일 `StatsWithMissing.scala`파일에 저장한 후 사용할 것이며, 멤버변수와 함수에 대해서는 코드의 뒷 부분에서 설명할 것이다.

```scala
import org.apache.spark.util.StatCounter

class NAStatCounter extends Serializable{
	val stats: StatCounter = new StatCounter()
	var missing: Long = 0
	
	def add(x: Double): NAStatCounter = {
		if(java.lang.Double.isNaN(x)) {
			missing += 1
		} else {
			stats.merge(x)
		}
		this
	}
	
	def merge(other: NAStatCounter): NAStatCounter = {
		stats.merge(other.stats)
		missing += other.missing
		this
	}
	
	override def toString = {
		"stats: " + stats.toString + " NaN: " + missing
	}
}

object NAStatCounter extends Serializable {
	def apply(x: Double) = new NAStatCounter().add(x)
}
```

앞서 정의한 `NAStatCounter` 클래스는 두 개의 멤버 변수를 갖고 있다. `stats`로 정의된 `StatCounter` 인스턴스는 immutable이고, `missing`으로 정의된 `Long` 변수는 mutable 변수이다. 이 클래스를 `Serializable` 객체를 상속시킨 이유는, Spark 의 RDD에서 이 객체를 사용하기 위해선 반드시 상속을 시켜주어야 한다. 만약 이 상속을 하지 않으면 RDD에서 에러가 발생한다.

클래스의 첫 번째 함수 `add`는 새로운 Double 형태의 값을 받아 `stats`변수가 값을 계속 관측할 수 있도록 한다. 만약 인자로 받은 값이 NaN이면, missing 값을 1 증가시키고, NaN이 아니라면 StatCounter 객체에 기록한다. 두 번째 함수 `merge`는 다른 NAStatCounter 인스턴스를 매개변수로 받아 지금의 인스턴스와 병합시키는 역할을 한다. 세 번째 함수 toString은 쉽게 NAStatCounter 클래스를 출력하기 위해서 오버라이딩 한 것이다. 스칼라에서는 부모 객체의 함수를 오버라이딩 하기 위해선 반드시 함수 앞에 `override` 키워드를 추가해야 한다.

그리고 class 정의와 함께 NAStatCounter 객체에 대한 `companion object`를 함께 정의한다. 스칼라에서 object 키워드는 자바에서의 static method 와 같이 어떤 클래스에 대한 helper method를 제공하는 싱글톤 객체를 선언하는데에 이용된다. 이 경우에서처럼 class 이름과 같은 object를 선언하는 것을 `companion object`를 선언한다고 하며, 여기서의 `apply` 함수는 `NAStatCounter` 클래스에 대한 새 인스턴스를 생성하고, 그 인스턴스를 반환하기 전에 Double 값을 더한다.

정상적으로 로드가 되었다면 다음과 같은 메시지가 출력된다.

```
import org.apache.spark.util.StatCounter
defined class NAStatCounter
defined module NAStatCounter
warning: previously defined class NAStatCounter is not a companion to object NAStatCounter.
Companions must be defined together; you may wish to use :paste mode for this.
```

경고가 출력되어 문제가 생긴것이 아닌가 할 수 있지만, 무시할 수 있는 경고이다.
다음과 같은 예시로 정상적으로 로드되었는지 확인해 볼 수 있다.

```scala
val nas1 = NAStatCounter(10.0)
nas1.add(2.1)
val nas2 = NAStatCounter(Double.NaN)
nas1.merge(nas2)
```

이제 작성한 `NAStatCounter`클래스를 이용하여 `parsed RDD`에 들어있는 MatchData 레코드를 처리하자. 각각의 MatchData 인스턴스는 Array[Double] 형태의 매칭 스코어를 포함하고 있다. 배열에 있는 각각의 엔트리마다 `NAStatCounter` 객체를 생성시켜, 모든 값을 추적하려 한다. 그렇게 하기 위해선 다음과 같은 방식으로 RDD 안에 존재하는 모든 레코드는 Array[Double]을 갖고 있기 때문이 이것을 Array[NAStatCounter]를 갖는 RDD로 변경하면 된다.

```scala
val nasRDD = parsed.map(md => {
	md.scores.map(d => NAStatCounter(d))
})
```

이제, 여러개의 Array[NAStatCounter] 를 하나의 배열로 합치면서 각 인덱스별로 존재하는 NAStatCounter를 병합하면 된다. 이를 위해 다음과 같은 코드를 이용한다.

```scala
val reduced = nasRDD.reduce((n1, n2) => {
	n1.zip(n2).map { case (a, b) => a.merge(b) }
})
reduced.foreach(println)
```

그 결과로 다음과 같이 모든 매칭 스코어에 대한 인덱스별 통계 수치를 구할 수 있다.

```
stats: (count: 5748125, mean: 0.712902, stdev: 0.388758, max: 1.000000, min: 0.000000) NaN: 1007
stats: (count: 103698, mean: 0.900018, stdev: 0.271316, max: 1.000000, min: 0.000000) NaN: 5645434
stats: (count: 5749132, mean: 0.315628, stdev: 0.334234, max: 1.000000, min: 0.000000) NaN: 0
stats: (count: 2464, mean: 0.318413, stdev: 0.368492, max: 1.000000, min: 0.000000) NaN: 5746668
stats: (count: 5749132, mean: 0.955001, stdev: 0.207301, max: 1.000000, min: 0.000000) NaN: 0
stats: (count: 5748337, mean: 0.224465, stdev: 0.417230, max: 1.000000, min: 0.000000) NaN: 795
stats: (count: 5748337, mean: 0.488855, stdev: 0.499876, max: 1.000000, min: 0.000000) NaN: 795
stats: (count: 5748337, mean: 0.222749, stdev: 0.416091, max: 1.000000, min: 0.000000) NaN: 795
stats: (count: 5736289, mean: 0.005529, stdev: 0.074149, max: 1.000000, min: 0.000000) NaN: 12843
```

또한 다음과 같이 `statsWithMissing`함수를 정의하고 이를 사용함으로써 좀 더 고급지게(?) 처리할 수 있다.

```scala
def statsWithMissing(rdd: RDD[Array[Double]]): Array[NAStatCounter] = {
	val nastats = rdd.mapPartitions((iter: Iterator[Array[Double]]) => {
		val nas: Array[NAStatCounter] = iter.next().map(d => NAStatCounter(d))
		iter.foreach(arr => {
			nas.zip(arr).foreach { case (n, d) => n.add(d)}
		})
		Iterator(nas)
	})
	nastats.reduce((n1, n2) => {
		n1.zip(n2).map { case (a, b) => a.merge(b)}
	})
}
```

<br>
<br>

## Simple Variable Selection and Scoring

앞서 작성한 `statsWithMissing` 함수를 이용해 다음과 같이 parsedRDD 로부터 각 데이터가 매치되는지 여부에 따라 scores의 분포가 어떻게 되는지 구할 수 있다.

```scala
val statsm = statsWithMissing(parsed.filter(_.matched).map(_.scores))
val statsn = statsWithMissing(parsed.filter(!_.matched).map(_.scores))
```

두 변수 `statsm`과 `statsn`은 전체 데이터를 매치되는지 여부에 따라 두 개의 서브셋으로 나누어 scores의 확률적 분포에 대한 정보를 갖고 있다. 여기서 두 개의 각 서브셋에 존재하는 scores 배열의 각 칼럼 별 정보를 비교함으로써, 두 서브셋 차이에 각 feature 별로 어떤 차이를 갖고 있는지를 비교할 수 있다.

```scala
statsm.zip(statsn).map { case(m, n) =>
	(m.missing + n.missing, m.stats.mean - n.stats.mean)
}.zipWithIndex.foreach(println)
```

위 코드의 수행 결과로 다음과 같은 결과가 나온다.

```
((1007,0.285452905746686),0)
((5645434,0.09104268062279874),1)
((0,0.6838772482597568),2)
((5746668,0.8064147192926266),3)
((0,0.03240818525033473),4)
((795,0.7754423117834044),5)
((795,0.5109496938298719),6)
((795,0.7762059675300523),7)
((12843,0.9563812499852178),8)
```

좋은 feature 는 두 가지의 속성이 있다. 하나는 그 feature에 따른 크기가 큰 차이가 존재하는 것이고, 모든 데이터 쌍(두 서브셋 사이의)에 대해서도 균일하게 발생한다는 것이다. 이러한 이론에 따라 인덱스 1의 feature 는 좋은 feature라고 할 수 없다. 값이 존재하지 않아 missing이 카운트 된 횟수가 매우 많으며, 두 서브셋의 평균값의 차이가 0.09로 매우 적다(값의 범위가 0 에서 1인 것임을 감안했을 때). Feature 4 또한 적절치 않다. Feature 4는 모든 값이 존재하지만 평균 값의 차이가 0.03이므로 두 데이터 그룹 사이에 별 차이가 없기 때문이다.

반면, feature 5와 7은 훌륭한 feature 이다. 대부분의 데이터에 대해 값이 존재하며, 두 그룹의 평균 차이가 매우 크기 때문이다. 그리고 feature 2, 6, 8 역시 괜찮은 feature 라 할 수 있다. Feature 0과 3은 좀 애매하다고 할 수 있다. Feature 0은 대부분의 데이터에서 관측 가능하지만 두 셋의 평균 차이가 크지 않고, 반대로 Feature 3은 두 셋의 평균 차이가 크지만 많은 데이터에서 관측하기가 어렵다. 이 정보들은 두 데이터 셋을 명확하게 표현하기가 어렵다.

앞서 설명한 내용을 바탕으로하여 쓸만한 feature(2, 5, 6, 7, 8)를 이용해 각 데이터에 대한 scoring model을 만들 것이다. 이 모델에서는 NaN 값은 0으로 처리하여 계산할 것이다.

```scala
def naz(d: Double) = if (Double.NaN.equals(d)) 0.0 else d
case class Scored(md: MatchData, score: Double)
val ct = parsed.map(md => {
	val score = Array(2, 5, 6, 7, 8).map(i => naz(md.scores(i))).sum
	Scored(md, score)
})
ct.map(s => s.md.matched).countByValue()

...

scala.collection.Map[Boolean,Long] = Map(true -> 20931, false -> 5728201)
```

생성한 `ct RDD` 에 여러 Threshold 값들을 지정하고, 매치되었는지 여부에 따라 카운팅을 함으로써 데이터의 속성에 대해 파악할 수 있다.

다음 결과는 Threshold 를 4.0 으로 정하였는데, 이것은 각각의 feature 에 대해 평균적으로 0.8 이상을 갖고 있음을 의미한다.

```scala
ct.filter(s => s.score >= 4.0).map(s => s.md.matched).countByValue()

...

scala.collection.Map[Boolean,Long] = Map(true -> 20871, false -> 637)
```

위와 같은 결과는 matched에 속하는 데이터 중 true를 갖는 데이터들의 99%이상이 feature(2, 5, 6, 7, 8) 합이 4 이상임을 나타내고, false를 갖는 데이터는 대부분(98.8%)이 4.0 이하의 값을 갖는다는것을 얘기한다.

다음과 같이 score 값을 2.0 으로 필터링 하면 다음과 같은 결과를 얻을 수 있다.

```scala
ct.filter(s => s.score >= 2.0).map(s => s.md.matched).countByValue()

...

scala.collection.Map[Boolean,Long] = Map(true -> 20931, false -> 596414)
```

이 결과는 matched 값이 true 인 경우에는 모든 데이터에가 feature 의 합이 2.0 이상이라는 것과, matched 값이 false인 경우에는 여전이 90% 이상이 feature 합이 2.0 미만이라는 것을 알 수 있다.

앞의 예시에서는 매우 간단한 특정 feature들의 합으로 matched 값에 따라 분류된 두 데이터 셋의 특성에 대해 알아봤지만, 이를 다양하게 변화시킴으로써 주어진 데이터셋에 대해 또 다른 새로운 정보들을 얻을 수 있을 것이다.

지금까지 Spark를 사용해보고, 이를 이용해 간단한 데이터셋을 필터링하고, 데이터셋의 특성에 대해 파악해보았다. 다음 장부터는 또 다른 데이터 셋을 이용해 좀 더 깊이있는 분석을 해 볼 것이다.