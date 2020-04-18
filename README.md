# Dacon_Behavioral_Data_Analysis
## 결과 : AUC=0.754 (리더보드 6위 수준)
* 데이터 다운로드 출처 : https://newfront.dacon.io/competitions/official/235583/data/
* 데이터 다운로드 출처 : https://github.com/Blizzard/s2client-proto#downloads


## Data Description
* 대회에서 제공하는 데이터는 게임 플레이어의 행동 정보를 담고 있습니다. 이 데이터를 사용하여 게임에서 승리하는 선수를 예측합니다. 데이터는 5만여 개의 경기 리플레이 데이터로 이루어져 있으며, 각 리플레이 데이터는 총 경기 시간의 일부에 대한 인게임 정보를 포함합니다. 

### train.csv / test.csv

* game_id : 경기 구분 기호
* winner : player 1의 승리 확률
* time : 경기 시간, 마침표(.)로 분과 초가 구분됩니다. ex) 2.24 = 2분 24초

* player : 선수

* ##1) 0: player 0

* ##2) 1: player 1

* species : 종족

* ##1) T: 테란

* ##2) P: 프로토스

* ##3) Z: 저그

* event : 행동 종류

* event_contents : 행동 상세

* ##1) Ability : 생산, 공격 등 선수의 주요 행동

* ##2) AddToControlGroup : 부대에 추가

* ##3) Camera : 시점 선택

* ##4) ControlGroup : 부대 행동

* ##5) GetControlGroup : 부대 불러오기

* ##6) Right Click : 마우스 우클릭

* ##7) Selection : 객체 선택

* ##8) SetControlGroup : 부대 지정
 
### sample_submission.csv

* game_id : 경기 구분 기호
* winner : player 1의 승리 확률

## Data Processing Plan
##### 데이터 특성상 winner (반응변수 0 or 1)를 예측하기 위해 독립변수(X)들을 직접 생성해내야 한다.

##### 따라서, Part를 나눠 Ch0 ~ Ch4 : "Data PreProcessing 및 Features 생성"을 하도록 하고,

##### Ch5 : "Feature Selection과 Prediction Modeling" 순서로 진행하려 한다.

### 0. 데이터의 용량을 줄이기
* Train : 67091776 rows, 7 columns, 약 3.5GB 메모리 차지하는 데이터
* Test   : 28714849 rows, 6 columns, 약 1.3GB 메모리 차지하는 데이터
* 데이터의 type 변경을 통해 용량을 줄인 후, pkl 형태로 데이터를 새로 저장한다.
* Train data : 3.5GB to 2.3GB (66% of initial size)
* Test  data  : 1.3GB to 0.9GB (66% of initial size)
* 본래 사용할 메모리보다 약 34% 여유공간을 확보할 수 있다.

### 1. Generating Features for each 2 time period
#### 데이터가 워낙 방대하다 보니, 38872개의 게임으로부터 (train 기준) 각각 키워드포함 횟수를 카운팅하는 것에 for문을 적용하기에는 데이터 처리속도가 매우 느리다는 것을 알 수 있었다.
#### 따라서, pandas의 groupby와 함께 value_counts 메소드를 통해 이를 해결하였으며 for문에 비해 약 50배 속도가 빠른 것을 확인하였다.

* 1번째, event 종류 ('Ability', 'AddToControlGroup', 'Camera', 'ControlGroup', 'GetControlGroup', 'Right Click', 'Selection', 'SetControlGroup') 들의 횟수를 게임과 Player 별로, 그리고 시간 2분 간격으로 카운팅하여 변수를 생성한다.
* ex1) 컬럼명: P0_Ability_bytime_0 = Player0의 0~2분까지 Ability event의 횟수
* ex2) 컬럼명: P1_GetControlGroup_bytime_8 = Player1의 8~10분까지 GetControlGroup event의 횟수

* 2번째, Starcraft2 게임 특성상 Map에 따라 게임의 승자에 영향을 줄 수 있다. Map이라는 Categorical Variable을 생성하도록 한다.
* Map 별로 플레이어의 시작 Camera위치(스타팅 화면좌표)에 차이가 있으므로 Clustering을 통해 Map의 종류를 나눌 수 있다.

