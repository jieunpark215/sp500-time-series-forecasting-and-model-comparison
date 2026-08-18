# S&P500 로그 수익률 기반 시계열 예측 모형 비교

S&P500 일별 로그 수익률에 통계 모형(ARIMA, GARCH)부터 딥러닝(LSTM), Transformer 계열(PatchTST), Foundation Model(Chronos)까지 네 세대의 시계열 예측 방법론을 동일한 데이터·평가 기준으로 적용하고 비교한 개인 프로젝트입니다.

## Overview

- **데이터**: S&P500(^GSPC) 일별 종가, 2015-01-01 ~ 2024-12-31 (약 2,500 거래일, Yahoo Finance)
- **목표**: 금융 시계열의 비정상성·변동성 군집·fat tail 특성이 모형별 가정과 어떻게 충돌하는지 확인하고, 통계 모형과 딥러닝/사전학습 모형 간 예측 성능·해석 가능성 트레이드오프를 비교
- **구성**: 전처리·정상성 검정은 공통, 이후 R(통계 모형)과 Python(신경망 모형)으로 분리 진행

## 분석 파이프라인

```
데이터 수집(quantmod/yfinance) → 로그 수익률 변환 → ADF 정상성 검정 → ACF/PACF
        │
        ├─ R:  ARIMA(2,0,2) → GARCH(1,1) [ARCH 효과 검정 → 변동성 모형]
        │
        └─ Python: LSTM → PatchTST → Chronos(Zero-shot)
                │
                └─ 공통 평가: RMSE, MAE (train/test 80:20 분리)
```

## 결과 요약

| 모형 | 구분 | 특징 | RMSE | MAE |
|---|---|---|---|---|
| ARIMA(2,0,2) | 통계 | 잔차 자기상관 거의 없음(Ljung-Box p=0.3218), ARCH 효과 존재(p<0.001) → GARCH 필요성 확인 | - | - |
| GARCH(1,1) | 통계 | 변동성 지속성 α+β≈0.997, 변동성 군집(COVID-19, 금리인상 구간에서 급등) 포착 | - | - |
| LSTM | 딥러닝 | 60일 시퀀스 입력, 2-layer(64→32) + Dropout | 0.008232 | 0.006283 |
| PatchTST | Transformer | 패치 단위(16일) 토큰화, input_size=336 | 0.008565 | 0.006608 |
| Chronos | Foundation Model | amazon/chronos-t5-small, 추가 학습 없는 Zero-shot 예측 | **0.008165** | **0.006220** |

Zero-shot인 Chronos가 별도 학습 없이도 LSTM·PatchTST와 유사하거나 더 낮은 오차를 보였다는 점이 이 비교의 핵심 발견입니다. 다만 세 모형 간 오차 차이 자체가 크지 않아, S&P500 로그 수익률 예측에서는 모형 복잡도를 높이는 것 대비 얻는 이득이 제한적이라는 해석도 가능합니다.

*(ARIMA/GARCH의 RMSE/MAE는 R 세션 결과값이 별도로 저장되지 않아 재실행 후 업데이트 예정입니다.)*

## 주요 발견

- **정상성**: 원시 가격은 비정상(추세 존재), 로그 수익률은 ADF 검정상 정상 시계열로 확인 → ARIMA/GARCH 등 선형 모형 적용의 전제 조건 충족
- **변동성 군집**: GARCH(1,1) 추정 결과 α+β≈0.997로 변동성 충격이 매우 오래 지속됨을 확인. COVID-19 충격, 2022년 Fed 금리 인상기에 변동성 급등 구간이 육안으로도 뚜렷하게 관찰됨
- **모형 간 비교**: 통계 모형(ARIMA/GARCH)은 해석 가능성이 높지만 비선형 패턴을 포착하지 못함. 신경망 모형은 비선형 패턴을 학습할 수 있지만 이번 실험에서는 통계 모형 대비 압도적인 성능 우위를 보이지 않음 — 효율적 시장 가설과 궤를 같이하는 결과
- **Foundation Model의 실용성**: Chronos는 도메인 특화 학습 없이도 baseline급 성능을 즉시 낼 수 있어, 빠른 벤치마킹 도구로서의 가능성을 보여줌

## Tech Stack

- **R**: quantmod, urca, FinTS, forecast, rugarch
- **Python**: yfinance, tensorflow/keras, neuralforecast(PatchTST), chronos-forecasting

## Repository 구조

```
├── r/
│   └── arima_garch.Rmd          # 데이터 수집, 정상성 검정, ARIMA, GARCH
├── python/
│   └── lstm_patchtst_chronos.ipynb  # LSTM, PatchTST, Chronos 구현 및 비교
└── README.md
```

## 한계 및 향후 과제

- 잔차에 일부 구조가 남아있어(Ljung-Box 일부 시차에서 유의) ARMA 모형만으로는 완전히 설명되지 않는 패턴 존재 → GARCH류 변동성 모형 결합 필요성 확인
- 현재는 로그 수익률 자체(단변량)만 사용 — 거시경제 지표, 뉴스 센티먼트 등 외부 변수를 결합한 멀티모달 예측은 후속 과제
- Chronos는 t5-small 버전 사용 — 더 큰 버전 또는 금융 도메인 fine-tuning 시 성능 변화 확인 필요
