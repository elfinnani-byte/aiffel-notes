
![[Pasted image 20260731115421.png]]

#### **CLIP(Contrastive Language–Image Pre-training) 모델의 핵심원리(학습과 추론 방식)**

[!info |] 이미지와 텍스트를 같은 공간에 넣어서 "서로 어울리는 이미지와 문장"을 찾도록 학습한 뒤, 새로운 분류 문제에도 사용할 수 있게 만든 방법

**전체 흐름**
```
**① 이미지-텍스트 관계 학습 → ② 텍스트로 분류기 만들기 → ③ 학습하지 않은 이미지도 분류(Zero-shot)**
```


## (1) Contrastive Pre-training (대조 학습)

### 목표

이미지와 텍스트를 보고 **서로 맞는 조합은 가깝게**, **틀린 조합은 멀게** 만드는 학습입니다.
그림 왼쪽 부분입니다.

### 1) 이미지와 문장을 준비

예:

이미지:

🐶 강아지 사진

텍스트:

- "Pepper the aussie pup"
    
- "A photo of a dog"
    
- "A photo of a bird"
    
- "A photo of a car"
    

---

### 2) 각각 Encoder를 통과

CLIP은 두 개의 신경망을 사용합니다.

### Image Encoder

이미지 → 숫자 벡터

```
강아지 사진
      ↓
Image Encoder
      ↓
[0.23, 0.51, -0.12 ...]
```

이것을 이미지 임베딩(Image embedding)이라고 합니다.

---

### Text Encoder

문장 → 숫자 벡터

```
"A photo of a dog"
          ↓
Text Encoder
          ↓
[0.25, 0.48, -0.10 ...]
```

---

### 3) 이미지와 텍스트의 유사도 계산

이미지 벡터와 텍스트 벡터를 비교합니다.

그림의 가운데 표:

||dog 문장|bird 문장|car 문장|
|---|---|---|---|
|강아지 이미지|높음 ⭐|낮음|낮음|
|자동차 이미지|낮음|낮음|높음 ⭐|

즉,

```
강아지 이미지 ↔ "A photo of a dog"
```

는 높은 점수

```
강아지 이미지 ↔ "A photo of a car"
```

는 낮은 점수

가 나오도록 학습합니다.

---

### 핵심

Contrastive Learning은 이렇게 학습합니다.

```
맞는 조합
이미지 + 설명
       ↓
가까워지게


틀린 조합
이미지 + 다른 설명
       ↓
멀어지게
```

---

# (2) Create dataset classifier from label text

### 목표

기존 딥러닝 분류기는 사람이 클래스를 학습시켜야 합니다.

예:

강아지 분류 모델

```
dog → class 0
cat → class 1
car → class 2
```

하지만 CLIP은 텍스트만 있으면 분류기를 만들 수 있습니다.

---

## 방법

분류하고 싶은 클래스 이름을 문장으로 변경합니다.

예:

분류 대상:

```
dog
cat
bird
```

↓

문장 생성:

```
"A photo of a dog"

"A photo of a cat"

"A photo of a bird"
```

---

이 문장들을 Text Encoder에 넣습니다.

결과:

```
dog 문장 벡터
       ↓
[0.2,0.8,...]


cat 문장 벡터
       ↓
[0.5,0.3,...]


bird 문장 벡터
       ↓
[0.7,0.1,...]
```

---

이것이 **텍스트 기반 classifier**가 됩니다.

즉,

일반 CNN:

```
이미지
 ↓
학습된 분류층
 ↓
dog
```

CLIP:

```
이미지
 ↓
Image Encoder
 ↓
텍스트들과 비교
 ↓
가장 가까운 문장 선택
```

---

# (3) Use for zero-shot prediction

### Zero-shot 의미

**학습하지 않은 새로운 클래스도 예측하는 것**

입니다.

---

예를 들어 CLIP은 처음 학습할 때:

```
dog
cat
car
bird
```

만 봤다고 가정합니다.

그런데 새로운 이미지가 들어옵니다.

---

## 입력 이미지

곰 사진 🐻

---

사용자가 후보 문장을 만듭니다.

```
"A photo of a dog"

"A photo of a bear"

"A photo of a bird"
```

---

이미지를 Image Encoder에 넣음

