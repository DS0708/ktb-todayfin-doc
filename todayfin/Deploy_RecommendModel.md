# 추천 모델 배포하기

## 추천 모델 작동 코드
```
# API 통신 코드

# uvicorn main:app --reload 로 코드 실행
from typing import List

from fastapi import FastAPI
from pydantic import BaseModel
from pymongo import MongoClient
from bson.objectid import ObjectId

import numpy as np
import pandas as pd
'''
input
유저 로그 데이터(_id) - List(str)

output
{recommend: List[str]}
'''

app = FastAPI()

# request body 데이터 설정
class UserLog(BaseModel):
    news_id: List[str] # 유저의 뉴스 로그

# DB load
MONGODB = ""

client = MongoClient(MONGODB)
db = client['newsDB']
news_collection = db['news']

# API
@app.post("/recommend/")
async def top5_recommendation(user_log: UserLog):
    '''
    1. db에서 user_log에 속하는 임베딩 벡터 조회
    2. 합산
    3. 합산된 벡터와 가장 유사한 기사 5개 리턴 recommend_id

    + 최신 기사 고려
    + 카테고리 고려
    '''
    news_id = [ObjectId(x) for x in user_log.news_id]
    read = pd.DataFrame(news_collection.find({'_id': {'$in': news_id}}))
    read['embedding'] = read['embedding'].apply(np.array) # 벡터 연산 가능하게 넘파이로 저장
    
    user_vec = read['embedding'].sum()

    pipeline = [
        {
            '$vectorSearch': {
                'index': 'recommend_vector_index', 
                'exact': False, # ANN, True시 ENN
                'path': 'embedding', 
                'queryVector': user_vec.tolist(), 
                'numCandidates': 150,
                '_id': {'$nin': news_id}, # 이미 조회한 기사 제외 
                'limit': 5
                }
        }, {
            '$project': {
                '_id': 1, 
                # 'title': 1, 
                # 'title_trans':1,
                # 'category': 1, 
                'score': {
                    '$meta': 'vectorSearchScore'
                    }
                }
            }
        ]
    result = [str(x['_id']) for x in news_collection.aggregate(pipeline)]
    score = [x['score'] for x in news_collection.aggregate(pipeline)]
    print(score)
    return {'recommed': result}




'''
covy
{
  "news_id": ["66bc620f9bf7b12d09a57c46", "66bc620f9bf7b12d09a57bd3", "66bb0d02eefbca8ec2b20cf8", "66bf0070a7d40a0ec0cf8f9a", "66bc620f9bf7b12d09a57c7b"]
}
'''
```

- 요청 url : `http://ip:port/recommend/`
- Body (JSON) - 예시 (newsDB의 '_id' 값 5개)
    ```
    {
    "news_id": ["66bc620f9bf7b12d09a57c46", "66bc620f9bf7b12d09a57bd3", "66bb0d02eefbca8ec2b20cf8", "66bf0070a7d40a0ec0cf8f9a", "66bc620f9bf7b12d09a57c7b"]
    }
    ```

## 도커 파일만들기
- Dockerfile
    ```
    # Dockerfile
    FROM python:3.9

    WORKDIR /app

    COPY requirements.txt /app/
    RUN pip install --no-cache-dir -r requirements.txt

    COPY . /app/

    EXPOSE 8000

    CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]

    ```
    - `--no-cache-dir` 옵션을 사용하여 Docker 이미지 크기를 줄이기 위해 캐시를 저장하지 않는다.
    - 8000번 포트를 열어 어떤 ip에서도 접근할 수 있도록 실행 시켰다.
- requirement.txt
    ```
    fastapi
    uvicorn
    pymongo
    pydantic
    numpy
    pandas
    ```
    - 이 코드가 실행되는데 필요한 파이썬 라이브러리들을 명시하였다.

## 도커 이미지 만들어 localhost로 테스트하기
- 도커 이미지 만들기
    ```
    $ docker build -t recommend_model .
    ```
- local에서 실행
    ```
    $ docker run -p 8000:8000 recommend_model
    ```
- `Postman` 사용해서 요청 보내보기
    - Method : `POST`
    - url : "http://localhost:8000/recommend/"
    - body(raw에 JSON형식)
        ```
        {
        "news_id": ["66bc620f9bf7b12d09a57c46", "66bc620f9bf7b12d09a57bd3", "66bb0d02eefbca8ec2b20cf8", "66bf0070a7d40a0ec0cf8f9a", "66bc620f9bf7b12d09a57c7b"]
        }
        ```
    - response body
        ```
        {
            "recommed": [
                "66c051ff04387eaa3f3321ba",
                "66c051ff04387eaa3f3321a2",
                "66bc620f9bf7b12d09a57c46",
                "66bb0d02eefbca8ec2b20cfa",
                "66c051ff04387eaa3f332135"
            ]
        }
        ```


## 서버에 배포하기
- 로컬에서 정상적으로 동작하는 것을 확인했으니 이제 서버에 배포할 차례이다.
- 내 로컬 환경은 mac을 사용하여 아키텍처를 arm을 사용하고 서버는 amd를 사용하기 때문에 buildx로 멀티 아키텍처 배포를 할 것이다.
    ```
    $ docker buildx build --platform linux/amd64,linux/arm64 -t jonum12312/recommend_model:latest --push .
    ```
    - buildx를 먼저 만들어주고 `docker login`을 통해 로그인 되어 있어야 한다.
    - buildx를 이용해 amd와 arm에서 모두 사용가능하도록 멀티 아키텍처 배포를 하였다.
    - 나의 도커 로그인의 id가 jonum12312 이다.
    - `--push` 옵션은 로컬에 이미지를 저장하지 않고 바로 도커 허브에 푸쉬하는 것이다
    - 마지막에 . 은 도커의 빌드 컨텍스트를 지정하는 것이며 해당 Dockerfile과 관련 파일들이 존재하는 곳에서 실행해야한다.
- 이제 서버에서 이미지를 다운 받아 실행시키면 된다.
- 그 전에 포트 부터 열어줘야 한다. GCP VM의 8000번 포트를 열어야 한다.
    - 해당 VM이 속한 VPC 설정에 간다.
    - 방화벽을 누른다.
    - 방화벽 규칙을 TCP로 8000번 포트를 0.0.0.0/0(모든 ip)에서 접근가능하도록 추가한다.
- 이제 서버에서 이미지를 다운 받아 배포한다.
    ```
    $ docker pull jonum12312/recommend_model

    $ docker run -p 8000:8000 -d jonum12312/recommend_model
    ```
- `http://serverIP:port/recommend/` 로 POST 요청을 보내 동일한 결과가 출력되는 것을 확인한다.

