---
layout:     post
title:      "Spark & 머신 러닝 - Overview"
subtitle:   "Advanced Analytics with Spark, 0. Overview"
category:   Data Analysis
date:       2015-07-01 17:09:00
author:     "Hyunje"
tags: [Spark, Machine Learning]
image:
  background: white.jpg
  feature: bg4.jpg
  credit: Hyunje Jo
keywords:	"데이터 분석, Data Analysis, Spark, Apache Spark, 머신 러닝, Machine Learning"
---

이 글들은 [Advanced Analytics with Spark](http://shop.oreilly.com/product/0636920035091.do)를 공부하면서 정리한 글이다.

Advanced Analytics with Spark는 2015년 4월에 발매된 책으로, Spark 를 이용해서 데이터 분석을 수행하는 방법과 예시에 대해 작성되어 있다.

Spark에 대한 기초적인 내용부터 시작하여 여러 오픈되어 있는 데이터셋을 이용해 데이터 분석을 수행하는 과정이 담겨있다. Spark에서 제공하는 mllib과 graphX 등을 사용하여 분석을 수행한다.

책에서는 Spark 버전은 1.3을 기준으로 하고 있지만, ~~최근 1.4가 공개되었으므로 1.4로 진행할 것이다.~~ 1.5로 진행할 것이다.

진행하는데에 앞서 필요한 사항과 개발 환경은 다음과 같다.
아직 책의 초반을 진행하고 있어서 IDE를 사용할 일이 적은데, 사용하게 된다면 IntelliJ IDEA를 사용할 것이다.

<br>

### Pre-requirements
* Hadoop 2.6+
* Spark 1.4
* Spark와 Hadoop에 대한 기본적인 지식


### Environment
* Scala 2.10
* Intellij IDEA 14 (사용하게 되면)


### Target
* 제대로 된 Spark의 사용법
* 데이터 분석 경험

<br>
<br>
아직까지 내가 겪어본 Spark는 단순하게 예제를 돌려보거나, 혼자의 감에 의지한 주먹구구식의 분석 뿐이었다. 이 책을 따라 공부하면서 제대로 Spark를 사용해 보고 데이터 분석을 경험해 볼 것이다. 그리고 그 경험을 이용해 내가 진행하였던 몇몇 분석 결과와 코드 등을 다시 돌아보고, 수정하여 말끔하게 다듬을 것이다.

또한 진행 도중에 괜찮은 아이디어가 떠오르면 그것을 직접 구현하고 결과를 공개 할 계획이다.