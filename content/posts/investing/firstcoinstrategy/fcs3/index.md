+++
title = '코인2024[2] - 전략 성능 평가 및 응용 전략 개발'
date = 2024-09-09T14:39:39+09:00
categories = ["investing"]
tags = ["coin2024", "coin", "investing", "research"]
toc = true
draft = false
+++

이번 글에서는 조사한 전략 성능 평가와 평가를 바탕으로 응용 전략을 개발한다.

<!--more-->

## 자료 수집

### 업비트 api 활용 가격 데이터 불러오기

업비트 거래소의 api를 활용하여 코인 가격 데이터를 수집한다.

아래의 get_candles 함수는 필요한 데이터의 시작 시점과 기타 필요한 인자를 입력하면 시작 시점부터 현재까지 가격 데이터를 불러온다. 업비트 api는 1회 요청당 200개를 초과하는 요청을 할 수 없고, 초당 30회 초과 요청을 할 수 없는 부분을 고려하여 코드를 작성했다.

```python
# handle_candle.py 중 일부

def get_candles(
    interval="minutes",
    market="KRW-BTC",
    count="200",
    start="2024-01-01 01:00:00",
    interval2="1",
):
    times = get_time_intervals(
        initial_time_str=start,
        interval=interval,
        interval2=interval2,
    )
    if len(times) == 1:
        # Get the current time
        current_time = str(datetime.now(timezone.utc))
        current_time = current_time.split(".")[0]
        current_time = datetime.strptime(current_time, "%Y-%m-%d %H:%M:%S")
        start_time = datetime.strptime(start, "%Y-%m-%d %H:%M:%S")
        start_time = start_time - timedelta(hours=9)
        time_difference = current_time - start_time
        # print(time_difference)

        # Convert the time difference to total minutes
        total_minutes = time_difference.total_seconds() // 3600 + 1
        count = int(total_minutes)
        # print(count)

    lst = []
    rate_limit_interval = 1 / 20  # Time in seconds to wait between requests

    for t in times:
        url = f"https://api.upbit.com/v1/candles/{interval}/{interval2}?market={market}&count={count}&to={t}"

        headers = {"accept": "application/json"}

        response = requests.get(url, headers=headers)
        response = ast.literal_eval(response.text)
        lst += response
        # Wait to respect the rate limit
        time.sleep(rate_limit_interval)
        # print(response, type(response))
        if type(response) == dict:
            print(response)
            return None

    lst = remove_duplicates(lst)
    # Sort the list of dictionaries by 'candle_date_time_utc'
    sorted_list = sorted(lst, key=lambda x: x["candle_date_time_utc"])

    df = pd.DataFrame(sorted_list)
    selected_columns = [
        "market",
        "candle_date_time_kst",
        "opening_price",
        "high_price",
        "low_price",
        "trade_price",
        "candle_acc_trade_price",
        "candle_acc_trade_volume",
    ]
    df_selected = df[selected_columns].copy()

    df_selected.rename(
        columns={
            "candle_date_time_kst": "time_kst",
            "opening_price": "open",
            "high_price": "high",
            "low_price": "low",
            "trade_price": "close",
            "candle_acc_trade_price": "volume_krw",
            "candle_acc_trade_volume": "volume_market",
        },
        inplace=True,
    )
    return df_selected


```

---

### 시가 종가 기준 정하기

get_candles 함수를 통해 원하는 기간의 시간당 가격 데이터를 확보했다면, 그것을 하루 기준 데이터로 바꾸는 작업을 해야한다. 업비트 api를 통해 하루 기준 데이터를 곧바로 받을 수도 있지만, UTC 0시(9시) 기준으로 제공하기 때문에 다른 시간대에 대한 데이터를 구하기 어렵다. 더구나 9시 기준은 이미 많은 자동 매매 기계들이 기준으로 선정하고 정각 기준 매수, 매도 시에 슬리피지가 타 시간대에 비해 높을 것이라고 상정했다. 따라서 새로운 기준시를 설정해야했다.

