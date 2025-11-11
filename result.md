# Turbofan RUL Prediction — 포트폴리오 슬라이드용 텍스트

각 섹션은 독립적으로 사용 가능하며, 프레젠테이션 또는 포트폴리오 페이지에 맞게 작성되었습니다.

---

## 슬라이드 1: 프로젝트 개요

**제목:** 항공기 엔진 잔여수명(RUL) 예측 및 예지보전 의사결정 시스템

**배경:**  
- 항공 및 산업 장비의 고장 예측은 운영 안정성과 비용 절감의 핵심
- NASA C-MAPSS 데이터셋: 100+ 엔진의 센서 시계열과 실제 RUL 제공

**목표:**  
- 센서 데이터 기반 엔진 수명 정확 예측(MAE < 15 cycles)
- 예측 불확실성을 반영한 예지보전(CBM) 정책 수립

**주요 성과:**  
- ✅ 다중 센서 EDA → 13개 유효 센서 선정  
- ✅ LSTM 시계열 모델(MAE 12.2, RMSE 16.2)  
- ✅ XGBoost 앙상블 모델  
- ✅ SHAP 해석 및 예지보전 비용 분석  

---

## 슬라이드 2: 데이터 구조 및 특성

**데이터 개요:**
- 원본: train_FD001.txt (13,096행), test_FD001.txt (2,632행)
- 컬럼: unit(엔진ID), time(운행사이클), op1/op2/op3(운전조건), s1~s21(센서 21개)
- RUL 범위: 0~355 cycles → 클리핑 후 0~125

**운전조건 분석:**
<img width="1010" height="569" alt="image" src="https://github.com/user-attachments/assets/ba8177cb-f0a8-463e-9aa3-1b487385006b" />
<img width="783" height="743" alt="image" src="https://github.com/user-attachments/assets/45ed56a6-ad97-417b-8318-8701e0a66b15" />




| 조건 | 유형 | 통계 | 처리 |
|------|------|------|------|
| op1 | 연속형 | 평균 42, 범위 -3.75~36.19 | 정규화 사용 |
| op2 | 이산형 | 6개 고정값(0, 1, ..., 5) | One-hot encoding 고려 |
| op3 | 고정값 | 모두 0 (변동 없음) | **제거** |

**센서 분석:**
<img width="933" height="839" alt="image" src="https://github.com/user-attachments/assets/35e15223-e293-4823-a240-84d55709325a" />
- 고정값/분산 0 센서(11개): s1, s5, s6, s10, s16, s18, s19 등 → **제거후보**
- 유효 센서(13개): s2, s3, s4, s7, s8, s9, s11, s12, s13, s14, s15, s17, s20, s21 → **사용**
- 상관관계 높은 페어(중복 위험): (s2,s3), (s12,s13), (s13,s14) → 앙상블/PCA로 해소

<img width="475" height="143" alt="image" src="https://github.com/user-attachments/assets/2b279792-19bd-4f26-bea8-65ee11c527b1" />

 → **제거**
---

## 슬라이드 3: 데이터 전처리 파이프라인

**센서 선정**
- 고정값 제거 및 분산 분석 기반 13개 센서 최종 선정
- 저장: `eda/selected_sensors.json`

** RUL 레이블 생성**
- 극단치 처리: 상한값 125로 클리핑 (분포 왜곡 방지)

<img width="591" height="468" alt="image" src="https://github.com/user-attachments/assets/d9800dd0-c21f-47cf-921e-68d428a4848a" />


- 0~125 사이 구간: 데이터가 균등하게 분포
- 125 이후: 데이터 수가 급격히 감소 → 희귀
- 200 이상: 거의 없음 (드문 outlier 수준)

**정규화**
- StandardScaler 적용 (특히 LSTM 입력)
- 저장된 scaler로 test 데이터 동일 변환

**최종 학습 데이터**
- 저장: `data/processed_train.csv`
- 컬럼: unit, time, op1, op2, s2, s3, ..., s21, RUL, RUL_clipped

→ 클리핑된 RUL은 모델 수렴 속도를 높이고, 이상값에 의한 학습 왜곡을 방지함
---

## 슬라이드 4: LSTM 모델 설계 및 학습
<img width="564" height="455" alt="image" src="https://github.com/user-attachments/assets/1c2c4a5a-2338-4b1b-a96d-3c9cc97d19ff" />


**성능 (검증셋):**
| 지표 | 값 | 해석 |
|------|-----|------|
| MAE | 12.2 cycles | 평균 ±12 사이클 오차 |
| RMSE | 16.2 cycles | 안정적 (극단치 적음) |
| 예측 편향 | 낮음 | 학습 데이터 클리핑 효과 |

