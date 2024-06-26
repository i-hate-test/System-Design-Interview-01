# 5장. 안정 해시 설계
> **해시(Hash)**:
> 결과가 고정된 길이의 문자열로 나오는 함수를 적용하여 숫자 또는 영숫자 구성의 문자열을 할당하는 것.
## 대규모 시스템과 해시 설게
### 수평적 규모 확장성
- 수평적 규모 확장성을 달성하기 위해서는 요청 또는 데이터를 서버에 균등하게 나누는 것이 중요.
- 이를 위해 클라이언트 해시 값에 특정 함수를 적용한 결과 값을 서버 인덱스로 사용하는 방법이 있다.
- 일반적으로 나머지 연산(Modular) 함수로 해싱한다. ( 서버 인덱스 = 해시(K) % 서버 pool(n) )
### 재배치 문제 
- 장애 등의 이유로 서버 pool이 감소한 경우 클라이언트의 서버 인덱스로 새 값이 부여되면서 엉뚱한 서버에 접속하게 된다. 
- 만약 캐시 서버로의 로드밸런싱에서 pool 의 변화가 있는 경우, 대규모 캐시 미스가 발생한다.
- 일반적인 해싱은 모든 키 값을 재배치하는데, 이는 대규모 시스템일 수록 데이터 저장소에 큰 부하가 걸리는 치명적인 문제가 있다.

## Consistent Hashing
> 해시 테이블 크기가 조정될 때 평균적으로 오직 `키 개수/슬롯 개수` 개의 키만 재배치하는 해시 기술
> 전통적인 해시 테이블은 슬롯의 수가 바뀌면 대부분 키를 재배치한다.
### 기본 개념
- 해시 공간: 해시 함수의 출력 범위를 의미. 일반적으로 SHA-1의 해시 공간은 0 ~ 2^160-1 까지
- 해시 링: 일직선의 해시 공간을 원형으로 이어 붙인 자료 구조

### 기본 구현
- 서버와 키를 균등 분포 해시 함수를 사용해 해시 링에 배치한다.
- 키의 위치에서 링을 시계 방향으로 탐색하다 만나는 최초의 서버가 해당 키의 서버가 된다.
- 장점: 서버 추가 및 삭제 시 해당 서버와 관련 있는 키 값만 재배치되고 나머지 키는 재배치되지 않는 장점이 있다.
- 단점: 서버 삭제 시 특정 서버의 파티션이 늘어남. 또한 키들이 특정 서버에만 보관될 수 있음.

### 가상 노드 (복제 노드) 방법
- 각 서버가 여러 복제 노드를 가져 여러 파티션을 관리한다. 
- Tradeoff: 노드 수가 늘어나면 표준 편차가 작아져 데이터가 고르게 분포되지만 저장 공간이 많이 필요하다.

# 6장. 키-값 저장소 설계
## 문제
1. 키-값 쌍 크기는 10KB 이하
2. 큰 데이터 저장 가능
3. 높은 가용성: 장애가 있더라도 빨리 응답
4. 높은 규모 확장성: 트래픽 양에 따라 자동 증설/삭제
5. 데이터 일관성 수준 조절 가능
6. 짧은 응답 지연시간

## 분산 해시 테이블
### CAP 정리
> Consistency: 모든 클라이언트는 노드 상관 없이 동일한 데이터를 본다.  
> Availability: 일부 노드에 장애가 있더라도 항상 응답을 받을 수 있다.  
> Partition Tolerance: Partition(노드 간 통신 장애)이 발생하더라도 시스템은 계속 동작한다.  

- CAP 모두를 동시에 만족하는 분산 시스템 설계는 불가능하다.
- 이상적으로, 네트워크 에러가 발생하지 않는 전제 하에, 모든 노드간 데이터가 자동적으로 복제되면 CAP 모두를 만족할 수 있다.
- 실세계: 한 노드의 장애로 파티션 발생
  - CP 시스템: 나머지 노드의 쓰기 연산 중단 => 가용성 X. ex. 은행권
  - AP 시스템: 파티션 문제 해결 이후 새 데이터를 복구된 노드에 전송

### 데이터 파티션
> 대규모 애플리케이션의 경우 데이터를 작은 파티션을 분할 후 여러 서버에 저장한다.
**[ 안정 해시 테이블 ]**
- 데이터를 여러 서버에 고르게 분산할 수 있는가? Y
- 노드 갯수의 수정 시 데이터 이동을 최소화하는가? Y
- 규모 확장 자동화 가능
- 가상 노드 갯수 조정 가능

### 데이터 다중화
> 높은 가용성을 위해 데이터를 N개 서버에 비동기적으로 다중화할 필요가 있다.
- 안정 해시 테이블의 경우, 링을 순회하면서 만나는 첫 N개 서버에 데이터를 보관하는 방법이 있다.
- 만약 가상 노드를 사용하는 경우, 중복되는 물리 서버를 선택하지 않아야 한다.
  - 예를 들어, N=3, 처음 만나는 노드들이 1-1, 2-2, 1-2, 3-1 인 경우 1-2를 선택하지 않아야 한다.

