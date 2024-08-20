# 크롤링 자동화, 임베딩 자동화 코드 도커파일로 만들어서, 서버에서 특정 시간에 실행시키기


## 문제 상황
- 기존에 만들었던 크롤링 코드가 수정되고 임베딩하는 코드까지 추가되었다.
- 그리고 현재는 로컬에서 돌렸지만, 특정 시간마다 로컬의 맥북을 켜놔야 하는 이슈로 구글 클라우드 서버에 crontab으로 실행시키려고 함

## 해결 방법
1. 구글 클라우드 VM 인스턴스 만들기
2. 맥북에서 도커 이미지 만들기
3. 도커 허브에 push하기
4. 구글 클라우드 서버에서 이미지 받아와 실행시키기
5. 구글 클라우드 서버의 crontab 설정하기


## 1. 구글 클라우드 VM 인스턴스 만들기
- 기존에 만들어져 있던 구글 클라우드 인스턴스를 이용했다.
- 사양은 vCPU 2개, 1코어, 메모리 4GB 이다.

## 2. 맥북에서 도커 이미지 만들기

### 뉴스 크롤링 코드 도커 이미지 만들기 - code
    - news_crawl_alp.py
    ```
    # -*- coding: utf-8 -*-
    # 20240813 16:40 첫 DB 저장

    import pandas as pd

    import requests
    from newspaper import Article
    from datetime import datetime, timedelta

    from pymongo import MongoClient
    from bson.objectid import ObjectId

    # from dotenv import load_dotenv
    import os

    # api key load
    # load_dotenv()
    apikey = ""
    mongodb = ""


    # cats_full = ['blockchain', 'earnings', 'ipo', 'mergers_and_acquisitions',
    #         'finacial_markets', 'economy_fiscal', 'economy_monetary', 'economy_macro',
    #         'energy_transportation', 'finace', 'life_sciences', 'manufacturing',
    #         'real_estate', 'retail_wholesale', 'technology']

    # 토픽이 너무 디테일해서 일단 간단한 토픽만 수집
    cats_sub = ['finace', 'life_sciences', 'manufacturing',
            'real_estate', 'retail_wholesale', 'technology']

    # 1일 간격으로 수집
    today = datetime.now().strftime('%Y%m%dT%H%M')
    yesterday = (datetime.now() - timedelta(days=1)).strftime('%Y%m%dT%H%M')
    today4log = datetime.today().strftime('%Y%m%d') # 로그용 날짜 변수

    temp = {}
    for cat in cats_sub:
        url = f'https://www.alphavantage.co/query?function=NEWS_SENTIMENT&&apikey={apikey}&time_from={yesterday}&time_to={today}&topics={cat}'
        response = requests.get(url)
        data = response.json()
        df = pd.DataFrame(data['feed'])
        df['category'] = cat
        temp[cat] = df

    news_df = pd.concat(temp)
    news_df.reset_index(drop=True, inplace=True)
    news_df = news_df.rename(columns = {'time_published': 'publishedAt', 'authors': 'author', 'banner_image': 'urlToImage'})
    # author 자료형 변경 list -> str
    news_df['author'] = news_df['author'].apply(lambda x: ','.join(x))
    print(f'#################### {today4log} Alpha Vantage API 데이터 조회 완료 ####################')

    # 원문 추출
    # 300개 기준 3분정도
    article_list = []
    for i, url in enumerate(news_df['url']):
        try:
            article = Article(url = url, follow_meta_refresh = True, verbose = True )
            article.download()
            article.parse()
            article_list.append(article.text)
        except :
            article_list.append('')
            pass
    news_art = pd.concat([news_df, pd.Series(article_list, name = 'article')], axis = 1)
    news_fin = news_art[news_art['article'] != ''].reset_index(drop=True)
    print(f'#################### {today4log} 원문 추출 완료 ####################')

    # DB 연결 - Cluster : NewsAPi-cluster, Database : newsDB, Collection : news
    client = MongoClient(mongodb)

    # 데이터베이스 선택
    db = client['newsDB']

    # 컬렉션 선택(백업 컬렉션에 저장)
    news_collection_backup = db['news_backup']

    query = []
    for i in range(len(news_fin)):
        temp_dic = {}
        temp_dic['title'] = news_fin["title"][i]
        temp_dic['title_trans'] = ''
        temp_dic['category'] = news_fin["category"][i]
        temp_dic['article'] = news_fin["article"][i]
        temp_dic['article_trans'] = ''
        temp_dic['url'] = news_fin["url"][i]
        temp_dic['urlToImage'] = news_fin["urlToImage"][i]
        temp_dic['publishedAt'] = news_fin["publishedAt"][i]
        temp_dic['source'] = news_fin["source"][i]
        temp_dic['author'] = news_fin["author"][i]
        temp_dic['embedding'] = ''
        query.append(temp_dic)

    # 삽입
    result = news_collection_backup.insert_many(query)
    print(f'#################### {today4log} news 정보 DB 저장 완료 ####################')
    ```
    
