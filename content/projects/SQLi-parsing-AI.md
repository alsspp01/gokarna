---
title: "SQLi parsing AI"
type: page
---

### 제작 계기
SQL injection query는 사람이 보면 구분할 수 있지만, 기본적으로 컴퓨터는 구분하지 못한다.  
이에 따라 SQL injection query를 구분해내는 AI 모델을 생성하여 SQL injection 내부 대응책을 설립하기 어려운 중소기업 웹 개발자들에게 도움을 주고자 한다.

### SQL injection이란?

웹 애플리케이션과 데이터베이스 간의 연동에서 발생하는 취약점을 이용해,  
공격자가 입력 폼에 악의적으로 조작된 쿼리를 삽입해  
데이터베이스 정보를 불법적으로 열람하거나 조작하는 공격  

### Overview

1. 구문 분석을 위해 Secquence Data로 전처리
2. 전처리 시 문법적인 부분을 강조하기 위해 기호, SQL 문법 구성 단어를 기준으로 feature 추출
3. 기호, SQL언어, 변수, 처리되지 않은 일반 string 순으로 분류하여 토큰화, 정규화
4. Sezuence Data를 처리할 수 있는 모델 - LSTM(Long Short-Term Memory) 사용
5. F1 Score 등을 통해 검증

### Process

#### 1. 사용한 모듈


```python
import pandas as pd
import re
import numpy as np
from sklearn.model_selection import train_test_split
from tensorflow.keras.models import Sequential
from tensorflow.keras.layers import Embedding, LSTM, Dense, Dropout, BatchNormalization
from tensorflow.keras.callbacks import EarlyStopping
from tensorflow.keras.preprocessing.sequence import pad_sequences
import matplotlib.pyplot as plt
```


#### 2. 데이터 구조 분석

```python
# CSV 파일 읽어오기
data = pd.read_csv("./clean_sql_dataset.csv")

# 데이터프레임 구조 확인
print(data.head())

# 데이터프레임의 컬럼 이름 확인
print(data.columns)

# Label 값의 개수 세기
label_counts = data['Label'].value_counts()
print(label_counts)
```

##### 실행 결과

![데이터구조분석](/image/sqli-parsing-ai/capture01.png)


#### 3. Tokenize

```python
def tokenize_query(query):
    # 정규 표현식 패턴 정의
    token_pattern = re.compile(r'''
    (--|/\*|\*/)                         # 주석 기호만
    |([+\-*/%&|^~=<>!]=?)                # 연산자
    |([(),;])                            # 구분자
    |([.\[\]{}])                         # 특수 문자
    |(\b\w+\b)                           # 단어 (키워드, 변수명 등)
    |([^\s])                             # 기타 모든 문자
    ''', re.MULTILINE | re.DOTALL | re.VERBOSE)
    
    # 패턴 적용하여 토큰화
    tokens = token_pattern.findall(query)
    
    result_tokens = []
    for token_tuple in tokens:
        for token in token_tuple:
            if token:
                result_tokens.append(token.strip())
    
    return result_tokens
```

정규표현식으로 주석, 연산자, 구분자, 특수문자와 같은 기호와 일반 단어를 구분한다.

##### 실행 결과

![토큰화](/image/sqli-parsing-ai/capture02.png)

#### 4. Normalize

