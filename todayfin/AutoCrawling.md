# 파이썬 크롤링 코드 자동화하기

## 계획
1. 파이썬 코드 실행하는 도커 이미지 만들기
2. 도커 이미지를 위한 Dockerfile, requirements.txt 작성
3. 한번 실행시켜 이미지가 잘 되는지 확인
4. Cron 작업을 통해 하루에 한 번 실행시키기

## 하루에 한번 실행 시켜야할 파이썬 코드 (main.py) (AI쪽에서 받은 코드)
```
# -*- coding: utf-8 -*-
'''
pandas version: 2.1.4
다른 버전은 경고가 뜨긴 하는데 실행엔 문제 없습니다.
'''
import pandas as pd

import requests
from newspaper import Article

from pymongo import MongoClient
from bson.objectid import ObjectId

# news api로 기사 정보 수집
api_key = #뉴스 api key 넣기
cats = ['business','entertainment','general','health','science','sports','technology']
temp = {}
for cat in cats:
    url = f'https://newsapi.org/v2/top-headlines?pageSize=100&country=us&apiKey={api_key}&category={cat}'
    response = requests.get(url)
    data = response.json()
    headline_df = pd.DataFrame(data['articles'])
    headline_df['category'] = cat
    temp[cat] = headline_df

headline_full = pd.concat(temp)

# source 변수의 딕셔너리에서 name만 추출하여 다시 source에 저장
headline_full['source'] = headline_full['source'].copy().apply(lambda x: x['name'])

# 인터넷 기사 원문 링크 리스트
idx = headline_full['description'].isna()
headline_clean = headline_full[-idx].reset_index(drop=True)
headline_clean['article'] = ''

# 원문 출력
for i, url in enumerate(headline_clean['url']):
    try:
        article = Article(url = url, follow_meta_refresh = True, verbose = True )
        article.download()
        article.parse()
        headline_clean['article'][i] = article.text
    except :
        pass

temp1 = headline_clean[['title', 'category', 'article', 'urlToImage', 'publishedAt', 'source', 'author']]

temp2 = temp1[temp1['article'] != ''].reset_index(drop=True)

"""# DB 연결"""

# DB 연결 - Cluster : NewsAPi-cluster, Database : newsDB, Collection : news
uri = # MongoDB Atlas Uri 넣기

client = MongoClient(uri)

# 데이터베이스 선택
db = client['newsDB']

# 컬렉션 선택
news_collection = db['news']

# f'{{"title": {temp2["title"][i]}, "article": {temp2["article"][i]}, "urlToImage": {temp2["urlToImage"][i]}, "publishedAt": {temp2["publishedAt"][i]}, "source": {temp2["source"][i]}, "author": {temp2["author"][i]}}}'
query = []
for i in range(len(temp2)):
    temp_dic = {}
    temp_dic['title'] = temp2["title"][i]
    temp_dic['article'] = temp2["article"][i]
    temp_dic['urlToImage'] = temp2["urlToImage"][i]
    temp_dic['publishedAt'] = temp2["publishedAt"][i]
    temp_dic['source'] = temp2["source"][i]
    temp_dic['author'] = temp2["author"][i]
    # 나중에 저장할 key(번역, 임베딩) 미리 생성
    # DB에서 해당 key가 공백인 데이터를 불러서 처리 후 다시 저장하는 용도로 사용할 예정
    temp_dic['trans'] = ''
    temp_dic['embedding'] = ''
    query.append(temp_dic)


# 삽입
result = news_collection.insert_many(query)
print('done!')
```

## 위의 파이썬 코드를 위한 Dockerfile, requirement.txt 작성
- Dockerfile
    ```
    # 공식 Python 이미지를 기반으로 설정
    FROM python:3.9

    # 작업 디렉토리 설정
    WORKDIR /app

    # 현재 디렉토리의 파일들을 컨테이너의 작업 디렉토리로 복사
    COPY . /app

    # requirements.txt에 명시된 필수 패키지들을 설치
    RUN pip install --no-cache-dir -r requirements.txt

    # 컨테이너에서 외부로 연결할 포트 번호 설정 (필요한 경우)
    EXPOSE 80

    # 컨테이너 실행 시 자동으로 Python 스크립트 실행
    CMD ["python", "main.py"]

    ```
- requirement.txt
    ```
    pandas==2.1.4
    requests
    newspaper3k
    pymongo
    lxml[html_clean]
    ```
- 도커 이미지 만들기
    ```
    docker build -t news-crawling .
    ```

## 생성된 도커이미지 실행시켜 디비에 잘 삽입되는지 확인

1. mongosh로 해당 mongoDB에 연결하기
    ```
    mongosh "uri"
    ```
2. 해당 DB, Collection 접근
    ```
    show dbs;
    use newsDB;
    show collections;
    db.news.countDocuments();  #현재 콜렉션에 있는 데이터 개수 출력
    ```
3. 도커 이미지 실행
    ```
    docker run --rm news-crawling
    ```
4. 실행되는데 조금 걸림. 그리고 몽고디비 쉘에서 데이터 개수와 최근 삽입된 날짜 확인
    ```
    db.news.countDocuments();

    db.news.find().sort({publishedAt: -1}).limit(5) -> 최근에 추가된 것 확인
    ```
    - 아까 실행했을때 169개이고 도커 이미지 실행 시킨 후에는 277개 이므로 성공
    - `db.news.find().sort({publishedAt: -1}).limit(5)` 명령어를 통해 8월 5일 날짜로 잘 삽입된 것 확인


## cron을 통해 하루에 한 번씩 실행 시키기
- crontab 설정
    ```
    crontab -e
    ```
- 처음 실행시킨 시간이 오후 5시 이므로 매일 오후 5시에 실행되도록 Cron작업 추가
    ```
    0 17 * * * /usr/local/bin/docker run --rm news-crawling
    ```
- 잘 추가 되었는지 확인
    ```
    crontab -l
    ```


<br>

> 이제 8월 7일 오후 5시 이후에 `db.news.countDocuments();`했을때 277개 이상이면 성공!!