- Dockerfile
    ```
    # 공식 Python 이미지를 기반으로 설정
    FROM python:3.9

    # 작업 디렉토리 설정
    WORKDIR /app

    # 현재 디렉토리의 파일들을 컨테이너의 작업 디렉토리로 복사
    COPY . /app

    # pip를 업그레이드하고 wheel을 설치
    RUN pip install --no-cache-dir --upgrade pip && pip install wheel

    # requirements.txt에 명시된 필수 패키지들을 설치
    RUN pip install --no-cache-dir -r requirements.txt

    # 컨테이너에서 외부로 연결할 포트 번호 설정 (필요한 경우)
    EXPOSE 80

    # 컨테이너 실행 시 자동으로 Python 스크립트 실행
    CMD ["python", "main.py"]
    ```

- requirements.txt
    ```
    pandas==2.1.4
    requests
    newspaper3k
    pymongo
    lxml[html_clean]
    ```

### 뉴스 임베딩 번역 코드 도커 이미지 만들기 - code
- news_embed_translate.py
    ```
    # -*- coding: utf-8 -*-

    # 모델 다운로드
    # pip install gdown
    # gdown "https://drive.google.com/uc?id=1Av37IVBQAAntSe1X3MOAl5gvowQzd2_j"
    # 파일 크기 1.65G

    # from dotenv import load_dotenv
    import os
    from datetime import datetime
    from pymongo import MongoClient
    from bson.objectid import ObjectId

    import pandas as pd
    import numpy as np

    import gensim
    import nltk
    from nltk.corpus import stopwords
    from nltk.tokenize import word_tokenize
    from googletrans import Translator

    # 한 번만 실행해도 됩니다.
    nltk.download('stopwords')
    nltk.download('punkt')
    nltk.download('wordnet')

    today4log = datetime.today().strftime('%Y%m%d') # 로그용 날짜 변수

    # DB 연결 - Cluster : NewsAPi-cluster, Database : newsDB, Collection : news
    # load_dotenv()
    mongodb = ""
    client = MongoClient(mongodb)

    # 데이터베이스 및 컬렉션 선택
    db = client['newsDB']
    news_collection_backup = db['news_backup']

    embed_translate = pd.DataFrame(list(news_collection_backup.find({'embedding': ''})))

    print(f'임베딩 생성이 필요한 데이터: {len(embed_translate)}')
    print(f'#################### {today4log} 데이터 조회 완료 ####################')

    # title 전처리
    preprocessed_title = []
    stop_words = set(stopwords.words('english'))

    for i in embed_translate['title']:
        tokenized_title = word_tokenize(i) # 토크나이저 변경 가능
        result = []
        for word in tokenized_title: # 모든 단어 소문자화, 생략 가능
            word = word.lower()

            if word not in stop_words: # 불용어 제거
                if len(word) > 2: # 단어 길이가 2이하인 단어 삭제
                    result.append(word)
        preprocessed_title.append(result)
    print(f'#################### {today4log} 데이터 전처리 완료 ####################')

    # 구글의 사전 학습 Word2Vec 모델 사용
    # 모델 로드하면 RAM 사용량 5.7GB까지 상승, 모델 로드 약 1~2분
    word2vec_model = gensim.models.KeyedVectors.load_word2vec_format('GoogleNews-vectors-negative300.bin.gz', binary=True)
    print(f'#################### {today4log} 모델 로드 완료 ####################')

    # 임베딩 함수
    def get_document_vectors(document_list):
        document_embedding_list = []
        vec_0_count = 0

        # 각 문서에 대해서
        for line in document_list:
            doc2vec = None
            count = 0

            for word in line:
                if word in word2vec_model.key_to_index:
                    count += 1
                    # 해당 문서에 있는 모든 단어들의 벡터값을 더한다.
                    if doc2vec is None:
                        doc2vec = word2vec_model[word]
                    else:
                        doc2vec = doc2vec + word2vec_model[word]

            if doc2vec is not None and count > 0:  # 단어 벡터가 존재하고, count가 0보다 큰 경우
                # 단어 벡터를 모두 더한 벡터의 값을 문서 길이로 나눠준다.
                doc2vec = doc2vec / count
                document_embedding_list.append(doc2vec)
            else:
                # 빈 문서나 단어가 없는 문서의 경우, 0 벡터를 추가 (차원 일치를 위해)
                vec_0_count += 1
                zero_vector = np.zeros(word2vec_model.vector_size)
                document_embedding_list.append(zero_vector)
        print(f'빈 문서나 단어가 없는 문서의 개수: {vec_0_count}')
        # 각 문서에 대한 문서 벡터 리스트를 리턴
        return document_embedding_list

    document_embedding_list = get_document_vectors(preprocessed_title)
    embed_translate['embedding'] = document_embedding_list
    print('문서 벡터의 수 :',len(document_embedding_list))
    print(f'#################### {today4log} 데이터 임베딩 완료 ####################')

    for idx in embed_translate._id:
        val = embed_translate[embed_translate['_id'] == idx]['embedding'].to_list()[0].tolist()
        news_collection_backup.update_one({'_id': idx}, {'$set': {'embedding': val}})
    print(f'#################### {today4log} 임베딩 DB 저장 완료 ####################')


    news_collection = db['news']
    news_collection_need2trans = db['news_need2trans']

    # googletrans 번역기 설정
    translator = Translator()

    def translate_article(article):
        try:
            # 번역
            title_trans = translator.translate(article['title'], src='en', dest='ko').text
            article_trans = translator.translate(article['article'], src='en', dest='ko').text
            
            # backup에 업데이트
            article['title_trans'] = title_trans
            article['article_trans'] = article_trans
            news_collection_backup.update_one(
                {'_id': article['_id']},
                {'$set': {'title_trans': title_trans, 'article_trans': article_trans}}
            )

            # news에 저장
            news_collection.insert_one(article)

            # print(f"Article with ID {article['_id']} translated and updated successfully.")
        except Exception as e:
            print(f"Error translating article with ID {article['_id']}: {e}")
            
            # need2trans에 저장
            news_collection_need2trans.insert_one(article)

            # backup에서 삭제
            news_collection_backup.delete_one({'_id': article['_id']})
            

    def translate_main():
        # 공백인 데이터 가져오기
        articles = news_collection_backup.find({'title_trans': '', 'article_trans': ''})

        for article in articles:
            translate_article(article)

    translate_main()
    print(f'#################### {today4log} 번역 DB 저장 완료 ####################')
    ```