```python
def normalize_query(tokens):
    normalized_tokens = []
    
    sql_keywords = {
        "SELECT", "FROM", "WHERE", "INSERT", "INTO", "VALUES", "UPDATE", "SET", "DELETE",
        "JOIN", "INNER", "OUTER", "LEFT", "RIGHT", "FULL", "ON", "AND", "OR", "NOT",
        "NULL", "LIKE", "IN", "EXISTS", "BETWEEN", "GROUP", "BY", "ORDER", "HAVING",
        "LIMIT", "OFFSET", "DISTINCT", "AS", "CASE", "WHEN", "THEN", "ELSE", "END",
        "CREATE", "TABLE", "PRIMARY", "KEY", "FOREIGN", "REFERENCES", "DROP", "ALTER",
        "ADD", "COLUMN", "INDEX", "VIEW", "TRIGGER", "PROCEDURE", "FUNCTION", "DATABASE"
        }
    
    special_characters = re.compile(r'''
        (--|/\*|\*/)                            # 단일 행 주석, 여러 행 주석 닫기
        |([+\-*/%&|^~=<>!]=?)                  # 비교 연산자 및 대입 연산자
        |([.,;(){}\[\]'"`])                    # 특수 문자 및 구분자
        ''', re.MULTILINE | re.DOTALL | re.IGNORECASE | re.VERBOSE)
    
    for token in tokens:
        
        # 특수문자는 따로 처리
        if re.match(special_characters, token):
            normalized_tokens.append(token)
        
        # SQL 키워드는 대문자로 통일
        elif token.upper() in sql_keywords:
            normalized_tokens.append(token.upper())
        
        # 숫자는 'NUM'으로 통일
        elif re.match(r"-?\d*\.?\d+", token):
            normalized_tokens.append("NUM")
            
        # 변수명은 'VAR'로 통일
        elif re.match(r"^[@#]?\w+$", token):
            normalized_tokens.append("VAR")
        
        # 나머지는 'STR'로 통일 (문자열 취급)
        else:
            normalized_tokens.append('STR')
    
    return normalized_tokens
```

기호를 1차 분류하고, SQL 키워드를 2차 분류, 숫자를 3차 분류, 변수는 VAR, 혹시 처리되지 않은 문자가 있으면 STR로 분류했다.


##### 실행 결과

![정규화](/image/sqli-parsing-ai/capture03.png)


#### 5. Vectorize

```python
def tokens_to_vector(tokens, token_to_index, max_length):
    vector = np.zeros(max_length)  # 최대 길이만큼 벡터 초기화
    for i, token in enumerate(tokens):
        if token in token_to_index:   # 토큰이 인덱스에 있는 경우에만 처리
            index = token_to_index[token]
            vector[i] = index + 1     # 인덱스를 1부터 시작하게 조정
    return vector

# vocabulary 만들기
vocabulary = set(token for tokens in data['normalized_query'] for token in tokens)
token_to_index = {token: i for i, token in enumerate(vocabulary)}

# 최대 길이 설정
max_length = max(len(tokens) for tokens in data['normalized_query'])
```

최대 길이의 문장에 길이를 맞춰 벡터를 초기화하고  
각 단어 및 기호를 vocabulary에 넣어 번호를 매긴 후  
그 번호를 벡터에 차례로 one hot encoding한다.

이를 통해 시퀀스 데이터를 생성할 수 있다.


##### 실행 결과

![벡터화](/image/sqli-parsing-ai/capture04.png)


#### 5. Learning

```python
vocab_size = len(vocabulary)

# 훈련 및 테스트 세트 분할
X_train, X_test, y_train, y_test = train_test_split(data['query_vector'], data['Label'], test_size=0.2, random_state=42)

# RNN 모델 정의
model = Sequential([
    Embedding(input_dim=vocab_size+1, output_dim=100, input_length=max_length, mask_zero=True),
    LSTM(128, return_sequences=True),
    Dropout(0.5),
    LSTM(64),
    Dropout(0.5),
    Dense(64, activation='relu'),
    BatchNormalization(),
    Dense(1, activation='sigmoid')
])

# 모델 컴파일
model.compile(optimizer='adam', loss='binary_crossentropy', metrics=['accuracy'])

X_train = pad_sequences(X_train, maxlen=max_length)
X_test = pad_sequences(X_test, maxlen=max_length)

# 조기 종료 설정
early_stopping = EarlyStopping(monitor='val_loss', patience=3, restore_best_weights=True)

# 모델 훈련
history = model.fit(X_train, y_train, epochs=50, batch_size=128, validation_split=0.1, callbacks=[early_stopping])

# 모델 평가
loss, accuracy = model.evaluate(X_test, y_test)
print("테스트 정확도:", accuracy)

# 학습 과정 시각화
plt.figure(figsize=(12, 5))

# 훈련 및 검증 데이터에 대한 정확도 그래프
plt.subplot(1, 2, 1)
plt.plot(history.history['accuracy'], label='accuracy')
plt.plot(history.history['val_accuracy'], label='val_accuracy')
plt.xlabel('Epoch')
plt.ylabel('Accuracy')
plt.legend(loc='lower right')
plt.title('Training and Validation Accuracy')

# 훈련 및 검증 데이터에 대한 손실 그래프
plt.subplot(1, 2, 2)
plt.plot(history.history['loss'], label='loss')
plt.plot(history.history['val_loss'], label='val_loss')
plt.xlabel('Epoch')
plt.ylabel('Loss')
plt.legend(loc='upper right')
plt.title('Training and Validation Loss')
plt.show()
```

LSTM을 사용하여 시쿼스 데이터를 학습시킨다.  
1차적으로 학습시켰을 때에는 overfitting이 의심되는 결과가 나왔다. (아래 그림)

![과적합](/image/sqli-parsing-ai/capture05.png)

이에 따라 Dropout 및 BatchNormalization을 추가하여 실행해보았다.
또한 대부분 0으로 패딩되어 있는 데이터를 학습시키므로 mask_zero를 True로 설정하였다.

![시각화](/image/sqli-parsing-ai/capture06.png)

Epoch를 Heuristic하게 결정(7)

#### 6. Validation

![predict](/image/sqli-parsing-ai/capture07.png)

data['query vector']를 tensorflow Tensor로 변환  
이후 입력 Tensor에 대한 예측값을 반환 → 0 ~ 1의 연속적인 확률값 → 임계값 0.5로 설정
pred list와 data[Label]을 dataframe으로 변환 후 인덱스 초기화  
모델의 예측값과 실제 레이블이 일치하는 경우 True, 일치하지 않을 경우 False 비율 계산  
⇒ True: 0.995... / False: 0.004...

##### 다른 데이터셋으로 검증

Kaggle 내의 다른 SQL injection Dataset을 모델에 넣어 정상 작동하는지 재확인

```
# 토큰화 적용
ex_data['tokenized_query'] = ex_data['Query'].apply(tokenize_query)

# 정규화 적용
ex_data['normalized_query'] = ex_data['tokenized_query'].apply(normalize_query)

# 벡터화 적용
ex_vocabulary = set(token for tokens in ex_data['normalized_query'] for token in tokens)
ex_token_to_index = {token: i for i, token in enumerate(vocabulary)}
ex_max_length = max(len(tokens) for tokens in ex_data['normalized_query'])
ex_data['query_vector'] = ex_data['normalized_query'].apply(lambda tokens: tokens_to_vector(tokens, ex_token_to_index, ex_max_length))

# tensor로 변환
ex_tensor = tf.convert_to_tensor(np.array(ex_data['query_vector'].tolist()))

# 예측값 생성
ex_prediction = model.predict(ex_tensor)
ex_binary_predictions = (ex_prediction > threshold).astype(int)

# 검증
ex_pred = []
for b in ex_binary_predictions:
    ex_pred.append(b[0])
ex_pred_sr = pd.Series(ex_pred).reset_index(drop=True)
ex_y_sr = pd.Series(ex_data['Label']).reset_index(drop=True)

(ex_pred_sr == ex_y_sr).value_counts(dropna=False, normalize=True)
```
True: 0.985... / False: 0.014...

```
from sklearn.metrics import accuracy_score, precision_score, recall_score, f1_score

# 정확도 계산
accuracy = accuracy_score(ex_y_sr, ex_pred_sr)
print(f"Accuracy: {accuracy:.4f}")

# 정밀도 계산
precision = precision_score(ex_y_sr, ex_pred_sr)
print(f"Precision: {precision:.4f}")

# 재현율 계산
recall = recall_score(ex_y_sr, ex_pred_sr)
print(f"Recall: {recall:.4f}")

# F1 Score 계산
f1 = f1_score(ex_y_sr, ex_pred_sr)
print(f"F1 Score: {f1:.4f}")
```

Accuracy: 0.9855  
Precision: 0.9709  
Recall: 0.9990  
F1 Score: 0.9848  

### conclusion
#### 모델 성능
Recall이 너무 높아서 위양성이 높을 가능성이 있지만, 보안은 탐지 목적인 만큼 위양성이 높은 편이 낫기 때문에 큰 문제가 되지 않을 것으로 생각된다.  
Accuracy 및 F1 Score이 충분히 높기에 자체 보안 엔진 개발이 어려운 중소기업에서 SQL injection 방지를 위해 사용하기엔 적합하다고 판단된다.  