### 데이터 일관성
> 여러 노드에 다중화된 데이터는 적절히 동기화되어야 한다.

**[ 정족수 합의 프로토콜 ]**
- 노드 목록의 프록시 역할을 하는 중재자가 정족수 기준에 맞게 데이터를 동기화한다.
- N = 사본 개수 / W = 쓰기 연산 정족수 / R = 읽기 연산 정족수
- tradeoff: 응답 지연 vs 데이터 일관성
  - R = 1, W = N: 빠른 읽기 연산
  - W = 1, W = R: 빠른 쓰기 연산
  - W + R 이 N보다 높을 수록 강한 일관성 보장

**[ 최종 일관성 모델 ]**
- 강한 일관성을 유지하려면 쓰기 연산이 완료될 때 까지 모든 읽기/쓰기가 금지되므로 가용성이 떨어진다.
- 갱신 결과가 결과적으로는 동기화 되는 최종 일관성 모델을 따르면서 **비일관성을 해소**하는 기법이 등장한다.
  - 예를 들어, 쓰기 연산이 병렬적으로 발생하면 저장된 값 간의 일관성이 깨짐. 
  - 이를 해소하기 위해선 일관성이 깨진 값을 읽은 클라이언트가 해결해야 한다.

**[ 데이터 버저닝 ]**
- 버저닝: 데이터를 갱신할 때마다 변경 불가능한 새로운 버전을 만드는 것.
- 벡터 시계: [서버, 버전]의 순서쌍을 데이터에 매단 것. D([S1, v1], [S2, v2], ...[Sn, vn])  
예를 들어 벡터 시계 D([S0, 1], [S1, 1])은 D([S0, 1], [S1, 2])의 이전 버전이다.
- 클라이언트 구현이 복잡해짐
- 순서쌍 개수가 굉장히 빨리 늘어나므로 임계치를 설정하고 오래된 순서쌍을 시계에서 제거해야 함.

### 장애 감지 및 처리
**[ 분산형 장애 감지 시스템 - 가십 프로토콜 ]**
> 모든 노드 사이의 멀티캐스팅 채널 구축하는 방법은 가장 쉽지만 서버가 많을 수록 비효율적
- 각 노드는 멤버십 목록[Member ID, Heartbeat counter, Time]을 유지.
- 각 노드는 주기적으로 자신의 박동 카운터를 증가
- 각 노드는 무작위로 선정된 노드들에게 주기적으로 자신의 멤버십 목록을 전송
- 목록을 받은 노드는 멤버십 목록을 최신 값으로 갱신(Counter ++, Time 갱신) 후 재전송
- 특정 멤버의 박동 카운터가 일정 시간 안에 갱신되지 않으면 장애 처리

**[ 일시적 장애 처리를 위한 hinted handoff 기법 ]**  
>**느슨한 정족수 접근법**
>- 가용성을 유지하기 위해 정족수 조건을 완화하는 것. 
>- 엄격한 정족수 = 강한 일관성 = 읽기 쓰기 연산 금지 = 가용성 감소  
>- 정족수 기준을 완화 = R, W 개수의 감소 = 장애 노드를 제외한 건강한 노드 선정하여 R, W 연산 진행

임시 노드에 장애 노드에 대한 힌트와 함께 데이터를 임시로 저장하고, 장애 노드가 복구되면 데이터를 인계하여 데이터 일관성을 보장한다.

**[ 영구 장애 처리를 위한 반-엔트로피 프로토콜 & 머클 트리 ]**
> 일시적 장애 서버는 서버 복구를 기다릴 수 있기 때문에 복구 이후 작업을 위한 준비를 하면 된다.  
하지만 장애가 영구적으로 발생한 경우, 사본 간 데이터를 완전히 동기화할 수 있는 방법을 생각해야 한다.

**반-엔트로피(anti-entropy) 프로토콜**  
사본들을 비교하여 최신 버전으로 갱신하는 과정을 포함하는 규약  

**머클 트리(해시 트리)**  
각 노드가 자식 노들이 보관 중인 값의 해시 또는 자식 노드들의 레이블로부터 계산된 해시 값을 레이블로 붙여두는 트리 구조.
머클 트리의 비교를 통해 서로 데이터가 다른 노드를 발견한 후, 이들만 동기화를 진행함으로써 효과적으로 데이터 일관성을 유지할 수 있다.

**[ 데이터 센터 장애 처리 ]**
정전, 네트워크 장애, 자연 재해 등의 요인이 있다.
데이터를 여러 데이터 센터에 다중화하는 것이 중요하다.

### 시스템 아키텍처 다이어그램