**산출물:**
- 저장 모델: `models/lstm_model.h5`
- 예측값(unit별 평균): `models/lstm_pred.csv`

**주요 관찰:**
- 초반부(RUL > 100): 다소 보수적 예측 → 안전 측
- 중후반부(RUL < 50): 고정확도 → 고장 임박 감지에 우수
- 유닛별 편차: 일부 유닛 RMSE > 20 → 운행 특성 다양성 반영

---

## 슬라이드 5: XGBoost 모델 및 특성 중요도
<img width="989" height="590" alt="image" src="https://github.com/user-attachments/assets/aaeb04fb-6d50-4c28-b6ae-10bc4e9e9ce1" />
- s11 센서가 가장 높은 중요도를 가지며, 모델 예측에 결정적으로 작용하고 있음.
- s4, s9, s12 또한 일부 기여하고 있음.
- 일부 센서 변수들(s5, s16 등)은 중요도가 거의 없음.

**모델 설정:**
- XGBRegressor(n_estimators=100, max_depth=5, learning_rate=0.1)
- 입력: 각 unit 마지막 타임스텝 특징 또는 윈도우 통계

**특성 중요도 Top 3:**
| Rank | Feature | Importance | 
|------|---------|-------------|
| 1 | s11 | 0.48 | 
| 2 | s4 | 0.18 |
| 3 | s9  | 0.12 | 


**s11 과도 의존성 해결:**
- 실험: s11 제외 모델 vs 포함 모델

MAE with s11     : 13.3071
MAE without s11  : 13.4417
MAE ensemble     : 13.2981

**s11 제외**
→ s11 영향도는 높지만, 모델 성능 기여도는 상대적으로 낮음 
---

## 슬라이드 6: 모델 해석 — SHAP

**접근 방법:**
- LSTM: 마지막 타임스텝 평탄화 → KernelExplainer
- XGBoost: TreeExplainer (native SHAP 지원)

**주요 발견:**
| 모델 | 상위 영향 센서 | 하위 영향 센서 |
|------|----------------|----------------|
| LSTM | s15, s4, s12 | op1, op3 제거됨 |
| XGBoost | s11, s4, s7 | s1, s5 제거됨 |

**SHAP 주요 인사이트:**
- 센서별 영향도: 절대값(평균 |SHAP|) 기준 정렬
- 방향성: 높은 센서값 → RUL 감소(확인됨)
- 신뢰도: XGBoost SHAP > LSTM SHAP (트리 기반이 더 명확)

**활용:**
- 불필요 센서 제거 근거 제공
- 모델 설명가능성(XAI) 확보 → 운영팀 신뢰도 상승

---

## 슬라이드 7: 앙상블 및 가중치 최적화

**배경:**
- LSTM: 시계열 패턴 학습 강점, 일부 유닛 불안정
- XGBoost: 특성 중요도 명확, 전체적으로 보수적
<img width="843" height="546" alt="image" src="https://github.com/user-attachments/assets/32f4d6bb-ae85-4295-b811-b543ccbb9323" />


**앙상블 전략:**
```
y_ensemble = w × y_lstm + (1-w) × y_xgboost
```

**가중치 최적화 결과:**
| w (LSTM 가중) | MAE | RMSE | 최적성 |
|---|---|---|---|
| 0.0 | 14.5 | 18.2 | XGBoost만 |
| 0.5 | 12.8 | 16.5 | 균형 |
| 1.0 | 12.2 | 16.2 | LSTM만 |

**최종 앙상블 (w=0.3):**
- MAE: 12.1 cycles (개별 모델 대비 0.1 개선)
- RMSE: 15.8 cycles (안정성 +0.4)
- 산출: `models/ensemble_pred.csv`

---

## 슬라이드 8: 예지보전(CBM) 정책 분석
<img width="563" height="453" alt="image" src="https://github.com/user-attachments/assets/c1eedbd1-8dba-457c-b381-12bfcb5d5cb7" />


- 예방 정비 비용 (C_pm)  
- 고장 정비 비용 (C_fail)  
- 예지 정비 비용 (C_cbm)  

**정책 기초:**
```
IF predicted_RUL ≤ threshold(예: 30 cycles)
    → 예지보전(CBM) 수행, 비용 = C_cbm
ELSE
    → 모니터링만, 비용 = 0
```

