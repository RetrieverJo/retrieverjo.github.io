---
layout:     post
title:      "OS X에 Hadoop 2.5.1 설치하기"
subtitle:   ""
category:   Framework
date:       2014-10-24 04:19:00
author:     "Hyunje"
tags: [hadoop install, hadoop]
image:
  background: white.jpg
  feature: bg1.jpg
  credit: Hyunje Jo
---


이 글은, OS X 에 Hadoop2.5.1을 설치하고, 정상적으로 설치가 되었는지 테스트 하는 방법에 대한 글입니다.<br>
실제 Cluster에 Hadoop 관련 Job을 제출하기 전에 Local에서 테스트 하기 위한 환경을 구축하기 위해 필요한 작업입니다.

### WARNING
이 글은 최신버전을 기준으로 설명된 글이 아닙니다. 최신 버전을 대상으로 하였을 때와 설치 과정 혹은 출력 결과가 다를 수 있습니다.

<br>

#### Target Version
설치하고, 연동하고자 하는 Hadoop의 버전은 `2.5.1`입니다.



#### Pre-requirements
설치에 앞서 필요한 사항입니다. ```homebrew```등을 이용해 쉽게 설치 가능합니다.
Mavericks 와, Yosemite 에서 테스트 되었습니다.

- JDK7
- Maven 3.x
<br>
<br>



#### Notation
이 문서에서 사용될 용어들에 대한 정의입니다.

- ${HADDOP_DIR} : Hadoop 압축 해제 위치

<br>




## I. Hadoop 설치

#### Hadopo 2.5.1 다운로드
[Hadoop 다운로드 페이지](http://apache.tt.co.kr/hadoop/common/)에서 hadoop-2.5.1.tar.gz 를 다운로드 하고 압축을 해제합니다.

<br>

#### hadoop-env.sh 설정
`${HADOOP_DIR}/etc/hadoop/hadoop-env.sh` 파일에 다음 두 가지를 설정해야 합니다.

1. JAVA\_HOME : Java가 설치된 경로
2. HADOOP\_PREFIX : Hadoop이 설치된 경로, 즉 ${HADOOP_DIR}

해당 변수가 있는 부분을 주석 해제 후, 적절한 경로를 입력하면 됩니다.<br>
[Oracle 에서 배포하는 JDK7](http://www.oracle.com/technetwork/java/javase/downloads/jdk7-downloads-1880260.html)를 통해서 JDK7을 설치한 경우 `/Library/Java/JavaVirtualMachines/jdk1.7_x` 가 기본 경로입니다.

<br>

#### core-site.xml 설정

`${HADOOP_DIR}/etc/hadoop/core-site.xml`파일을 다음과 같이 설정합니다.

```xml
<configuration>
        <property>
                <name>fs.defaultFS</name>
                <value>hdfs://localhost:9000</value>
        </property>
</configuration>
```

<br>

#### hdfs-site.xml 설정

`${HADOOP_DIR}/etc/hadoop/hdfs-site.xml`파일을 다음과 같이 설정합니다.

```xml
<configuration>
        <property>
                <name>dfs.replication</name>
                <value>1</value>
        </property>
        <property>
                <name>dfs.namenode.name.dir</name>
                <value>네임노드 정보가 저장될 경로</value>
        </property>
        <property>
                <name>dfs.datanode.data.dir</name>
                <value>데이터노드 정보가 저장될 경로</value>
        </property>
</configuration>
```

<br>

#### mapred-site.xml 설정

`${HADOOP_DIR}/etc/hadoop/mapred-site.xml`파일을 다음과 같이 설정합니다.

```xml
<configuration>
        <property>
                <name>mapreduce.framework.name</name>
                <value>yarn</value>
        </property>
</configuration>
```

<br>

#### yarn-site.xml 설정

`${HADOOP_DIR}/etc/hadoop/yarn-site.xml`파일을 다음과 같이 설정합니다.

```xml
<configuration>
        <property>
                <name>yarn.nodemanager.aux-services</name>
                <value>mapreduce_shuffle</value>
        </property>
</configuration>
```

<br>

#### SH 인증키 생성
콘솔창에서 다음 명령어를 수행합니다.

```bash
$ ssh-keygen -t dsa -P '' -f ~/.ssh/id_dsa
$ cat ~/.ssh/id_dsa.pub >> ~/.ssh/authorized_keys
```

<br>



## II.Hadoop 설치 결과 테스트
설치한 Hadoop이 정상적으로 설치되었는지 확인합니다.

#### Namenode format
다음 명령어를 수행합니다.

```bash
${HADOOP_DIR}/bin/hdfs namenode -format
```

콘솔에 출력된 결과 중,

```
common.Storage: Storage directory "hdfs-site.xml에서 설정한 경로" has been successfully formatted
```
메시지를 확인합니다.

<br>

#### DFS(Distributed File System) 실행
다음 명령어를 수행합니다.

```bash
${HADOOP_DIR}/sbin/start-dfs.sh
```
다음 페이지에서 Live Nodes 항목이 1로 표시되는지 확인합니다.

```
http://localhost:50070
```

<br>

#### YARN 실행
다음 명령어를 수행합니다.

```bash
${HADOOP_DIR}/sbin/start-yarn.sh
```
다음 페이지에서 Active Nodes 항목이 1로 표시되는지 확인합니다.

```
http://localhost:8088
```

<br>



## III. 예제 수행
다음 명령어들이 수행되는 기본 경로는 `${HADOOP_HOME}`이라 가정합니다.

```bash
$ bin/hdfs dfs -put etc/hadoop /input
$ bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.5.1.jar grep /input /output 'dfs[a-z.]+'
$ bin/hdfs dfs -cat /output/p*
```
위 명령어의 최종 수행 결과가 다음과 같은 텍스트를 출력하였다면, 정상적으로 수행된 것입니다.

```
6   dfs.audit.logger
4   dfs.class
3   dfs.server.namenode.
2   dfs.audit.log.maxbackupindex
2   dfs.period
2   dfs.audit.log.maxfilesize
1   dfsmetrics.log
1   dfsadmin
1   dfs.servers
1   dfs.replication
1   dfs.file
1   dfs.datanode.data.dir
1   dfs.namenode.name.dir
```

<br>

#### 참고 사항
Hadoop 2.5.1에서 사용되는 Native Library 는 Mac OS X Platform 에서 사용이 불가능합니다. 때문에 Hadoop 관련 명령어를 수행할 때 마다 Native Library를 사용할 수 없다는 경고 메시지가 출력됩니다.
