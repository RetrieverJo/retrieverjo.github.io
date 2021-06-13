---
layout:     post
title:      "Spark 1.1.0 설치, Hadoop 2.5.1과의 연동"
subtitle:   ""
category:	Framework
date:       2014-10-24 10:49:00
author:     "Hyunje"
tags: [Spark install, Spark]
image:
  background: white.jpg
  feature: bg2.jpg
  credit: Hyunje Jo
---


이 글은, OS X에 Spark 1.1.0을 설치하고, Hadoop 2.5.1 의 Yarn과 연동시키는 과정에 대한 글입니다.<br>
실제 Hadoop Cluster에 Spark Job을 제출하기 전에 Local 에서 테스트 하기 위한 환경을 구축하기 위해 필요한 작업입니다.

<br>

###WARNING
이 글은 최신버전을 기준으로 설명된 글이 아닙니다. 최신 버전을 대상으로 하였을 때와 설치 과정 혹은 출력 결과가 다를 수 있습니다.

<br>

####Target Version
설치하고 연동하고자 하는 Hadoop과 Spark의 버전입니다.

- Hadoop : 2.5.1
- Spark : 1.1.0

<br>


####Pre-requirements
설치에 앞서 필요한 사항입니다. ```homebrew```등을 이용해 쉽게 설치 가능합니다. <br>
Mavericks 와, Yosemite 에서 테스트 되었습니다.

- JDK7<br>
- Maven 3.x
- Hadoop 2.5.1

Hadoop의 설치 과정은 [OS X에 Hadoop2.5.1 설치하기](http://hyunje.com/post/os-xe-hadoop2-dot-5-1-seolcihagi/)를 참고해 주시기 바랍니다.

<br>

####Notation
이 문서에서 사용될 용어들에 대한 정의입니다.

- ${HADDOP_DIR} : Hadoop이 설치된 위치
- ${SPARK_DIR} : Spark 압축 해제 위치

<br>



## I. Spark 설치

#### Spark 1.1.0 다운로드
[Spark 다운로드 페이지](http://spark.apache.org/downloads.html)에서 다음과 같은 옵션으로 Spark를 다운로드합니다.

* Version : 1.1.0
* Package Type : Source Code (Hadoop 2.5.1에 맞는 Pre-build Version이 없으므로)
* Download Type : Direct

그리고 다운로드 한 파일을 압축해제합니다.

<br>

#### Spark 빌드
${SPARK_DIR}에서 다음 명령어를 차례로 입력합니다. (15분 가량 소요됩니다.)

```bash
$ export MAVEN_OPTS="-Xmx2g -XX:MaxPermSize=512M -XX:ReservedCodeCacheSize=512m"
$ mvn -Pyarn -Phadoop-2.4 -Dhadoop.version=2.5.1 -DskipTests clean package
```

``-Phadoop`` 옵션을2.4로 하는 이유는, 해당 옵션을 2.5.1로 설정하여 빌드하면, Spark에서 Hadopo 2.5.1에 해당하는 Profile이 없다는 다음과 같은 메시지를 출력한다.

```bash
[WARNING] The requested profile "hadoop-2.5" could not be activated because it does not exist.
```

하지만, [이 페이지](http://mail-archives.apache.org/mod_mbox/spark-user/201410.mbox/%3CCAEYhXbV+kGipfDhsV8Kt_pZKOtcMkoJJrgi3rH0eWt1PnPb+Cg@mail.gmail.com%3E) 를 보면, 2.4를 그대로 쓰면 2.4+에 적용되므로 괜찮다는 얘기가 있다. 그러므로, 2.4 옵션을 사용한다.

<br>



## II. Hadoop과의 연결

#### Spark 설정
`${SPARK_DIR}/conf/spark-env.sh.template` 을 복사하여 같은 경로에 `spark-env.sh` 파일을 생성합니다.<br>
파일을 열고, 다음 변수를 추가합니다.

```bash
export HADOOP_CONF_DIR=${HADOOP_DIR}/etc/hadoop
```

<br>

#### Yarn-client와 연결
다음 명령어가 정상적으로 수행되는지 확인합니다.<br>
단, 수행하기 이전에 Hadoop 2.5.1 과 Yarn이 정상적으로 수행되고 있어야합니다.

```bash
$ ${SPARK_DIR}/bin/spark-shell --master yarn-client
```
중간에 에러 메시지 없이 

```bash
scala>
```
가 출력된다면 정상적으로 연결이 된 상태입니다.
