---
layout:     post
title:      "쇼핑 상품 카테고리 분류에서의 Multimodal Machine Learning"
subtitle:   "Introduction"
category:   Data Analysis
date:       2021-06-20 00:00:00
author:     "Hyunje"
tags:       [Multimodal, Deep Learning, Machine learning, Classification, Research]
---

<br>
이 글에서는 왜 Multimodal Machine Learning에 대해 Survey를 시작하게 되었는지, 어떻게 진행하고 있는지에 대해 설명한다.

<br>

## 개요

버즈니에서 근무한 5년동안 진행해온 프로젝트들은 상품 카테고리 분류기를 위한 `데이터셋을 구축`하고, 분류기에 사용될 `이미지 분류기를 학습`하고, `상품 분류기를 학습`시키고, `서비스에 적용`하고, `Scale-out`을 진행하는 등 상품의 카테고리 분류에 관련된 업무들이었다.

그동안에 외부에 공개된 기고문([마이크로소프트웨어 기고문](http://it.chosun.com/site/data/html_dir/2018/02/05/2018020585047.html), [컴퓨터월드 기고문](http://www.comworld.co.kr/news/articleView.html?idxno=49901&fbclid=IwAR0AsMEuuh146MGndyLMyEGfVGn6GtxzBUdTCyCzdWBxaQ8DjVnU2v8c6fI)) 에서 여러 번 언급되었듯, 상품의 카테고리 분류에 활용되는 주된 정보는 상품의 텍스트(상품명) 정보와, 이미지 정보(대표이미지)이다.

지금은 이제 기고문에서 작성했던 모델보다 개선된 부분들이 존재한다. 텍스트기반의 카테고리 분류기는 다른 팀원이 [ALBERT](https://arxiv.org/abs/1909.11942) 기반의 모델을 구축하여 서비스에 적용된 상태이며, 이미지 모델 역시 또 다른 팀원이 많이 개선을 해 주어서 기존 카테고리 분류기에 적용된 이미지 모델(InceptionResnetV2) 보다 훨씬 정확도가 높은 모델을 보유하고 있는 상황이다.

그렇다면 중요한 부분은 어떻게 상품명에서 추출된 정보와, 이미지에서 추출된 정보를 잘 활용하여 시너지가 나는 최종 모델(Text + Image Hybrid)을 학습시키는가 인데, 이 것과 관련된 키워드가 Multimodal 이다.

<br>
<br>

## Multimodal Machine Learning

Multimodal 은 두 가지 이상의 도메인 정보를 모두 활용하여 어떠한 문제를 해결하는 것을 말한다. 지금까지의 여러 딥러닝을 필두로 한 머신러닝 분야의 연구들은 하나의 도메인 내에서의 연구들인 경우가 많았다. 하지만 실제 현업에서 문제를 접하다보면 많은 문제들이 하나의 도메인에만 국한되어 있는 것이 아니다. 텍스트와 이미지, 텍스트와 소리, 텍스트와 동영상, 소리와 동영상 등 두개 이상의 도메인 정보를 활용하여 해결해야 하는 경우가 비일비재하다. 이러한 문제를 해결하기 두 가지 이상의 도메인 정보를 모두 활용하는 형태로 머신 러닝 모델을 구성하는 것이 Multimodal machine learning 이다.

Multimodal machine learning 에서 유명한 논문 중 하나인 [Tadas Baltrusaitis, Chaitanya Ahuja, and Louis-Philippe Morency, Multimodal Machine Learning: A Survey and Taxonomy](https://arxiv.org/abs/1705.09406) 에서는 Multimodal machine learning의 목적을 다음과 같이 `여러 modality 사이의 관계를 연관짓고, 처리할 수 있는 모델을 생성하는 것이 Multimodal machine learning의 목표이다.` 라고 정의하고 있다.

> Multimodal machine learning aims to build models that can process and relate information from multiple modalities.


<br>
<br>

## 기존에 적용된 방법

기존(2019년도)에 개발한 모델은 텍스트 정보와 이미지 정보를 활용할 때, 다음 그림에서도 볼 수 있듯이 단순한 concatenation 으로만 수행을 했었다.

<p align="center">
  <img src="https://cdn.comworld.co.kr/news/photo/202007/49901_34776_2127.png" />
</p>

가장 간편하게 모델에 multimodality를 적용시키는 방식이기 때문이다. ~~사실 어떤 방법이 있는지 알아볼 여유가 없었다.~~

다음 카테고리 분류 모델을 개발할 때에는 단순히 concatenation 이 아닌 고차원적인 Multimodal 접근법을 적용시켜보고자 한다. 이 글 시리즈는 기존의 상품 카테고리 분류기를 어떤 과정으로 모델을 업그레이드 해 나가고 있는지, 그 과정에서 찾아본 것들은 어떤 것들인지에 대해 기록하는 글이 될 것이다.

<br>
<br>

## 앞으로의 컨텐츠 (예상)

#### 1. Multimodal Survey

#### 2. ALBERT

#### 3. ALBERT + Multimodal

#### 4. Comparison

#### 5. Conclusion

## Survey
기존에 학습된 텍스트 기반 모델에 이미지 피쳐를 적용하여 카테고리 분류기를 학습시킬 때, 기존처럼 단순히 concatenation 만으로 진행하는 것보다 좀 더 다양한 방법을 적용하여 테스트 해 보고 싶었다.
테스트를 진행하기에 앞서 막연하게 단순 키워드를 입력해서 검색하는 것 보다는, 우선 Multimodal에 대한 대략적인 개념과, 접근법들을 확인하고 싶었다. 따라서 Survey 논문인 [Multimodal Intelligence: Representation Learning, Information Fusion, and Applications](https://arxiv.org/abs/1911.03977) 논문과, [New Ideas and Trends in Deep Multimodal Content Understanding: A Review](https://arxiv.org/abs/2010.08189) 논문을 읽어보았다.

<br>
<br>

## Multimodal