* 3번째, event_contents의 텍스트 자체에서 "특정 키워드를 포함하는가"로 횟수를 카운팅하여 변수를 생성할 수 있다. 이 또한, 1번째와 마찬가지로 시간 2분 간격으로 나누어 그 횟수를 카운팅한다.
* 변수생성을 위해 포함 키워드를 정한 도메인은 다음과 같다.
* Ability - 일꾼생산개수, 공격력 업그레이드, 방어력 업그레이드, 인구수, 멀티건물개수, 가스건물개수, 일꾼공격횟수, 멀티공격횟수, 총공격횟수, 일점사횟수, 기타업그레이드횟수, 병력생산횟수, 방어건물갯수, Patrol횟수, 생산병력수치화(생산하는 데 소모되는 자원을 기준으로 가중치 적용)
* Selection - 생산건물선택횟수, Selection_Empty, 일꾼선택횟수,
* RightClick - 미네랄우클릭횟수, 멀티우클릭횟수, 가스우클릭횟수, 중립건물우클릭횟수

* 1번째와 3번째에서 각각 2분씩 간격을 두어 따로 카운팅하였던 0~2, 2~4, 4~6, 6~8, 8~10, 10~12 변수들을 apply & sum 을 통해 횟수의 누적합으로 바꾸어주었다. (원래의 각 구간별 카운팅 횟수를 그대로 쓰는 것보다 이렇게 누적합을 취한 것이 실제로 예측력이 좋았다.)
* (1) 0-2 -> 0-2 (그대로)
* (2) 2-4 -> 0-4 ( 0-2 + 2-4 )
* (3) 4-6 -> 0-6 ( 0-2 + 2-4 + 4-6 )
* (4) 6-8 -> 0-8 ( 0-2 + 2-4 + 4-6 + 6-8 )
* (5) 8-10 -> 0-10 ( 0-2 + 2-4 + 4-6 + 6-8 + 8-10 )
* (6) 10-12 -> 0-12 ( 0-2 + 2-4 + 4-6 + 6-8 + 8-10 + 10-12 )

### 2. Generating Features from event_contents in Ability for each 2 time period
####  Ch2.1 & Ch2.2 : Ability
* 먼저, 정규표현식을 활용해 소괄호, 대괄호 안의 숫자 혹은 문자를 모두 제거하였음. (소괄호 대괄호 안의 문자들은 식별자 or 위치좌표)
* CounterVectorizer을 통해 Game별로, 그리고 Player별로 event_contents의 unique 값들 개수를 카운팅 (시간 2분간격으로 나누어서 변수생성함)
#### Ch2.3 : Ability
* 정규표현식 활용없이, 본래 텍스트 그대로 CountVectorizer을 통해 Game별로, Player 별로 event_contents 카운팅.
* 정규표현식 활용 안했기 때문에, event_contents의 unique값들 종류가 셀 수 없이 많다.
* 이를 모두 CounterVectorizer를 통해 카운팅하게 되면 변수의 개수가 기하급수적으로 늘어나 차원의 저주 문제 발생 우려됨.
* 예측력의 감소가 미미한 선에서 min_df 옵션을 크게 하여 변수의 개수를 줄이기 위해 노력하였음.

### 3. Generating Features from event_contents in Selection for each 2 time period
####  Ch3.1 & Ch3.2 : Selection
* 먼저, 정규표현식을 활용해 소괄호, 대괄호 안의 숫자 혹은 문자를 모두 제거하였음. (소괄호 대괄호 안의 문자들은 식별자 or 위치좌표)
* CounterVectorizer을 통해 Game별로, 그리고 Player별로 event_contents의 unique 값들 개수를 카운팅 (시간 2분간격으로 나누어서 변수생성함)
#### Ch3.3 : Selection
* 정규표현식 활용없이, 본래 텍스트 그대로 CountVectorizer을 통해 Game별로, Player 별로 event_contents 카운팅.
* 정규표현식 활용 안했기 때문에, event_contents의 unique값들 종류가 셀 수 없이 많다.
* 이를 모두 CounterVectorizer를 통해 카운팅하게 되면 변수의 개수가 기하급수적으로 늘어나 차원의 저주 문제 발생 우려됨.
* 예측력의 감소가 미미한 선에서 min_df 옵션을 조정함으로써 변수의 개수를 줄이기 위해 노력하였음.