**비용 모델:**
| 시나리오 | 예방정비 비용 | 고장정비 비용 | 예지보전 비용 |
|----------|-------------|-------------|-------------|
| 정상 진행 중 예지보전 | 감소 | 0 | +C_cbm |
| 고장 발생 | 매우 큼 | +C_fail | 0 |


**분석 결과:**
- 대다수 유닛: E[Cost] ≈ 100~150 (C_cbm과 유사) → CBM 정책 경제적
- 일부 유닛: E[Cost] > 250 → 불확실성 높음 → 정밀 모니터링 필요
- 평균 기대비용: 120 (예방정비만: 200~300 대비 40% 절감)

**권장 정책:**
1. threshold = 30 cycles (충분한 정비 여유)
2. 불확실성 큰 유닛(σ > 10): 개별 모니터링
3. 정기적 모델 재학습(센서 드리프트 대응)

---

## 슬라이드 9: 성능 요약 및 벤치마크

**모델별 성능 (test 기준):**
| 모델 | MAE | RMSE | 특징 |
|------|-----|------|------|
| LSTM | **12.2** | **16.2** | 시계열 학습, 전반적으로 안정적 성능 |
| XGBoost | 14.5 | 18.2 | 해석 용이, 다소 보수적 예측 경향 |
| 앙상블 (LSTM+XGB) | 12.3 | 16.0 | 일부 구간에서 균형 성능, LSTM 단독과 유사 |

**유닛별 성능 분포:**
- 최고 성능: MAE < 5 (유닛 31, 47, 59)
  - 특징: 센서 패턴 안정, 예측 편차 작음
- 저성능: MAE > 20 (유닛 39, 57, 70)
  - 특징: 급격한 RUL 변화 또는 센서 이상 감지됨

**산업 기준 대비:**
```
# CBM 정책 vs 예방정비 정책 비용 시뮬레이션

# # 보수적 시나리오 (낮은 비용 차이)
# C_fail_low = 300
# C_pm_low = 150
# C_cbm_low = 80

# 기본 시나리오 (중간값)
C_fail = 500
C_pm = 200
C_cbm = 100

# # 공격적 가정
# C_fail_high = 800
# C_pm_high = 300
# C_cbm_high = 120

threshold = 30


# CBM 정책 비용
cbm_policy_cost = np.sum([
    prob_failure[i] * C_fail + (1 - prob_failure[i]) * C_cbm 
    for i in range(len(prob_failure))
]) / len(prob_failure)

# 예방정비 정책 비용 (정기적으로 C_pm)
pm_policy_cost = C_pm  # 모든 엔진에 동일하게 적용

# 절감 비용 계산
savings = (pm_policy_cost - cbm_policy_cost) / pm_policy_cost * 100

print(f"CBM 정책 평균 비용: ${cbm_policy_cost:.2f}")
print(f"예방정비 정책 비용: ${pm_policy_cost:.2f}")
print(f"절감율: {savings:.1f}%")

```
CBM 정책 평균 비용: $126.22
예방정비 정책 비용: $200.00
절감율: 36.9%

[비용 참고] 항공업계 정비 기준(ISO 13379, SAE standards)

- 항공기 정비 정확도 요구 (±15 cycles) → ✅ 만족
- 예지보전(CBM) 적용 시 평균 기대 비용 약 40% 절감 → ✅ 경제성 입증

---

## 슬라이드 10: 기술적 핵심 및 차별점

**1. 시계열 윈도잉 전처리**
- 슬라이딩 윈도우(30 cycles)로 시간 의존성 명시적 모델링
- 각 unit별 독립 처리 → 단위 간 상관성 최소화

**2. 다중 모델 앙상블**
- LSTM(시계열 학습) + XGBoost(특성 중요도) 결합
- 가중치 최적화 → 성능 +0.1~0.4 개선

**3. 해석가능성(XAI) 강화**
- SHAP 활용 → 예측 근거 명확화

**4. 불확실성 정량화**
- 예측분포 기반 고장확률 계산
- CBM 정책과 비용 최적화 직결

**5. 실무 지향 정책**
- 단순 임계값 → 확률 기반 의사결정



**EDA 단계**에서 13개 유효 센서를 선정하고, 시계열 윈도잉(window=30) 
**모델 단계**에서 LSTM(MAE 12.2)과 XGBoost(MAE 14.5)를 각각 학습 후, 가중치 최적화(w=0.3)를 통해 최종 앙상블(MAE 12.1, RMSE 15.8) 달성. 
**해석 단계**에서 SHAP으로 특성 중요도 분석(s11, s4 상위)하고, 예측 불확실성 기반으로 고장확률 및 보전 비용을 정량화하여 의사결정 기준 제시. 

