date: 2019-08-31 11:57:14
title: 크레딧잡 연봉상위 데이터는 어떻게 표시될까
tags: ['kreditjob', 'percentile', 'nps', 'curve_fit', 'cdf']
layout: post


![Image](/static/images/kreditjob_percentile/2019-08-31-11:47.png)

크레딧잡은 평균연봉, 입사율, 퇴사율 등 매달 공개되는 국민연금 데이터를 기반으로 전국 45만 기업의 고용정보를 제공해온 서비스입니다.

스크린샷과 같이 크레딧잡에서 보여지는 연봉상위 % 데이터를 구하기 위해 데이터팀과 서버 팀이 협업을 하게된 스토리를 공유합니다.

## 팀 협업

서버팀과 데이터팀의 역할이 나뉘게 되는 경우는 다양합니다. 보통 데이터 수집, 관리, 가공하는 일, 제공하는 서비스 전반에 활용될 수 있도록 데이터를 분석하여 팀에 공유하는 일 등을 데이터팀이 맡아서 합니다. 서버팀은 새로운 기능 개발이나 서비스 시스템 관리를 안정적으로 하는 일을 맡아서 합니다.

크레딧잡의 연봉 상위 데이터를 표시할 경우는 서버팀이 API 를 통해 데이터를 표시하는 기능과 관련된 부분을 다루고, 데이터팀이 연봉 상위를 표시할 때 필요한 데이터를 전달하는 역할로 나눌 수 있습니다.

## Percentile(백분위수)