### 4. Generating Features from event_contents in RightClick for each 2 time period
####  Ch4.1 : RightClick
* 먼저, 정규표현식을 활용해 소괄호, 대괄호 안의 숫자 혹은 문자를 모두 제거하였음. (소괄호 대괄호 안의 문자들은 식별자 or 위치좌표)
* CounterVectorizer을 통해 Game별로, 그리고 Player별로 event_contents의 unique 값들 개수를 카운팅
#### Ch4.2 : RightClick
* 정규표현식 활용없이, 본래 텍스트 그대로 CountVectorizer을 통해 Game별로, Player 별로 event_contents 카운팅.
* 정규표현식 활용 안했기 때문에, event_contents의 unique값들 종류가 셀 수 없이 많다.
* 이를 모두 CounterVectorizer를 통해 카운팅하게 되면 변수의 개수가 기하급수적으로 늘어나 차원의 저주 문제 발생 우려됨.
* 예측력의 감소가 미미한 선에서 min_df 옵션을 조정함으로써 변수의 개수를 줄이기 위해 노력하였음.

### 5. Generating Difference Features & Feature Selection & Optuna Bayesian Optimization for LGBM
* Game내의 Winner를 예측하는 Modeling을 위해 Player1 - Player0 = diff 변수를 추가로 생성한다.
* 수많은 Features를 생성해냈지만, Model의 성능에 도움이 되는 feature들만 선별해내는 방법론이 필요함
* Feature Selection 방법 - 1. lightgbm 모델을 활용한 Permutation Importance 방법론 & 2. RandomForest Feature Importance 기준 변수선택
#### Optuna AutoML 라이브러리
* Optuna 모듈을 활용한 AutoML 적용을 통해 최적의 LightGBM Parameter를 찾는다.
* 베이지안 최적화를 통해 자동으로 최적의 모수를 찾아주는 라이브러리인데, 주의할 점은 자동 최적화라 하더라도 너무 넓은 범위에서 파라미터를 튜닝하려해서는 안된다.
* 베이지안 최적화에서 모수의 초기값은 설정한 seed에 따라 Random Searching을 기반으로 하고 있기 때문이다.
* AutoML을 쓰다가 느끼게 된 점은 최대한 narrow한 range에서 파라미터 튜닝을 따로 수차례 하는 것이 좋다는 점이다.
* 예를 들어, bagging_fraction과 같은 과적합 방지 parameter에 대한 튜닝을 할 때,
* 가능 범위인 0부터 1 사이 전체로 범위를 설정하고 100번 searching하는 것보다, 먼저 0.5에서 1로 설정한 뒤 50번 searching을 통한 best_score 도출, 그 후 두번째 차례로 범위를 0에서 0.5로 50번 searching한 best_score 도출 후 비교를 통해 최적의 모수를 찾는 것이 좋다는 것이다.
* 이 경우에 또 다른 장점은 0에서 0.5 범위와 0.5에서 1 범위 두 가지의 best_score가 같은 값이 나왔을 경우, 해당 모수의 특성을 생각하여 과적합이 덜하다고 판단되는 모수를 택할 수 있게 된다. (예를 들어, bagging_fraction = 0.8일 때와 0.3일 때 Score 값이 같게 나온다면, 당연히 0.3인 경우를 best_score로 판단하는 것이 과적합에 있어서 상대적으로 자유로울 수 있게 되기 때문이다.)
* 게다가, 또 하나의 Optuna 모듈의 장점은 임의로 설정한 Warm_Starts 횟수 이후에 Score들의 Median 값을 확인한 후, pruning을 하는 것이 가능하다는 점이다. 이를 통해 AutoML의 단점인 많은 시간 소모를 해결할 수 있다.(pruning을 사용하지 않았을 때보다 몇 배 더 많은 적합을 시도해볼 수 있게 된다.)

### 6. 결론
* 가장 높은 score를 취할 수 있었던 경우의 변수들의 형태는 다음과 같다.
##### 2분씩 구간 나누어 event_contents의 종류를 카운팅하여 생성한 변수들
##### 2분씩 구간 나누지 않고 정규표현식 활용하여 event_contents의 unique 값 CountVectorizer 변수
##### 2분씩 구간 나누지 않고 정규표현식 활용하지 않고 event_contents의 unique 값 CountVectorizer 변수
* 가장 높은 score를 취할 수 있었던 경우의 변수선택법은 다음과 같다.
##### Permutation Importance 방법론 적용 후, RandomForest 변수 중요도를 이용한 상위 변수들 선택
* 서로 다르게 튜닝된 LightGBM 모델 2개의 예측값을 평균내어 앙상블한 것이 더 높은 점수를 가져다 주었다.
