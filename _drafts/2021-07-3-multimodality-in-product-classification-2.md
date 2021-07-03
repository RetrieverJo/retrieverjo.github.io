---
layout:     post
title:      "쇼핑 상품 카테고리 분류에서의 Multimodal Machine Learning"
subtitle:   "Multimodal Survey"
category:   Data Analysis
date:       2021-06-20 00:00:00
author:     "Hyunje"
tags:       [Multimodal, 멀티모달,  Deep Learning, Machine learning, Classification, Research, Survey]
---

<br>
이 글에서는 Multimodal 에 대해 Survey를 진행했던 과정에 대한 글이다.

<br>

## Survey Paper

Multimodal Machine Learning 분야에 대해 처음 접하다 보니, 자연스럽게 Survey 논문에 눈이 갔고, 찾아보니 유명한 Survey 논문이 두 개가 있었다.

하나는 [Multimodal Intelligence: Representation Learning, Information Fusion, and Applications](https://arxiv.org/abs/1911.03977) 였으며, 다른 하나는 [New Ideas and Trends in Deep Multimodal Content Understanding: A Review](https://arxiv.org/abs/2010.08189) 였었다.

두 논문 모두 Multimodal Machine Learning 에 대한 설명과 접근법들이 잘 설명되어 있었지만, 장단점이 존재했다. [Multimodal Intelligence: Representation Learning, Information Fusion, and Applications](https://arxiv.org/abs/1911.03977)는 간략하게 설명을 해 주고 있어서 대략적인 Multimodal Machine Learning 에 대한 개념을 잡기 좋았지만, 최신 트렌드나 기술들까지 커버하고 있지는 않은 것 같았다. 그리고 세부적인 접근법까지 파악하기 위해서는 간략하게 언급된 키워드만으로 다시 찾아보거나, 레퍼런스로 되어 있는 논문을 다시 읽어 봐야만 하였다.

반면 [New Ideas and Trends in Deep Multimodal Content Understanding: A Review](https://arxiv.org/abs/2010.08189) 논문은 매우 많은 논문들을 비교적 세세하게 리뷰하고 있어, 논문별로 어떤 형태로 접근법을 제안하고 있는지 좀 더 쉽게 알 수 있었다. 하지만 너무 논문의 길이가 길어 내가 적용하고자 하는 부분과 연관성이 깊은 부분만 취사선택 해야 할 것이라 판단되었다.

따라서, 우선적으로 [Multimodal Intelligence: Representation Learning, Information Fusion, and Applications](https://arxiv.org/abs/1911.03977) 를 읽고 대략적인 Multimodal Machine Learning에 대해 파악을 하고, 이후 내가 해결하고자 하는 문제에 맞는 방법을 찾고자 할 때 [New Ideas and Trends in Deep Multimodal Content Understanding: A Review](https://arxiv.org/abs/2010.08189) 논문을 참고하는 것으로 결정하였다.

<br>
<br>

이후 내용은 논문을 일부 번역하였기 때문에, 문장이 과거형이 아닌 현재형이며, 일부 오역이 있을 수 있습니다.

## Multimodal Intelligence: Representation Learning, Information Fusion, and Applications

이 논문에서는 NLP(Natural Language Processing)와 CV(Computer Vision) 분야의 합성에 관한 Modality를 집중적으로 설명하고 있으며, Multimodal 을 *Representations*, *Fusion*, *Applications* 세 가지 주제를 기준으로 설명한다.
<br>
<br>


## Represestations
특정 분야에서의 Representation learning 으로써의 딥러닝은 많은 인공신경망을 이용해 raw data를 특정 태스크에 맞는 representation(표현) 혹은 feature(피쳐)를 발견하는 것에 집중하고 있다. 좋은 Representation을 생성하는 것은 보통 문제를 매우 간결하게 만들어 주기 때문에 중요한 역할을 하고 있다. 지난 수십년동안 하나의 modality를 해결하기 위해 많은 연구들이 진행되어 왔고 효과적이고 robust한 representation들이 연구되었지만, multimodal representation이 점점 주목받고 있음에도 여전히 복잡한 cross-modal interaction과 학습 데이터와 각 modality에서 테스트 데이터 사이의 불일치로 인해 여전히 어려운 문제이다.

이 섹션에서 multimodality 에서 joint representation space를 학습하는데 사용되는  supervised 방식과 unsupervised 방식을 주로 설명하고, 연관되어 있는 zero-shot 방식과, Language Model 에서 주로 사용되었던 방식에서 착안한 큰 unimodal(하나의 Modal)들의 데이터셋을 활용하여 multimodal representation 을 학습하는 방법 역


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