[백분위수](https://ko.wikipedia.org/wiki/%EB%B0%B1%EB%B6%84%EC%9C%84%EC%88%98)는 값들을 나열했을 때 백분율로 나타낸 특정 위치의 값을 이르는 용어입니다. 백분위수는 정렬을 작은 값에서 큰 값 순서대로 나열하므로 백분위수 99% 가 연봉상위 1% 라고 생각하시면 되겠습니다.

45만개 기업의 백분위수를 각각 구하는 것은 간단합니다. Python의 통계 라이브러리인 numpy, pandas 의 함수만으로 아래와 같이 사용하면 됩니다.

<div class="def">데이터베이스 및 코드 내부를 공개할 수는 없기에 예제를 변형했습니다.</div>

```python
import numpy as np
import pandas as pd

# fetch average salary data
salary_info = pd.read_sql("select company_id, avg_salary from company", sqlalchemy_engine)
# percentile threshold integer value i%
salary_percentile = [np.percentile(salary_info.avg_salary, i) for i in range(1, 100)]
```

코드는 간단하지만 크리티컬한 문제가 있습니다.

서비스를 이용하는 세션 하나하나가 저 코드를 실행한다면 매번 모든 연봉 데이터를 가져와야 하기 때문에 금방 데이터베이스 서버에 과부하가 걸리게 됩니다.

## 여러가지 해결책들

이 문제를 해결하기 위해 여러가지 방법이 있습니다. 예를 들면

1. Redis 같은 In-memory 데이터베이스에 `salary_info` 를 캐싱해 놓는다.
1. 기업 정보가 있는 테이블에 percentile 컬럼을 추가한다.
1. 1부터 100까지 정수에 대응되는 percentile 테이블을 추가한다.

이 중에서는 1번의 방법이 그나마 가장 좋아보입니다. 하지만 이는 서버에서 작업해야 할 내용입니다. 데이터 업데이트 시점에 맞추어 sync 하는 작업이 까다롭게 됩니다.

2번 같은 경우는 기업 정보가 있는 테이블 성격과 맞지 않는 불필요한 aggregation 컬럼이 생깁니다. 또한 크레딧잡 데이터가 한달에 한 번 정도 국민연금 가입 사업장 내역이 업데이트 되는데 그때마다 매 row 를 업데이트 해야 하기 때문에 시간이 매우 오래 걸릴 것입니다.

3번 같은 경우 서버에서 구현이 까다로워집니다. 예를 들어 메타정보를 제외하고 아래와 같은 테이블이 있다고 가정할 수 있습니다.

avg_salary | percent
---------- | -------
20000000 | 100
21421320 | 99
… | …

지금 이 글을 읽으시는 분이 서버를 개발한다고 가정해 봅시다.

> 이 데이터를 사용해서 A 기업이 평균연봉이 3000만원이고, B 기업이 평균연봉이 5억일 경우 percentile 값을 어떻게 가져와야 할까요?

어쨌든 이 테이블을 매 세션마다 모두 조회해서 구간값을 계산해야 할 겁니다. 심지어 상위 1%는 오픈 구간(연봉 값이 무한대로 가능한 구간)이라 다르게 처리해야 합니다.

## 곡선 피팅

위의 3번 방법에 대해 좀 더 고민을 하던 중, 곡선 피팅이라는 방법을 적용하게 됐습니다.

핵심은 연봉을 Input 으로 넣었을 때 percentile 을 output 으로 하는 함수를 만드는 것입니다.

꼭 함수가 프로그래밍에서의 프로시저가 될 필요는 없습니다.

데이터를 파악하기 위해 그 동안 국민연금에서 받았던 월별 데이터를 쭉 그려보았습니다.

<div class="def"><a href="https://www.data.go.kr/dataset/3046071/fileData.do">공개데이터</a>를 사용했으니 한 번 직접 해보셔도 됩니다.</div>

<div class="warn">국민연금 고지 금액과 평균연봉은 다릅니다. 실제 크레딧잡에서는 평균연봉 데이터로 하기 때문에 방식이 다를 수 있습니다.</div>

```python
from glob import glob

import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns

nps_path = glob('국민연금*.csv')
frames = [pd.read_csv(path, encoding='cp949', usecols=[0, 22-4, 22-3]) for path in nps_path]
plots = []
for df in frames:
    plot = pd.DataFrame()
    paid = df.iloc[:, 2] / df.iloc[:, 1] / 1e5
    plot['paid'] = paid.describe(percentiles=np.arange(0, 1.01, 0.01))
    plot['date'] = df.iat[0, 0]
    plot = plot.loc[plot.index.str.contains('%')]
    plot.index = (100 - plot.index.str.strip('%').astype(float)) / 100
    plots.append(plot)
nps = pd.concat(plots, sort=False).reset_index()

sns.set(style='white', font_scale=.8)
plt.clf()
sns.relplot(x='paid', y='index', hue='date', alpha=.2, data=nps, s=10)
plt.show()
```

![Image](/static/images/kreditjob_percentile/2019-08-31-11:53.png)

<div class="info">분산 값을 줄이기 위해 정의역을 1e5(100000) 으로 나눈 (0, 10)  구간으로 제한했습니다.</div>

세 달간 데이터를 그려 봤을때 놀라울 정도로 곡선 모양이 일치했습니다.

그래프의 성질은 대체로 paid가 0에 가까울수록 1에 가까워지고, 무한대에 접근할수록 percentile이 0에 가까워집니다.

이는 그래프가 [누적 분포 함수(Cumulative Distribution Function)](https://en.wikipedia.org/wiki/Cumulative_distribution_function) 양상과 거의 정확히 일치함을 알 수 있습니다.

따라서 누적 분포 함수식으로 곡선을 피팅한 후 서버팀에게 공식과 계수만 전달하면 되는 것입니다.

```python
import math

from scipy.special import erf
from scipy.optimize import curve_fit


def cdf(x, a, b):
    return 1 / 2 * (1 + erf(-x + a) / (b * math.sqrt(2)))


def fit(data_ym):
    nps6 = nps.query(f'date == {data_ym}')
    return curve_fit(cdf, nps6['paid'], nps6['index'], absolute_sigma=True)

(a, b), pcov = fit(201906)

plt.clf()
sns.relplot(x='paid', y='index', hue='date', alpha=.2, data=nps, s=10, legend=None)
sns.lineplot(x=np.arange(0, 5, 0.1), y=[cdf(x, a, b) for x in np.arange(0, 5, 0.1)], legend=None)
plt.show()
```

![Image](/static/images/kreditjob_percentile/2019-08-31-11:54.png)

이대로 끝나도 괜찮지만 몇가지 고려해야 할 사항이 있습니다.

- 아무래도 데이터가 평균에 몰려있다보니 피팅하는 과정에서 연봉 상위 구간 쪽(1% ~ 20%)의 데이터가 정확하지 않습니다.

*크레딧잡에서는 연봉 상위에 해당하는 기업들을 정확하게 구분하기 위해 상위 1% ~ 50% 데이터가 정확해야 합니다. 따라서 위에서 적용한 방식의 튜닝이 필요했습니다.따라서 실제로 피팅하는 정의역을 제한하고 함수식을 변형했습니다. 그 결과 ±1% 내로 정확해 졌습니다.*

- `from scipy.special import erf` 서버 쪽에 주게되는 함수가 `cdf` 가 될 텐데, `scipy` 패키지가 웹 서버에 필요할까요? 기본 라이브러리를 사용한 `math.erf` 가 있지만 곡선 피팅 시 상수가 아니라 함수라서 vector 연산이 되지 않기 때문에 아래와 같은 `server_cdf` 함수로 분리할 필요가 있습니다.

```python
def server_cdf(x, a, b):
    """서버 팀에게 전달할 기본 라이브러리로 이루어진 공식"""
    return 1 / 2 * (1 + math.erf(-x + a) / (b * math.sqrt(2)))
```

따라서 위 공식을 처음 서버팀에게 한번만 전달하면 되고, `a`, `b` 값을 매달 서버에 전달하는 것입니다.

## 결론

문제를 푸는 방법은 여러가지가 있습니다.

이 경우 머신러닝을 적용할 수도 있었고 앞서 곡선을 피팅하기 전에 소개드린 방법대로 진행할 수도 있었습니다.

하지만 협업을 통해 커뮤니케이션 하는 과정에서 각각의 불필요한 작업이 생기기 마련이며 이를 최소화 하는 올바른 방향이 필요했습니다.

원티드랩의 서버팀은 데이터베이스 및 서버 최적화를 끊임없이 고민하고, 불필요한 패키지 설치 및 작업을 최소화 하려고 노력합니다.

따라서 데이터 파이프라인에서 곡선 피팅을 진행하고, 서버팀에서 처음 공식과 매달 계수를 전달하는 방식으로 컴팩트하게 진행이 되었습니다.(실제 코드도 몇 줄 안됩니다.)

머신러닝을 사용하면 많은 시간이 들지만, 곡선 피팅 방식을 사용하면 시간은 덜 투입되면서도 정확도는 높은 경우가 많습니다. 시간 및 여건에 알맞은 도구를 사용하는 것이 중요하다고 느꼈습니다.

## 참고

- [https://kr.mathworks.com/help/matlab/ref/erf.html](https://kr.mathworks.com/help/matlab/ref/erf.html)

- [http://mathworld.wolfram.com/Erf.html](http://mathworld.wolfram.com/Erf.html)

- [https://docs.scipy.org/doc/scipy/reference/generated/scipy.optimize.curve_fit.html](https://docs.scipy.org/doc/scipy/reference/generated/scipy.optimize.curve_fit.html)
