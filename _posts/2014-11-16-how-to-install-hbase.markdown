---
layout:     post
title:      "HBase Pseudo-distributed Mode 설치법"
subtitle:   ""
category:	Framework
date:       2014-11-16 9:59:00
author:     "Hyunje"
tags: [HBase]
image:
  background: white.jpg
  feature: bg3.jpg
  credit: Hyunje Jo
---


이 문서는 OS X 에서 HBase를 설치하고, Hadoop2 와 연동하는 방법에 대한 글입니다.<br>
Pseudo Distributed Mode로 설치하고 실행하는 과정입니다.<br>
Java Client API 를 이용하여 간단한 테스트까지 수행합니다.


### WARNING
이 글은 최신버전을 기준으로 설명된 글이 아닙니다. 최신 버전을 대상으로 하였을 때와 설치 과정 혹은 출력 결과가 다를 수 있습니다.

<br>

### Install Environments
설치 과정에서 사용된 여러 환경에 관한 내용입니다.

- Hadoop 2.5.1
- Maven 3
- IntelliJ IDEA


### Procedure
설치 과정에 대한 요약입니다. 5번과 6번은 다음 포스트에서 이어서 다루도록 하겠습니다.

1. hosts 수정
1. Zookeeper 설치 & 설정 
2. HBase 설치 & 설정
3. HBase 실행
4. 간단한 쿼리를 이용한 설치 테스트
4. Client API 사용한 테스트



#### 1. /etc/hosts 수정

Hbase에서 peudo-distributed 모드를 정상적으로 사용하기 위해서는 /etc/hosts를 수정해주어야 합니다.<br>

```
127.0.0.1   localhost
```
부분을 삭제 혹은 주석처리 후, 다음 예시와 같이 실제 IP를 localhost 로 지정해주어야 합니다.

```
a.b.c.d     localhost
```

#### 2. Zookeeper 설치 & 설정
[다음 페이지](http://mirror.apache-kr.org/zookeeper/)에서 Zookeeper를 다운로드합니다.<br>
저는 Stable 버전인 3.4.6 버전을 다운로드 하였습니다.<br>
압축 해제한 경로를 `${ZK_HOME}`으로 정의합니다.
`${ZK_HOME}/conf`폴더로 이동하여 다음 명령어로 설정 파일을 생성합니다.

```bash
$ cp zoo_sample.cfg zoo.cfg
```
새로 생성한 파일 `zoo.cfg`에서 `dataDir`변수를 Zookeeper 데이터를 저장 할 경로를 지정해줍니다. 다른 설정값은 변경하지 않아도 무방합니다.<br>
다음 명령어를 이용하여 정상적으로 Zookeeper Server가 수행되는지 확인합니다.

```bash
$ ${ZK_HOME}/bin/zkServer.sh start
$ ${ZK_HOME}/bin/zkCli.sh -server IP_Address:2181
```
`quit`명렁어로 클라이언트를 종료한 후, 다음 명령어로 다시 Zookeeper 서버를 종료합니다.

```bash
$ ${ZK_HOME}/bin/zkServer.sh stop
```

#### 3. HBase 설치 & 설정
이 과정부터는 Hadoop 2.x 가 켜져 있는 상태임을 가정합니다. Hadoop이 켜져 있지 않은 상황이라면 Hadoop을 켜 주시기 바랍니다.<br>
[다음 페이지](http://mirror.apache-kr.org/hbase/stable/)에서 HBase를 다운로드합니다.<br>
다운로드 할 때는, Hadoop의 버전과 맞는 접미사를 다운로드해야 합니다. 이 문서에서는 Hadoop 2.5.1을 기반으로 하기 때문에 `hbase-0.98.7-hadoop2-bin.tar.gz`를 다운로드 하였습니다.<br>
다운로드한 압축파일을 해제하고, 그 경로를 `${HBASE_HOME}`으로 정의합니다.<br>
HBase에서 수정해야 할 설정파일은 `hbase-site.xml`과 `hbase-env.sh` 입니다.

##### hbase-site.xml
`${HBASE_HOME}/conf/hbase-site.xml`파일을 다음과 같이 설정합니다.

```xml
<configuration>
        <property>
                <name>hbase.rootdir</name>
                <value>hdfs://IP_ADDRESS:9000/hbase</value>
        </property>
        <property>
                <name>hbase.zookeeper.quorum</name>
                <value>IP_ADDRESS</value>
        </property>
        <property>
                <name>hbase.zookeeper.property.dataDir</name>
                <value>Zookeeper의 DataDir</value>
        </property>
        <property>
                <name>hbase.cluster.distributed</name>
                <value>true</value>
        </property>
        <property>
                <name>hbase.master.info.port</name>
                <value>60010</value>
        </property>
        <property>
                <name>hbase.master.info.bindAddress</name>
                <value>IP_ADDRESS</value>
        </property>
        <property>
                <name>dfs.support.append</name>
                <value>true</value>
        </property>
        <property>
                <name>dfs.datanode.max.xcievers</name>
                <value>4096</value>
        </property>
        <property>
                <name>hbase.zookeeper.property.clientPort</name>
                <value>2181</value>
        </property>
        <property>
                <name>hbase.regionserver.info.bindAddress</name>
                <value>IP_ADDRESS</value>
        </property>
</configuration>
```

##### hbase-env.sh
`${HBASE_HOME}/conf/hbase-env.sh`파일의 몇몇 변수를 다음과 같이 설정합니다.(주석 해제 후 값을 변경해 주면 됩니다.) <br>

* JAVA_HOME
    
    시스템에 설치되어 있는 JDK의 경로를 지정해줍니다. JDK6 or JDK7을 추천합니다. [참고](http://hbase.apache.org/book/configuration.html#java)
* HBASE_MANAGES_ZK

    true로 설정합니다.
    
다음은 설정의 예시입니다.

```bash
export JAVA_HOME=/usr/lib/jvm/java-7-openjdk-amd64
export HBASE_MANAGES_ZK=true
```
위 두 항목이 주석 해제되어 있고, 정상적인 값으로 설정되어있어야 합니다.


#### 4. Hbase 실행
다음 명령어를 이용하여 HBase를 실행합니다.

```bash
$ ${HBASE_HOME}/bin/start-hbase.sh
```
다음 명령어를 이용하여 HDFS에 HBase 폴더가 정상적으로 생성되었는지 확인합니다.

```bash
$ ${HADOOP_HOME}/bin/hdfs dfs -ls /hbase
```
그리고 다음 명령어를 이용하여 Zookeeper에도 Hbase가 정상적으로 등록되었는지 확인합니다.<br>
(`${HADOOP_HOME}`은 Hadoop이 설치되어 있는 경로입니다.)

```
${ZK_HOME}/bin/zkCli.sh -server IP_ADDRESS:2181
...
[zk:IP_ADDRESS:2181 (CONNECTED)] ls /
```
출력되는 항목에 `hbase`가 있어야 합니다.
