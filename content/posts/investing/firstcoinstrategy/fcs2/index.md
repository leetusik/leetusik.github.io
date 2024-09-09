+++
title = '코인2024[2] - 선행 연구 리서치'
date = 2024-09-09T13:29:53+09:00
categories = ["investing"]
tags = ["coin2024", "coin", "investing", "research"]
toc = true
draft = false

+++

코인 투자 선행 연구들을 찾아본다.

<!--more-->

## 선행 연구

1. 딥러닝과 단기매매전략을 결합한 암호화폐 투자 방법론 실증 연구 - 이유민 등
2. 딥러닝을 이용한 비트코인 투자전략의 성과 분석 - 김선웅
3. 기타 인터넷 출처 연구들

---

### 1. 딥러닝과 단기매매전략을 결합한 암호화폐 투자 방법론 실증 연구

#### 1.1 메인 아이디어

변동성돌파 + LSTM 모델 가격 예측 전략

#### 1.2 연구 설계 부분

하락 추세였던 2022년 1월 ~ 5월 자료를 사용했음.
과거 7일의 데이터로 다음 날 종가를 예측.

#### 1.3 데이터

- 날짜
- 시가
- 고가
- 저가
- 종가
- 거래량

#### 1.4 LSTM 파라미터

1. learning rate = 0.002
2. 1st layer unit = 64
3. 2nd layer unit = 4
4. epoch = 200
5. batch size = 64
6. loss function = MSE

7. window size = 7
8. 활성화 함수(activation function) = 하이퍼볼릭 탄젠트(hyperbolic tangent)
9. 옵티마이저(optimizer) = Adam (오차 감소 속도가 빠름(?))

#### 1.5 best model 매매 규칙

당일 고점과 당일 예측 종가가 당일 타겟가보다 높을 때, 당일 타겟가에 구매해서 당일 종가에 판매.

```python
# 매수 규칙
if high(t) > target(t) & pred(t) > close(t):
 buy(price=target(t))


# 매도 규칙
then = "9am" # It's for example. it would be datetime obj in real machine.

if now >= then and if get_balance("KRW-BTC") != 0:
  sell(price = market)
```

#### 1.6 연구 결과 성과 분석을 위한 평가 지표

{{< image_caption src="performance_indicators.png" alt="전략 성과 평가 지표" caption="전략 성과 평가 지표" >}}

#### 1.7 궁금증

- 종가 기준은 몇시?
  아마 pyupbit에서 일별 데이터 그대로 가져다 쓴 것 같은데 이런 경우에는 오전 9시로 설정됨

  - 시간대는 모두 고려해야함

- 슬리피지 고려 했나?

  - 따로 언급이 없는 것으로보아 고려하지 않음. 매수 조건이 달성되었을 때 바로바로 매매가 실행되는 것이 아니므로 매수, 매도 시에 슬리피지를 고려해야함.

- 메인 아이디어가 시장참여비율이 낮아서 수익률이 높을 수도. 하락장에 시자아 참여 비율이 낮다는 것은 긍정적이지만, 상승장에서 따라갈 수 있는지도 확인해야.

---

### 2. 딥러닝을 이용한 비트코인 투자전략의 성과 분석 - 김선웅

#### 2.1 메인 아이디어

기술적 지표(이동평균선) + LSTM 전략.

#### 2.2 데이터 소개

2017년 5월 23일 ~ 2021년 1월 23일

#### 2.3 연구 방법

1. 비트코인 일별 시가, 고가, 저가, 종가, 일별 수익률 자료 활용 다음 날 비트코인 종가 예측
2. LSTM 비교 모형으로 단순 RNN
3. 최적화 알고리즘 adam optimizer 이용 -> 오차 감소 속도
4. 활성화 함수 -> hyperbolic tangent
5. 지도학습(epoch) 100번. 추가적으로 확대해도 크게 성과 차이 x

{{< image_caption src="hyperparameters.png" alt="LSTM 파라미터 설정" caption="LSTM 파라미터 설정" >}}

---

#### 2.4 비교 투자 전략

1. 이동평균선 교차 전략 + lstm

   - 단기 이평선(5)이 장기 이평선(20)을 상향 돌파할 경우 돌파 시점 종가로 매수. 반대의 경우 돌파 시점 종가로 매수 포지션 청산

   {{< image_caption src="strategy.png" alt="연구에 활용된 전략" caption="연구에 활용된 전략" >}}

2. 그냥 이동평균선 교차 전략

   - 1 but no predict C just C

3. buy & hold

#### 2.5 전략 별 성과

{{< image_caption src="performance2.png" alt="전략 별 성과" caption="전략 별 성과" >}}

#### 2.6 궁금증

그냥 ma 전략이랑 차이가 이 정도면 그냥 예측 효과가 있다고 보기 어려울 듯.

---

### 3. 기타 인터넷 출처 연구들

#### 3.1 RSI

기본적인 rsi 전략은 다음과 같다.

```python
# Calculate the RS (Relative Strength) and RSI
df['rs'] = df['avg_gain'] / df['avg_loss']
df['rsi'] = 100 - (100 / (1 + df['rs']))
```

