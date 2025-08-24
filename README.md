# 파이썬 Pandas/NumPy 실전 족보 (시험용 스니펫)

> 2시간/10문제 환경(주피터)에서 바로 붙여넣어 통과 가능한 최소·정답 지향 코드 모음. 불필요 출력·장황한 주석 지양, 에러 포인트는 한 줄 설명.

---

## 0. 공통
```python
import pandas as pd
import numpy as np
```

---

## 1) 문자열 빈도수 + 최빈 문자들
**요구**: 문자열 s에서 문자 빈도 dict, 최빈값, 최빈문자 리스트
```python
s = "skhynix"
frequency = {}
for ch in s:
    if ch in frequency:
        frequency[ch] += 1
    else:
        frequency[ch] = 1
max_frequency = max(frequency.values())
most_frequent_chars = []
for ch, cnt in frequency.items():
    if cnt == max_frequency:
        most_frequent_chars.append(ch)
```

---

## 2) `equipment` → `Sub` (언더바 분리 후 뒷부분 사용)
**요구**: `EQP_2_C` → `Sub` = `SUB_C`
```python
df = pd.read_csv("data.csv")
df['Sub'] = 'SUB_' + df['equipment'].str.split('_').str[-1]
```

### 2-1) Sub별 `Quantity` 합, 내림차순 정렬, 인덱스 초기화 → `df_sum`
```python
df_sum = (
    df.groupby('Sub', as_index=False)['Quantity']
      .sum()
      .sort_values(by='Quantity', ascending=False)
      .reset_index(drop=True)
)
```

---

## 3) `Equipment` → `Equipment2` (중간+뒷부분 추출 후 접두어)
**요구**: `EQ_2_C` → `Equipment2` = `EQP_2_C`
```python
df = pd.read_csv("data.csv")
parts = df['equipment'].str.split('_')
df['Equipment2'] = 'EQP_' + parts.str[1] + '_' + parts.str[2]
```

### 3-1) `Quantity*Time` → `Total Time`
```python
df['Total Time'] = df['Quantity'] * df['Time']
```

### 3-2) `Equipment2`별 `Quantity`, `Total Time` 합 → `df_sum`
```python
df_sum = (
    df.groupby('Equipment2', as_index=False)[['Quantity','Total Time']]
      .sum()
      .reset_index(drop=True)
)
```

### 3-3) 가중 시간(`Weighted Time`) 추가
```python
df_sum['Weighted Time'] = df_sum['Total Time'] / df_sum['Quantity']
```

### 3-4) `Weighted Time` 오름차순 상위 5개 (`Equipment2`, `Weighted Time`만)
```python
df_sum_top5 = (
    df_sum.sort_values(by='Weighted Time', ascending=True, ignore_index=True)
          .head(5)[['Equipment2','Weighted Time']]
)
```

---

## 4) 첫 번째 컬럼명 저장 후 삭제
```python
df = pd.read_csv("data.csv")
first_column = df.columns[0]
df = df.drop(columns=[first_column])
```

---

## 5) `ALIAS_LOT_ID.WF_ID` → `Groupkey` (WF_ID 2자리 0패딩)
```python
df['WF_ID'] = df['WF_ID'].astype(str).str.zfill(2)
df['Groupkey'] = df['ALIAS_LOT_ID'] + '.' + df['WF_ID']
```

### 5-1) 특정 키 필터링 → `df_filtered`
```python
df_filtered = df[df['Groupkey'] == 'MIM0071.01']
```

---

## 6) 각 컬럼의 고유값 개수 dict
```python
unique_counts = {col: df[col].nunique() for col in df.columns}
```

---

## 7) `df_operation`에서 `['Machine','Max']`별 `Damage` 합 → `df_sum`
```python
df_sum = (
    df_operation.groupby(['Machine','Max'], as_index=False)['Damage']
               .sum()
               .reset_index(drop=True)
)
```

### 7-1) 진행 가능량 = `floor(Max / Damage)` (정수형)
```python
df_sum['진행 가능량'] = np.floor(df_sum['Max'] / df_sum['Damage']).astype(int)
```

### 7-2) `Machine`, `진행 가능량`만 → `df_result`
```python
df_result = df_sum[['Machine','진행 가능량']]
```