9시부터 3시. 거래 시간이 정해져있는 주식 시장과 달리 코인은 24/7 운영한다는 것이 특징이다. 따라서 시가, 종가를 구분하는데 주관이 들어갈 수밖에 없는데, 나는 거래량이 가장 적은 시간대를 기준으로 하루하루를 나누기로 했다. 거래량이 가장 낮을 때부터 가장 높은 때를 거쳐 다시 가장 낮은 때까지의 데이터가 하루의 가격을 가장 잘 표현할 것이라고 생각했기 때문이다. 주식을 생각하더라도 하루를 구분 하는 것은 거래량이 0인 시간들이다.

```python
import pandas as pd

# Assuming your DataFrame is named df and the 'time_kst' column is already in datetime format
df['hour'] = df['time_kst'].dt.hour  # Extract hour from time_kst

# Group by the 'hour' column
grouped_by_hour = df.groupby('hour').agg({
    'open': 'mean',        # Example: Calculate the average 'open' price per hour
    'close': 'mean',       # Example: Calculate the average 'close' price per hour
    'volume_market': 'mean'    # Example: Calculate the total volume per hour
})

# Reset index if you want the hour to be a column again
grouped_by_hour = grouped_by_hour.reset_index()

# Display the result
print(grouped_by_hour)
```

위의 코드를 통해 아래의 결과를 얻었다.

```output
    hour          open         close  volume_market
0      0  8.209518e+07  8.199403e+07     230.753713
1      1  8.198445e+07  8.206782e+07     142.130093
2      2  8.207024e+07  8.203003e+07     101.746834
3      3  8.202853e+07  8.195905e+07      96.986020
4      4  8.223710e+07  8.224954e+07      97.926199
5      5  8.225085e+07  8.229677e+07     109.404200
6      6  8.229964e+07  8.235667e+07     156.290143
7      7  8.235418e+07  8.231408e+07     178.391202
8      8  8.230631e+07  8.212413e+07     166.067811
9      9  8.212238e+07  8.215585e+07     265.116783
10    10  8.238482e+07  8.234353e+07     265.593120
11    11  8.233866e+07  8.232726e+07     175.199980
12    12  8.232679e+07  8.222097e+07     125.965980
13    13  8.221874e+07  8.210666e+07     130.761464
14    14  8.210850e+07  8.208426e+07     159.409739
15    15  8.208653e+07  8.214166e+07     173.807352
16    16  8.213476e+07  8.215863e+07     166.615198
17    17  8.215642e+07  8.215600e+07     122.398108
18    18  8.215797e+07  8.217637e+07     107.269996
19    19  8.217379e+07  8.214097e+07     113.517471
20    20  8.213921e+07  8.220413e+07     116.285750
21    21  8.220008e+07  8.222984e+07     179.321342
22    22  8.223139e+07  8.218176e+07     212.871202
23    23  8.217453e+07  8.208797e+07     355.349909
```

모든 시간대별 거래량의 통계를 낸 결과 새벽 3시~5시가 가장 거래량이 적고, 오후 11시가 가장 거래량이 높았다. 따라서 오전 4시를 기준으로 데이터를 정리했다.

```python
# Step 1: Set the index to 'time_kst'
df.set_index('time_kst', inplace=True)

# Step 2: Resample the data starting at 4:00 AM each day
# This will give daily intervals starting from 4:00 AM
daily_df = df.resample('24H', offset='4H', origin='epoch').agg({
    'open': 'first',           # Open price at 4:00 AM
    'high': 'max',             # Highest price in the interval
    'low': 'min',              # Lowest price in the interval
    'close': 'last',           # Close price at 4:00 AM next day
    'volume_krw': 'sum',       # Total volume in the interval
    'volume_market': 'sum'     # Total market volume in the interval
}).reset_index()

# Step 3: Rename the time column
daily_df.rename(columns={'time_kst': 'day_starting_at_4am'}, inplace=True)

# Display the result
print(daily_df)
```

