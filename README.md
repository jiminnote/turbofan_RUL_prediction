# NASA Turbofan Engine RUL Prediction

잔존 수명 예측 기반 예지보전 시스템 구축 프로젝트  
캐글 공개 데이터셋(C-MAPSS)을 활용한 시계열 기반 머신러닝 모델링

## 프로젝트 개요

항공기 터보팬 제트 엔진의 센서 데이터를 분석하여  
각 엔진의 남은 수명(RUL, Remaining Useful Life)을 예측하고,  
이를 기반으로 예지보전(PdM, Predictive Maintenance) 체계를 설계합니다.

### 목적
- 고장 시점을 사전에 예측하여 정비 시점 최적화
- 불필요한 조기 교체 방지 및 비용 절감
- 안전성 확보 및 데이터 기반 의사결정 지원

## 데이터셋 정보

- 출처: NASA C-MAPSS Dataset (Kaggle)
- 형식: 시계열 센서 데이터 (온도, 압력, 속도 등)
- 엔진 수: 100+ 개별 엔진, 다양한 운항 조건과 고장 모드
- 구성
  - train_FD00X.txt: 고장까지 전체 주기 데이터
  - test_FD00X.txt: 고장 직전까지만 수집된 데이터
  - RUL_FD00X.txt: 각 test 엔진의 실제 잔존 수명

## 모델링 전략

1. 시계열 기반 딥러닝
   - LSTM: 점진적인 열화에 강한 모델
   - TCN: 급격한 변화 탐지에 강점

2. 트리 기반 회귀
   - XGBoost: 통계 피처 기반의 회귀 예측

3. 앙상블
   - LSTM/TCN과 XGBoost의 예측값을 결합하여 성능 보완

## 전처리 및 특징

| 모델 유형 | 주요 전처리 방식 |
|-----------|-----------------|
| 딥러닝 (LSTM/TCN) | 시계열 분할 (슬라이딩 윈도우), 정규화 |
| 트리 기반 (XGBoost) | 통계적 피처 생성 (최근값, 평균, 변화율, 임계 이벤트 횟수 등) |

- 센서별 변화 계수(CV)를 활용해 모델 선택 (LSTM vs TCN)
- 임계값 초과 이벤트 피처 활용하여 이상 징후 감지

## 성능 평가

- 지표: RMSE, MAE, Scoring Function (PHM 대회 방식)
- 기타:
  - SHAP: XGBoost 피처 해석
  - Attention Map: LSTM 시각화
  - 시계열 추정 그래프 및 오차 분석

## 폴더 구조

turbofan_RUL_prediction/
├── data/               # Raw 및 전처리 데이터
├── eda/                # 탐색적 분석
├── preprocessing/      # 전처리 스크립트
├── models/             # LSTM, TCN, XGBoost 등 모델
├── ensemble/           # 앙상블 로직
├── notebooks/          # 실험용 주피터 노트북
├── reports/            # 결과 분석, 시각화
├── presentation/       # 발표 자료
└── README.md

## 사용 기술

- 언어: Python
- 프레임워크: PyTorch, Scikit-learn, XGBoost
- 시각화: Matplotlib, Seaborn, Plotly
- 튜닝: Optuna, GridSearchCV
- 해석도구: SHAP, Attention Layer

## 확장 가능성

- 산업현장 실시간 PdM 시스템 적용 (IoT + 모델 배포)
- 전기차 배터리, 공장 설비 등 타 도메인 확장 가능
- Meta Learning, RL 기반 유지보수 최적화로 확장

## 팀 정보

- 제로베이스 AI 부트캠프 4인 팀 프로젝트
- 기간: 2025년 11월 (총 4주)

## 참고 문헌

- Saxena et al., "Damage Propagation Modeling for Aircraft Engine", PHM08
- C-MAPSS 공식 문서 및 유저 가이드