---

## 8) 시계열 리샘플 – 월말 기준 첫/끝 가격, 월수익률
**입력**: `df` 열: `date, AAPL, GOOGL, MSFT`
```python
df['date'] = pd.to_datetime(df['date'])
df = df.set_index('date').sort_index()
first_prices = df.resample('M').first()
last_prices  = df.resample('M').last()
returns = (last_prices - first_prices) / first_prices * 100
```

### 8-1) 월별 최고 수익 종목 dict
```python
results = {}
for idx, row in returns.iterrows():
    key = f"{idx.year}-{idx.month}"
    stock = row.idxmax()
    value = round(row[stock], 1)
    results[key] = {'stock': stock, 'return': value}
```

---

## 9) 조건 필터 후 그룹 통계 (상/하위 25% 평균, 표준편차)
**입력**: `filtered_df = df[df['sales'] >= 50]`
```python
std_dev = (
    filtered_df.groupby('product')['sales']
               .std()
               .reset_index(name='std_dev')
)
upper_25 = (
    filtered_df.groupby('product')['sales']
               .apply(lambda x: x[x >= x.quantile(0.75)].mean())
               .reset_index(name='upper_25')
)
lower_25 = (
    filtered_df.groupby('product')['sales']
               .apply(lambda x: x[x <= x.quantile(0.25)].mean())
               .reset_index(name='lower_25')
)
result_df = std_dev.merge(upper_25, on='product').merge(lower_25, on='product')
result_df['gap'] = result_df['upper_25'] - result_df['lower_25']
max_gap_product = result_df.loc[result_df['gap'].idxmax(), 'product']
```

---

## 10) 날짜 변환, 정렬, 이동평균, 골든/데스 크로스
```python
# 날짜 변환 + 정렬
df['날짜'] = pd.to_datetime(df['날짜'])
df = df.sort_values(by=['주식 이름','날짜'])

# 이동평균
df['MA5']  = df['주식 가격'].rolling(window=5).mean()
df['MA20'] = df['주식 가격'].rolling(window=20).mean()

# 단순 상태 레이블(행 단위 비교)
def check_cross(row):
    if pd.isna(row['MA5']) or pd.isna(row['MA20']):
        return 'N/A'
    if row['MA5'] > row['MA20']:
        return '골든 크로스'
    elif row['MA5'] < row['MA20']:
        return '데스 크로스'
    else:
        return '동일'

df['CROSS SIGNAL'] = df.apply(check_cross, axis=1)

# (선택) 골든/데스 "발생 시점"만 True로 표시하려면 주식별로 이전 상태 비교
# df['MA5_gt_MA20'] = df['MA5'] > df['MA20']
# df['prev'] = df.groupby('주식 이름')['MA5_gt_MA20'].shift(1)
# df['골든_발생'] = (df['MA5_gt_MA20'] == True) & (df['prev'] == False)
# df['데스_발생']  = (df['MA5_gt_MA20'] == False) & (df['prev'] == True)
# df = df.drop(columns=['MA5_gt_MA20','prev'])
```

---

## 11) 기타 자주 나오는 한 줄들
```python
# 인덱스 초기화
df = df.reset_index(drop=True)

# 다중 정렬
df = df.sort_values(['col1','col2'], ascending=[True, False])

# 컬럼 선택
df2 = df[['A','B']]

# 조건 필터
df2 = df[(df['A'] > 0) & (df['B'] == 'x')]

# 결측 처리
df['x'] = df['x'].fillna(0)

# 그룹 집계
g = df.groupby('key', as_index=False)['val'].agg(['sum','mean']).reset_index()
```

---

### 빠른 체크 포인트(실패 빈도 높은 곳)
- `reset_index(name='...')`는 **Series**에만 동작. DataFrame이면 `.rename(...).reset_index()`
- `sort_values`는 `by=` 명시 안전.
- 시간 리샘플은 인덱스가 datetime이어야 함: `set_index('date')` 후 사용.
- 0으로 나눔 주의: 필요 시 `replace(0, np.nan)` 후 계산.
- 문자열 분할 시 인덱스 범위: `str.split('_').str[i]`에서 i 범위 확인.

