# 7장 분산 시스템을 위한 유일 ID 생성기 설계

유일 ID 생성기는 auto_increment 속성이 설정된 관계형 데이터베이스의 기본 키를 쓰면 되지 않을까 생각할 수 있지만, 

분산 환경에서 데이터베이스 서버 한 대로는 그 요구를 감당하기 힘들고, 

여러 데이터베이스 서버를 쓰는 경우에는 지연시간을 낮추기가 힘들어서 auto_increment 속성을 사용하는 것이 제한된다.



### 1단계. 문제 이해 및 설계 범위 확정

시스템 설계 문제를 풀기 위해서는 적절한 질문이 필요하다.

이번 문제에 대해서 요구사항은 다음과 같다.

- ID는 유일해야 한다.
- ID는 숫자로만 구성되어야 한다.
- ID는 64비트로 표현될 수 있는 값이어야 한다.
- ID는 발급 날짜에 따라 정렬 가능해야 한다.
- 초당 10,000개의 ID를 만들 수 있어야 한다.

### 2단계. 개략적 설계안 제시 및 동의 구하기



##### 다중 마스터 복제

데이터베이스의 auto_increment 기능을 활용시키되, ID의 값을 증가시킬 때 1만큼 증가시키는 것이 아니라 데이터 서버의 갯수 k만큼 증가시킨다.

- 1번 서버 - 1, 4, 7, 10
- 2번 서버 - 2, 5, 8, 11
- 3번 서버 - 3, 6, 9, 12

###### 다만 이 방법은 다음과 같은 단점이 있다.

- 여러 데이터 센터에 걸처 규모를 늘리기 어렵다.

- ID의 유일성은 보장되지만 값이 시간 흐름에 맞추어 커지도록 보장할 수는 없다.

- 서버를 추가하거나 삭제할 때도 잘 동작하도록 만들기 어렵다.

  

##### UUID

유일성이 보장되는 ID를 만드는 방법이고, 128비트짜리 식별자이다. UUID는 충돌 가능성이 지극히 작다.

매 초 10억개의 uuid를 100년에 걸쳐서 생성할 때 단 하나의 uuid가 중복될 확률은 50%이다.

UUID는 63075E8C-3DE6-4584-87F1-0278049CA2D4 와 같은 형태를 띤다.

###### 장점

- UUID를 만드는 것은 단순하고 동기화 이슈가 없다.
- 각 서버가 자기가 쓸 ID를 알아서 만드므로 규모 확장도 쉽다.

###### 단점

- ID가 128비트로 길다.

- ID를 시간 순으로 정렬할 수 없다.

- ID에 숫자가 아닌 값이 포함될 수 있다.

  

##### 티켓 서버

티켓 서버는 auto_increment 기능을 갖춘 데이터베이스 서버, 티켓 서버를 중앙 집중형으로 하나만 사용하는 것이다.

###### 장점

- 유일성이 보장되는 숫자로만 구성된 ID를 쉽게 만들 수 있다.
- 구현하기 쉽고, 중소 규모 애플리케이션에 적합하다.

###### 단점

- 티켓 서버가 SPOF(Single Point Of Failure)가 된다. 서버에 장애가 발생하면 해당 서버를 이용하는 모든 시스템이 영향을 받는다.

  

##### 트위터 스노플레이크 접근법

트위터에서 사용하는 독창적인 ID 생성 기법이다.

생성해야 하는 ID의 구조를 여러 절(section) 으로 분할한다.

![image-20220405215823810](./image/7/image-20220405215823810.png)

- 사인 비트 : 1비트를 할당한다. 음수와 양수를 구별하는 데 사용 가능하다.
- 타임스탬프 : 41비트를 할당한다. 기원 시각 이후로 몇 밀리초가 경과했는지 나타낸다.
- 데이터센터 ID : 5비트를 할당하여 32개 데이터센터를 지원한다.
- 서버 ID : 5비트를 할당항 데이터센터 당 32개 서버를 사용할 수 있다.
- 일련번호 : 12비트를 할당하고 각 서버에서는 ID를 생성할 때마다 일련번호를 1만큼 증가시킨다. 이 값은 1밀리초가 경과될 때마다 0으로 초기화된다.



### 3단계. 상세 설계

트위터 스노플레이크 접근법의 세부적인 쓰임새는 다음과 같다.

##### 타임스탬프

타임스탬프는 41비트를 차지하고 있으며 시간 흐름에 따라 점점 큰 값을 갖게 되므로 시간 순으로 정렬 가능하다.

41비트로 표현할 수 있는 타임스탬프의 최댓값은 대략 69년에 해당한다.

##### 일련번호

일련번호는 12비트이므로 같은 밀리초2^12 = 4096개의 값을 가질 수 있다. 



### 4단계. 마무리

ID 생성기 구현에는 다양한 전략이 쓰일 수 있으며, 트위터 스노플레이크 같은 경우 다양한 요구사항을 만족하면서 분산 환경에서 규모 확장이 가능하다는 장점이 있다.

다음과 같은 사항들을 추가로 고려할 수 있다.

- 시계 동기화 : ID 생성 서버들이 같은 시계를 사용한다고 가정하였지만, 이런 가정은 하나의 서버가 여러 코어에서 실행된 경우 유효하지 않을 수 있다. (시차)
- 각 절 길이 최적화 : 동시성이 낮고  수명이 길다면 일련번호 절의 길이를 줄이고 타임스탬프 절의 길이를 늘리는 것이 효과적일 수 있다.
- 고가용성 : ID 생성기는 필수 불가결 컴포넌트이기 때문에 높은 가용성이 필요하다.