```output
   day_starting_at_4am        open        high         low       close  \
0  2024-08-01 04:00:00  92986000.0  93101000.0  88001000.0  88779000.0
1  2024-08-02 04:00:00  88751000.0  92300000.0  88320000.0  88710000.0
2  2024-08-03 04:00:00  88737000.0  89170000.0  85158000.0  85160000.0
3  2024-08-04 04:00:00  85181000.0  86311000.0  81756000.0  83292000.0
4  2024-08-05 04:00:00  83292000.0  84333000.0  72100000.0  76349000.0
5  2024-08-06 04:00:00  76349000.0  82150000.0  75777000.0  80687000.0
6  2024-08-07 04:00:00  80687000.0  81950000.0  78260000.0  78410000.0
7  2024-08-08 04:00:00  78411000.0  84071000.0  77782000.0  83332000.0
8  2024-08-09 04:00:00  83332000.0  87980000.0  83236000.0  84889000.0
9  2024-08-10 04:00:00  84850000.0  86370000.0  84512000.0  85650000.0
10 2024-08-11 04:00:00  85650000.0  86424000.0  84150000.0  84657000.0
...

```

거래량이 가장 적은 시간대를 24시간 운영하는 시장의 하루하루를 구분하는 기준으로 사용한다는 점에서 논리는 나쁘지 않다고 보인다. 하지만 투자 전략 상 기준 시간대에 거래를 많이 해야하는 것을 고려했을 때, 오히려 거래량이 가장 많은 시간대를 기준으로 하는 것이 거래 비용을 감소시킬 수도 있겠다는 여지가 있다. 이는 추후에 연구해 보겠다.

---

### 백테스트 기간 정하기

백테스트 기간에 따라 투자 전략의 성능이 천차만별이므로 기간을 설정하는 것은 매우 중요하다. 이번 연구에서는 비트코인이 개발되고 가격이 상승만 하다가 처음 대하락장을 경험하기 직전인 2021년 4월부터 현 시점인 2024년 8월까지로 설정했다. 그 이후의 가격 패턴이 하락장, 횡보장, 상승장의 성격을 가진 구간이 모두 잘 나타나기 때문이다.

{{< image_caption src="chart.png" alt="비트코인 가격 데이터" caption="비트코인 가격 데이터" >}}

---

## 전략 평가 도구

본격적인 전략 평가에 앞서 전략들을 일률적으로 평가하기 위한 평가 도구를 소개한다.

우선 평가 지표는 다음과 같다.

1. 연평균수익률(CAGR)
2. 최대 손실률(MDD)
3. 총 투자 기간(첫 번째 매수 시점으로부터)

각기 다른 데이터셋에 대해 위의 성능 지표들을 확인하기 위해 아래와 같은 get_performance 함수를 작성하였다.

```python
# performance.py 중 일부

def get_performance(df, title, add_to_excel=False, file_path=None):
    # get total return
    initial_value = df["cumulative_returns"].iloc[1]
    final_value = df["cumulative_returns"].iloc[-1]
    tr = get_total_return(
        inital_value=initial_value,
        final_value=final_value,
    )

    # # get cagr
    # days = len(df)

    # Calculate the number of days from the first signal to the last day in the DataFrame
    # Find the first day where a position is taken (signal == 1)
    first_signal_day = df[df["signal"] == 1].index[0]

    # Calculate the number of days from the first signal to the last day in the DataFrame
    days = len(df) - first_signal_day
    cagr = get_cagr(
        total_return=tr,
        days=days,
    )
    # print(len(df), first_signal_day, days)

    # get mdd
    balance = df["cumulative_returns"]
    mdd = get_mdd(balance)

    try:
        initial_value2 = df["cumulative_returns2"].iloc[1]
        final_value2 = df["cumulative_returns2"].iloc[-1]
        tr2 = get_total_return(
            inital_value=initial_value2,
            final_value=final_value2,
        )

        cagr2 = get_cagr(
            total_return=tr2,
            days=days,
        )

        balance2 = df["cumulative_returns2"]
        mdd2 = get_mdd(balance2)

        print_things(
            strategy=title,
            total_return=tr,
            cagr=cagr,
            mdd=mdd,
            total_return_w_fee=tr2,
            cagr_w_fee=cagr2,
            mdd_w_fee=mdd2,
            investing_days=days,
        )

        if add_to_excel:
            new_data = {
                "strategy_name": title,
                "cagr": get_rounded(cagr2),
                "mdd": get_rounded(mdd2),
                "investing_days": days,
            }
            add_row_to_excel(file_path=file_path, new_data=new_data)

    except:
        print_things(
            strategy=title,
            total_return=tr,
            cagr=cagr,
            mdd=mdd,
            investing_days=days,
        )

```

