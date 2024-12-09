---
title: "League of legends Data Analysis project"
type: page
---

## League of legends(LoL) Data Analysis
project in Dankook Univ. Sports Psychology Laboratory

#### 사용 언어
Python

---

### 1. Stroop Test
#### 제작 목적
시간에 따른 인지 피로도 변화 측정 및 시각화

#### 제작 도구
- UI: PyQt
- Test: Pygame
- data: Pandas
- analysis: Numpy
- visualization: Seaborn

---

### 2. 전적 분석 사이트 크롤링 및 분석
#### 대상
[Fow.lol](Fow.lol)

#### 제작도구
- data: Pandas, Numpy
- crawlling: requests, json, tqdm
- parsing: BeatifulSoup (bs4)

#### 제작 방법
1. html 코드 분석 및 API 호출 구문 확인
2. time seed 값으로 페이지 추가 생성이 되는 것을 확인
3. request를 통해 fow.lol 사이트 API에 직접 timeseed를 바탕으로 데이터를 요청하여 json으로 읽어들임
4. bs4를 이용하여 parsing 및 플레이어별 데이터 분류
5. 10분 단위로 연속 8게임을 행한 게임 플레이에 대해 따로 저장
6. 데이터 분석 및 시각화 진행

---

### 3. LoL API 이용 플레이어 데이터 자동 수집 및 분석
#### 제작도구
- API.py(Dedicated module): requests, json, re, csv, time
- data collection: API, Pandas, tqdm
- data analysis & visualization: Pandas, numpy, seaborn

#### 제작 방법
1. 활용할 LoL API를 정리한 API.py 모듈 제작
   ``` python
   import requests as rq
   import json
   import re
   import csv
   import time

   REQ_TERM = 1.25  # API requset limit

   # tier, division 목록
   high_tier = ['CHALLENGER', 'GRANDMASTER', 'MASTER']
   low_tier = ['DIAMOND', 'EMERALD', 'PLATINUM', 'GOLD', 'SILVER', 'BRONZE']
   tiers = ['CHALLENGER', 'GRANDMASTER', 'MASTER', 'DIAMOND', 'EMERALD', 'PLATINUM', 'GOLD', 'SILVER', 'BRONZE']
   tiers_tft = tiers[3:]
   division = ['I', 'II', 'III', 'IV']


   def summonerData(tier, division, page, api_key):
     url = f"https://kr.api.riotgames.com/lol/league-exp/v4/entries/RANKED_SOLO_5x5/{tier}/{division}?page={page}"
     header = {"X-Riot-Token": api_key}
     data = rq.get(url, headers=header).json()
     result = []
     for temp in data:
       result.append(temp['summonerId'])

     time.sleep(REQ_TERM)

     return result

   def summonerTopuuid(summonerId, api_key):
     url = f"https://kr.api.riotgames.com/lol/summoner/v4/summoners/{summonerId}"
     header = {"X-Riot-Token": api_key}

     time.sleep(REQ_TERM)

     return rq.get(url, headers=header).json()['puuid']

   def puuidTogameName(puuid, api_key):
     url = f"https://asia.api.riotgames.com/riot/account/v1/accounts/by-puuid/{puuid}"
     header = {"X-Riot-Token": api_key}

     time.sleep(REQ_TERM)

     return rq.get(url, headers=header).json()

   def puuidTomatchId(puuid, api_key):
     url = f"https://asia.api.riotgames.com/lol/match/v5/matches/by-puuid/{puuid}/ids?type=ranked&start=0&count=100"
     header = {"X-Riot-Token": api_key}

     time.sleep(REQ_TERM)

     return rq.get(url, headers=header).json()

   def gameInfo(matchId, api_key):
     url = f"https://asia.api.riotgames.com/lol/match/v5/matches/{matchId}"
     header = {"X-Riot-Token": api_key}

     time.sleep(REQ_TERM)

     return rq.get(url, headers=header).json()

   def timeline(matchId, api_key):
     url = f"https://asia.api.riotgames.com/lol/match/v5/matches/{matchId}/timeline"
     header = {"X-Riot-Token": api_key}

     time.sleep(REQ_TERM)

     return rq.get(url, headers=header).json()

   def tft_summonerData(tier, division, page, api_key):
     url = f"https://kr.api.riotgames.com/tft/league/v1/entries/{tier}/{division}?queue=RANKED_TFT&page={page}"
     header = {"X-Riot-Token": api_key}
     response = rq.get(url, headers=header)
     result = []

     if response.status_code == 200:
       data = response.json()

       for temp in data:
         result.append(temp['puuid'])

       time.sleep(REQ_TERM)

     return result

   def tft_puuidTomatchId(puuid, api_key):
     url = f"https://asia.api.riotgames.com/tft/match/v1/matches/by-puuid/{puuid}/ids"
     header = {"X-Riot-Token": api_key}

     time.sleep(REQ_TERM)

     return rq.get(url, headers=header).json()

   def tft_gameInfo(matchId, api_key):
     url = f"https://asia.api.riotgames.com/tft/match/v1/matches/{matchId}"
     header = {"X-Riot-Token": api_key}

     time.sleep(REQ_TERM)

     return rq.get(url, headers=header).json()

   def gameNameTopuuid(gameName, tagLine, api_key):
     url = f"https://asia.api.riotgames.com/riot/account/v1/accounts/by-riot-id/{gameName}/{tagLine}"
     header = {"X-Riot-Token": api_key}

     time.sleep(REQ_TERM)

     return rq.get(url, headers=header).json()
   ```
2. API.py를 활용해 데이터 자동 수집 프로그램 제작
   - 1차 티어별 ID 및 tagline 수집
   - 2차 플레이 데이터 수집: 게임 시작, 종료 시간 timestamp(Unix timestamp)를 활용해 10분 간격으로 8게임 연속 플레이한 플레이 데이터만을 추출하여 저장
   - 티어별로 CSV파일로 변환 저장
4. 수집한 데이터를 Fow.lol 크롤링 데이터의 경향성을 기반으로 분석 및 시각화

---

연구 마무리 후 업데이트 예정입니다.
