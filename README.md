# Dacon_Behavioral_Data_Analysis
## 결과 : AUC=0.754 (리더보드 6위 수준)
* 데이터 다운로드 출처 : https://newfront.dacon.io/competitions/official/235583/data/
* 데이터 다운로드 출처 : https://github.com/Blizzard/s2client-proto#downloads


## Data Description
* 대회에서 제공하는 데이터는 게임 플레이어의 행동 정보를 담고 있습니다. 이 데이터를 사용하여 게임에서 승리하는 선수를 예측합니다. 데이터는 5만여 개의 경기 리플레이 데이터로 이루어져 있으며, 각 리플레이 데이터는 총 경기 시간의 일부에 대한 인게임 정보를 포함합니다. 

### train.csv / test.csv

##### game_id : 경기 구분 기호
##### winner : player 1의 승리 확률
##### time : 경기 시간, 마침표(.)로 분과 초가 구분됩니다. ex) 2.24 = 2분 24초

##### player : 선수

##### ##1) 0: player 0

##### ##2) 1: player 1

##### species : 종족

##### ##1) T: 테란

##### ##2) P: 프로토스

##### ##3) Z: 저그

##### event : 행동 종류

##### event_contents : 행동 상세

##### ##1) Ability : 생산, 공격 등 선수의 주요 행동

##### ##2) AddToControlGroup : 부대에 추가

##### ##3) Camera : 시점 선택

##### ##4) ControlGroup : 부대 행동

##### ##5) GetControlGroup : 부대 불러오기

##### ##6) Right Click : 마우스 우클릭

##### ##7) Selection : 객체 선택

##### ##8) SetControlGroup : 부대 지정
 
### sample_submission.csv

##### game_id : 경기 구분 기호
##### winner : player 1의 승리 확률

## Data Processing Plan
##### 데이터 특성상 winner (반응변수 0 or 1)를 예측하기 위해 독립변수(X)들을 직접 생성해내야 한다.

##### 따라서, Part를 나눠 Ch0 ~ Ch4 까지 Data PreProcessing 및 Features 생성을 하도록 하고,

##### Ch5에서 Feature Selection과 Prediction Modeling 순서로 진행하려 한다.

### 0. 데이터의 용량을 줄이기
* Train : 67091776 rows, 7 columns, 약 3.5GB 메모리 차지하는 데이터
* Test  : 28714849 rows, 6 columns, 약 1.3GB 메모리 차지하는 데이터
* 데이터의 type 변경을 통해 용량을 줄인 후, pkl 형태로 데이터를 새로 저장한다.
* Train data : 3.5GB to 2.3GB (66% of initial size)
* Test  data : 1.3GB to 0.9GB (66% of initial size)
* 본래 사용할 메모리보다 약 34% 여유공간을 확보할 수 있다.

### 1. Generating Features from each 2 time period