투자 시작 시점이 아니라 매수 시작 시점부터 계산하기 때문에 CAGR이 높게 나올 수는 있지만, 전체 데이터 길이를 기준으로 판단했을 때, 이동평균선 180일과 같은 오랜 시간이 지난 후에야 시작되는 전략과 이동평균선 5일과 같은 곧바로 시작할 수 있는 전략 간의 실질 투자 시간이 차이나기 때문에 매수 시점을 기준으로 투자 일수를 정했다. 추후에는 200일 이상 버퍼 데이터를 확보하고, 백테스트를 실행해야겠다.

## 전략 평가

벤치마크인 매수 후 보유 전략의 성능은 다음과 같다.

Strategy : Buy&Hold

total_return : 15.71

cagr : 4.4

mdd : 74.23

그리고 다양한 전략들의 성과는 다음과 같다.

{{< image_caption src="results2-1.jpg" alt="성능 평가 결과 1" caption="성능 평가 결과 1" >}}
{{< image_caption src="results2-2.jpg" alt="성능 평가 결과 2" caption="성능 평가 결과 2" >}}
{{< image_caption src="results2-4.png" alt="성능 평가 결과 3" caption="성능 평가 결과 3" >}}

자동 매매에서 변동성돌파전략(Volatility Breakout)을 많이 활용하는 가운데, 의외로 성과 지표가 좋지 않고, 이동평균선이나, 모멘텀을 결합한 전략들에서 그 성능을 보여준다.
또 가장 간단한 이동평균선이나 모멘텀 전략이 높은 성능을 발휘하는 것으로 보이는 와중에, rsi에 최고가 대비 스탑 로스 5%를 포함하는 전략이 잘 작동하는 것을 볼 수 있다.

이후 추가적인 백테스트에서 rsi 전략은 일정 수준 이상 하락한 이후에 반등하는 것을 노리는 전략으로, 이동평균선이나 모멘텀 전략과 결합했을 때 시너지 효과가 잘 나지 않는 것을 확인했다. 이는 좋은 가격 모멘텀을 가질 때 매수에 진입하는 이동평균선과 모멘텀 전략과 조합이 좋지 않기 때문인 것으로 보인다. 정리해서 말하자면 rsi는 하락장에서 상승장으로 돌입할 때, 이동평균선은 상승 또는 횡보 장에서 더 큰 상승장으로 돌입할 때 매수 진입하므로 코인 투자금 둘 이상으로 분리해 병렬로 전략들을 사용하면 상관관계가 조금이나마 떨어지는 효과를 얻을 수 있을 것으로 보인다.

## 추후 개선점

1. 오히려 거래량이 가장 많은 시간대를 기준으로 하는 것이 거래 비용을 감소시킬 수도 있겠다는 여지가 있다.
2. 추후에는 200일 이상 버퍼 데이터를 확보하고, 백테스트를 실행해야겠다.

#### 같은 주제 게시글 모음

{{< related_posts tag="coin2024" >}}