- Dockerfile
    ```
    # 공식 Python 이미지를 기반으로 설정
    FROM python:3.9

    # 작업 디렉토리 설정
    WORKDIR /app

    # 현재 디렉토리의 파일들을 컨테이너의 작업 디렉토리로 복사
    COPY . /app

    # pip를 업그레이드하고 wheel을 설치
    RUN pip install --no-cache-dir --upgrade pip && pip install wheel

    # requirements.txt에 명시된 필수 패키지들을 설치
    RUN pip install --no-cache-dir -r requirements.txt

    # NLTK 데이터 다운로드
    RUN python -c "import nltk; nltk.download('punkt'); nltk.download('stopwords'); nltk.download('wordnet'); nltk.download('punkt_tab'); nltk.download('tokenizers/punkt/english.pickle')"

    # Word2Vec 모델 다운로드
    RUN pip install --no-cache-dir gdown
    RUN gdown "https://drive.google.com/uc?id=1Av37IVBQAAntSe1X3MOAl5gvowQzd2_j" -O GoogleNews-vectors-negative300.bin.gz

    # 컨테이너에서 외부로 연결할 포트 번호 설정 (필요한 경우)
    EXPOSE 80

    # 컨테이너 실행 시 자동으로 Python 스크립트 실행
    CMD ["python", "main.py"]
    ```