매수, 매도 조건은 다음과 같다.

```python
df['signal'] = 0  # Default to no position
for i in range(1, len(df)):
    # 매수 조건
    if (df['rsi'].iloc[i] >= 30) and (df['rsi'].iloc[i-1] < 30):
        df['signal'].iloc[i] = 1
    # 매도 조건
    elif (df['rsi'].iloc[i] <= 70) and (df['rsi'].iloc[i-1] > 70):
        df['signal'].iloc[i] = -1
```

#### 3.2 이동평균선

이동평균선 전략은 다음과 같다.

```python
# ...

df[f"{interval}_ma"] = df['close'].rolling(window=interval).mean()

df['signal'] = 0  # Default to no position
for i in range(interval, len(df)):
    # Buy condition
    if df.loc[i, 'close'] >= df.loc[i, f'{interval}_ma'] and df.loc[i-1, 'close'] < df.loc[i-1, f'{interval}_ma']:
        df.loc[i, 'signal'] = 1
    # Sell condition
    elif df.loc[i, 'close'] < df.loc[i, f'{interval}_ma'] and df.loc[i-1, 'close'] >= df.loc[i-1, f'{interval}_ma']:
        df.loc[i, 'signal'] = -1

# ...

# 다음의 이동평균선을 모두 고려한다.
interval_list = [5, 10, 20, 50, 90, 120, 200]
```

#### 3.3 가격 모멘텀

가격 모멘텀 전략은 매일 가격의 모멘텀을 판단하고 양의 모멘텀일 경우에 매수, 보유 중 음의 모멘텀으로 전환될 시 매도하는 전략이다. 매수와 매도는 하루 중 기준 시간에 실행된다.

```python
#...

df[f"{momentum}_momentum"] = 0

df['signal'] = 0  # Default to no position
for i in range(momentum, len(df)):
    df.loc[i, f"{momentum}_momentum"] = df.loc[i, "close"] - df.loc[i-momentum, "close"]
    # Buy condition
    if df[f'{momentum}_momentum'].iloc[i] >= 0 and df[f'{momentum}_momentum'].iloc[i-1] < 0:
        df.loc[i, 'signal'] = 1
    # Sell condition
    elif df[f'{momentum}_momentum'].iloc[i] < 0 and df[f'{momentum}_momentum'].iloc[i-1] >= 0:
        df.loc[i, 'signal'] = -1
#...

# 아래의 모멘텀을 고려한다.
momentum_list = [30, 60, 90, 120, 150, 180]
```

#### 3.4 hash rate

흔히 사용되는 지표는 아니지만, 강환국 유튜브에서 접한 지표이다. 30일 평균 hashrate가 60일 평균 hashrate를 상승 돌파할 때 매수하고, 하락 돌파할 때 매도한다.

```python
df["HR_MA_30"] = df["Hashrate (TH/s)"].rolling(window=30).mean()
df["HR_MA_60"] = df["Hashrate (TH/s)"].rolling(window=60).mean()

df['signal'] = 0  # Default to no position
for i in range(26, len(df)):
    # Buy condition
    if df.loc[i, "HR_MA_30"] >= df.loc[i, "HR_MA_60"] and df.loc[i-1, "HR_MA_30"] < df.loc[i-1, "HR_MA_60"]:
        df.loc[i, 'signal'] = 1
    # Sell condition
    elif df.loc[i, "HR_MA_30"] < df.loc[i, "HR_MA_60"] and df.loc[i-1, "HR_MA_30"] >= df.loc[i-1, "HR_MA_60"]:
        df.loc[i, 'signal'] = -1
```

#### 3.5 변동성 돌파

금일 가격이 전일 변동량의 일정 부분(k) 이상 상승했을 때 매수, 매수 이후 매일 정해진 시간에 매도하는 전략이다.

```python
#...
df["range"] = df["high"] - df["low"]

for i in range(1, len(df)):
    df.loc[i, "target_price"] = df.loc[i-1, "range"] * k + df.loc[i, "open"]

# df["ma5"] = df["open"].rolling(window=5).mean()


df['signal'] = 0  # Default to no position
df['strategy_returns'] = 0.0
df['strategy_returns2'] = 0.0


for i in range(5, len(df)):
    # Buy condition
    if df.loc[i, "high"] >= df.loc[i, "target_price"]:
        df.loc[i, 'signal'] = 1
        df.loc[i, "strategy_returns"] = df.loc[i, "close"]/df.loc[i, "target_price"] - 1
        df.loc[i, "strategy_returns2"] = (df.loc[i, "close"] * 0.998)/(df.loc[i, "target_price"] * 1.002) - 1

# ...

# 고려하는 k 값
ks = [k / 10.0 for k in range(1, 10)]
```

---

다음 글에서는 위의 전략들을 백테스트하고 그 결과를 바탕으로 새로운 전략을 도출한다.

#### 같은 주제 게시글 모음

{{< related_posts tag="coin2024" >}}