```
곰 이미지
 ↓
Image Encoder
 ↓
이미지 벡터
```

---

텍스트도 Encoder 통과

```
"A photo of a bear"
 ↓
Text Encoder
 ↓
텍스트 벡터
```

---

둘을 비교합니다.

결과:

|문장|유사도|
|---|---|
|A photo of a dog|0.32|
|A photo of a bird|0.40|
|A photo of a bear|0.89 ⭐|

---

따라서:

```
입력 이미지 = 곰
예측 결과 = bear
```

가 됩니다.

---

# 전체 흐름 한 번에 정리

```
             [1] Contrastive Pre-training

이미지                     텍스트
  ↓                         ↓
Image Encoder          Text Encoder
  ↓                         ↓
이미지 벡터              텍스트 벡터
          ↘             ↙
          유사도 학습
          (맞으면 가까이)
          (틀리면 멀리)



             [2] Text Classifier 생성

Label:
dog
cat
bird

↓

"A photo of a dog"
"A photo of a cat"
"A photo of a bird"

↓

Text Encoder

↓

각 클래스 벡터 생성



             [3] Zero-shot Prediction

새 이미지
   ↓
Image Encoder
   ↓
이미지 벡터

        비교

dog 벡터
cat 벡터
bird 벡터

        ↓

가장 가까운 텍스트 선택

        ↓

예측 결과
```

---

한 문장으로 요약하면:

**CLIP은 이미지와 문장을 같은 공간에 학습해 놓고, 새로운 이미지가 들어오면 "이 이미지가 어떤 문장과 가장 비슷한가?"를 판단하는 방식으로 분류하는 모델입니다.**


[!question]+ **CLIP가 왜 Zero Shot 모델인가?**
#### 1. CLIP이 '제로샷(Zero-shot)'인 이유

- **추가 학습(Fine-tuning) 불필요:** 기존 비전 모델들은 새로운 분류 과제나 데이터셋을 적용하려면 해당 라벨의 정답 데이터로 모델을 다시 학습시켜야 했습니다. 그러나 CLIP는 분류하고 싶은 새로운 정답 라벨에 대해 모델을 새로 학습시키지 않아도 바로 분류를 수행해 줍니다.
	- **사전 학습(Pre-training) 완료:** 단, 학습을 '전혀' 안 한 것이 아니라, 이미 4억 개의 이미지-텍스트 쌍으로 **'이미지와 언어의 관계'를 미리 대규모로 학습**해 두었기 때문에 가능한 능력이에요.
    
- **텍스트와의 유사도 기반 예측:** CLIP은 4억 개의 이미지-텍스트 쌍을 미리 대조 학습(Contrastive Pre-training)하여 이미지와 언어를 같은 공간에서 이해합니다.
    
- **라벨의 텍스트화:** 분류하고 싶은 새로운 라벨 이름을 `"A photo of a [클래스 이름]"`과 같은 문장으로 만든 뒤, 입력 이미지와 가장 유사도가 높은 문장을 고르는 방식으로 동작하므로 단 하나의 정답 샘플 학습 없이도 예측(Zero-shot Prediction)이 가능합니다.

---

#### 2. 일반 모델 vs CLIP 차이점

- **일반 비전 모델:** "강아지, 고양이"를 분류하려면 해당 정답 데이터로 **분류기(Classifier)를 직접 학습**시켜야만 분류할 수 있습니다.
    
- **CLIP:** 텍스트 라벨을 문장으로 만들면 모델이 알아서 이미지와 가장 유사한 문장을 찾아주므로, 별도의 분류기 학습 없이(Zero-shot) 바로 분류(Classify)해 줍니다.

---

#### 3. 출처 및 공식 정의

- **출처 논문:** OpenAI (2021), _“Learning Transferable Visual Models From Natural Language Supervision”_ (Radford et al.)
    
- **논문 내용:** 해당 논문에서는 이 방식을 'Zero-Shot Transfer(제로샷 전이)'라고 공식 정의하며, 과제 특화 학습 없이 자연어 지시(Prompt)와의 유사도 비교만으로 즉각적인 분류가 가능함을 핵심 원리로 제시합니다. (ImageNet 등 다양한 데이터셋에서 추가 학습 없이도 기존 지도학습(Supervised) 모델에 필적하는 높은 성능을 달성함을 증명했습니다.)