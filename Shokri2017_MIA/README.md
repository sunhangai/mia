# 🧠 Membership Inference Attack (MIA) 정리

## 🔎 배경 시나리오
보건복지부가 AI 모델을 이용해 "수면, 식사, 운동 등 설문 데이터를 기반으로 암 발생 위험 예측"을 시도.  
이러한 모델에 대해, **"이 사람이 훈련에 사용되었는가?"를 추론하는 공격(MIA)**이 가능한지를 살펴봄.

---

## 🧱 전체 구조: 3단계 모델 구성

| 구성 요소 | 설명 |
|------------|------|
| 🎯 Target Model | 보건복지부가 만든 암 예측 모델 (black-box로 가정) |
| 🧪 Shadow Model | target과 유사하게 행동하도록 공격자가 별도 학습 |
| 🔥 Attack Model | shadow model 출력값 기반으로 in/out 여부 예측 |

---

## 🛠️ 구현 흐름

### 1. Shadow Model 생성
- 실제 보건복지부 모델은 접근 불가 → Google API 또는 유사한 공개 데이터를 활용
- Google Colab + Keras로 구현

### 2. Attack Data 구축
- Shadow 모델의 학습 데이터 → in
- 보지 않은 테스트 데이터 → out
- 둘에 대해 softmax 결과 + 정답 레이블 추출

### 3. Attack Model 학습
- 위 데이터 기반으로 이진 분류 모델 학습
- 입력: softmax 확률 + label → 출력: member / non-member

### 4. Target Model에 공격
- 대상(K)의 설문 데이터를 넣고 softmax 결과를 attack model에 투입
- K가 훈련 데이터에 포함되었는지 여부 판단 가능

---

## ❓ 핵심 질문
- 모델이 훈련에 사용된 데이터를 **출력값만으로 식별할 수 있는가?**
- **특정 데이터가 training set의 member였는가**를 추론하는 것이 목적

---

## 🎯 공격 시나리오
- 완전 블랙박스 접근 (입력 ↔ 출력만 알 수 있음)
- 과적합 모델은 훈련 데이터에 대해 더 확신에 찬 출력 → 그 차이를 이용해 공격 가능

---

## ⚙️ Shadow Model 방식
1. 타겟과 유사한 구조로 여러 개의 shadow model 생성
2. 각각에 in/out 데이터를 넣고 훈련
3. 출력 패턴의 차이를 기반으로 attack model 훈련
4. attack model이 target의 출력만 보고 membership 여부 예측

---

## 🧪 실험에 사용된 데이터셋

| 데이터셋 | 특징 |
|----------|------|
| Purchase | 쇼핑 이력 기반 600 binary features, 최대 100개 클래스 |
| Texas | 병원 입원 기록 기반 6,170 binary features |
| Location | Foursquare 위치 체크인, 30개 클래스 |
| CIFAR-10/100 | 이미지 분류 |
| MNIST, UCI Adult | 필기체/인구 통계 |

---

## 💡 공격이 가능한 이유

### 🎯 Member 데이터
- Softmax 분포가 확실: 예) [0.99, 0.01, 0.00, ...]
- Entropy 낮고, margin 큼 → 확신에 찬 예측

### ⛔ Non-member 데이터
- 분산된 분포: 예) [0.3, 0.2, 0.2, 0.3, ...]
- Entropy 높고, margin 작음 → 조심스러운 예측

→ 이 차이를 이용해 attack model이 학습

---

## 🛡️ 방어 기법 요약

| 기법 | 설명 |
|------|------|
| Top-k 제한 | 상위 k개 클래스만 출력 |
| Softmax precision 감소 | 소수점 자리수 줄이기 |
| Temperature scaling | softmax 평탄화 |
| Regularization | 과적합 억제 (Dropout 등) |
| Differential Privacy | 노이즈 삽입으로 정보 유출 방지 |

👉 그중 Regularization, DP가 가장 효과적

---

<img width="567" height="389" alt="image" src="https://github.com/user-attachments/assets/8a9e60b2-e6e8-4263-ad3d-127aeff5b8ec" />
<img width="567" height="298" alt="image" src="https://github.com/user-attachments/assets/68d8f758-069e-45f7-a80b-c099af3c5d9e" />
<img width="567" height="408" alt="image" src="https://github.com/user-attachments/assets/8d1dc165-9c27-4a1f-9fc3-bedc311eb8bc" />
<img width="567" height="328" alt="image" src="https://github.com/user-attachments/assets/b3a91665-6582-4051-909d-23332353ce9e" />
<img width="567" height="313" alt="image" src="https://github.com/user-attachments/assets/4098ca66-ff82-4716-b137-33ce140f458b" />


## 🧪 실습 요약 (MNIST, Colab 환경)

### ✅ 기본 실험 (overfitting X)
- Target Model: epochs=20, validation_split=0.1
- Attack Accuracy: **0.5600**
- Precision: 0.5319
- Recall: 1.0000  
→ 모델이 일반화되어 member/non-member 구별 어려움

---



### ✅ 과적합 유도 실험
- Target Model: epochs=100, Dropout 없음, val_split=0
- Shadow Model: 5개, 검증 없이 학습
- Attack Accuracy: **0.5400**
- Precision: 0.5233
- Recall: 0.9000  
→ precision은 낮지만 recall은 높음 (공격 가능성 존재)

---

## 🔍 실험 실패 원인

1. Shadow Model 다양성 부족 (논문은 100개 이상 실험)
2. MNIST는 간단한 데이터셋이라 in/out 분포가 유사함
3. threshold 조정, ROC/AUC 분석 미실시 → 논문 수준 분석 부족

---

## 🧠 결론

- MIA는 현실적인 위협
- Black-box 환경에서도 훈련셋 유출 가능
- 과적합 방지, 정규화, Differential Privacy가 핵심 대응책

---

✍️ *by Sunhangai — Weekly Paper Log*