- requirements.txt
    ```
    pandas
    numpy
    pymongo
    gensim
    nltk
    googletrans==3.1.0a0
    requests
    newspaper3k
    lxml[html_clean]
    gdown
    ```

### 도커 이미지 만들기
```
$ docker build -t news-crawling .
$ docker build -t news-embed-translate .
```


## 3. 도커 허브에 push하기
- 먼저 도커 허브에 push 하기위에 tag만들어주기
    ```
    $ docker tag news-crawling jonum12312/news-crawling:latest
    $ docker tag news-embed-translate jonum12312/news-embed-translate:latest
    ```
- 맥북은 arm 아키텍처를 사용하지만 구글 클라우드 서버는 amd 아키텍처를 사용하므로 buildx를 사용하여 multi-architecture로 build하여 push해야한다.
    ```
    # buildx 빌더 생성 및 사용
    $ docker buildx create --name mybuilder --use

    # multi-architecture 빌드 및 푸시
    $ docker buildx build --platform linux/amd64,linux/arm64 -t jonum12312/news-crawling:latest --push .
    $ docker buildx build --platform linux/amd64,linux/arm64 -t jonum12312/news_embedding_translation:latest --push .
    ```


## 4. 구글 클라우드 서버에서 이미지 받아와 실행시키기
- docker images pull
    ```
    $ docker pull jonum12312/news-crawling:latest
    $ docker pull jonum12312/news-embed-translate:latest
    ```
- pull 받은 이미지 잘 되나 실행해보기
    ```
    $ docker run --rm jonum12312/news-crawling:latest
    $ docker run --rm jonum12312/news-embed-translate:latest
    ```

> 참고로 vCPU 2개, 1코어, 4GM 메모리를 가진 VM으로 실행했지만, 실행 도중 에러가 떠서 vCPU 4개, 2코어, 16GM 메모리로 교체하여 돌렸는데 잘 실행이 되었다.

## 5. 구글 클라우드 서버의 crontab 설정하기
- 먼저 서버의 시간을 확인해야한다.
    ```
    $ date
    Mon Aug 19 08:42:29 UTC 2024
    ```
- 한국 시간으로는 오후 5시쯤 코드를 실행했지만, 서버 시간을 기준으로 설정을 해야 하기 때문에 매일 오전 08시 정도에 코드를 실행시키겠다.
- crontab Log 파일 만들기
    ```
    root@todayfin-alphavantage-api-server-3:/home/sindongseong# pwd
    /home/sindongseong
    root@todayfin-alphavantage-api-server-3:/home/sindongseong# mkdir -p cronLog/news-crawling.log
    root@todayfin-alphavantage-api-server-3:/home/sindongseong# mkdir -p cronLog/news-embed-translate.log
    ```
- crontab 설정하기
    ```
    root@todayfin-alphavantage-api-server-3:/home/sindongseong# crontab -l
    0 7 * * * /usr/bin/docker run --rm jonum12312/news-crawling >> /home/sindongseong/cronLog/news-crawling.log 2>&1
    30 7 * * * /usr/bin/docker run --rm jonum12312/news-embed-translate >> /home/sindongseong/cronLog/news-embed-translate.log 2>&1
    ```
    - UTC 오전 7시는 한국시간 오후 4시 이므로 이렇게 실행
    - `/usr/bin/docker ps` 를 통해 도커가 작동하는 것 확인
    - docker 명령어로 실행한 표준에러와 표준 출력을 동시에 log파일에 저장하는 코드이다.