## 카프카 내부 메커니즘

### 클러스터 멤버십

---

- 카프카는 현재 클러스터의 멤버인 브로커들의 목록을 유지하기 위해 **아파치 주키퍼**를 사용
- **각 브로커는** <span style='color:#f7b731'>브로커 설정 파일에 정의</span>되었거나 아니면 <span style='color:#f7b731'>자동으로 생성된 고유한 식별자</span>를 지님
- 브로커 프로세스는 시작될 때마다 <span style='color:#f7b731'>주키퍼에 Ephemeral(임시) 노드의 형태로 ID를 등록</span>

---

- 컨트롤러를 포함한 카프카 브로커들과 몇몇 생태계 툴들은 브로커가 등록되는 주키퍼의 /brokers/ids 경로를 구독함으로써 브로커가 추가되거나 제거될 때마다 알림을 받음
- 만약 동일한 ID를 가진 다른 브로커를 시작한다면 에러가 발생
	- 새 브로커는 자신의 ID 를 등록하려 하겠지만, 이미 동일한 브로커 ID 를 갖는 **Znode**가 있기 때문

---

> **znode**는 클러스터를 구성하고 있는 각각의 서버(컴퓨터)를 뜻 함
> 1. **Persistent znode**: 명시적으로 제거되기 전에는 계속해서 zookeeper의 데이터 트리에 남아있는 데이터 노드
> 2. **Ephemeral znode**: 세션에 종속되어, znode를 만든 클라이언트와의 세션이 종료되면 삭제되는 일시적인 데이터 노드

---

- 브로커와 주키퍼 간의 연결이 끊어질 경우, <span style='color:#f7b731'>Ephemeral 노드는 자동으로 주키퍼에서 삭제</span>
- 브로커가 정지하면 브로커를 나타내는 Znode가 삭제되지만, 브로커의 ID는 다른 자료구조에 의해서 남아 있게됨
	- 각 토픽의 레플리카 목록에는 해당 레플리카를 저장하는 브로커의 ID가 포함됨
	- 그렇기 때문에 만약 특정한 브로커가 완전히 유실되어 동일한 ID를 가진 새로운 브로커를 투입할 경우, 곧바로 클러스터에서 유실된 브로커의 자리를 대신해서 이전 브로커의 토픽과 파티션들을 할당 받음

---

### 컨트롤러

---

- 클러스터에서 **가장 먼저 시작되는 브로커**는 주키퍼의 <span style='color:#f7b731'>/controller 에 Ephemeral노드를 생성함으로써 컨트롤러</span>가 됨
- 컨트롤러는 일반적인 카프카 브로커의 기능을 하면서, <span style='color:#f7b731'>파티션 리더를 선출하는 역할</span>을 추가적으로 맡음

---

- 다른 브로커 역시 시작할 때 해당 위치에 노드를 생성하려고 시도하지만, <span style='color:#f7b731'>'노드가 이미 존재함' 예외를 받게 되기 때문에 컨트롤러 노드가 이미 존재</span>한다는 것을 알아차리게 됨
- 브로커들은 주키퍼의 <span style='color:#f7b731'>컨트롤러 노드에 뭔가 변동이 생겼을 때 알림을 받기 위해</span>서 이 **노드에 와치를 설정**함
- 이 때문에 클러스터 안에 **단 한개의 컨트롤러만 존재**할 수 있도록 보장할 수 있음

---

- 컨트롤러 브로커가 멈추거나 주키퍼와의  연결이 끊어질 경우, 이 **Ephermal 노드는 삭제**됨
- Ephermal 노드가 삭제될 경우, 클러스터 안의 <span style='color:#f7b731'>다른 브로커들은 주키퍼에 설정된 와치를 통해 컨트롤러가 없어졌다는 것을 알아차리게 됨</span>
- 주키퍼에 <span style='color:#f7b731'>가장 먼저 새로운 노드를 생성하는데 성공한 브로커가 다음 컨트롤러</span>가 되며, 다른 브로커들은 이전처럼 노드가 이미 존재함 예외를 받고 **새 컨트롤러 노드에 대한 와치를 다시 생성**

---

- 브로커는 새로운 컨트롤러가 선출될 때마다 **주키퍼의 조건적 증가 연산**에 의해 증가된 에포크 값 을 전달 받음
- 브로커는 현재 컨트롤러의 에포크 값을 알고 있기 때문에, 만약 <span style='color:#f7b731'>더 낮은 에포크 값을 가진 컨트롤러로부터 메시지를 받을 경우 무시</span>함

---

> 이것은 컨트롤러 브로커가 오랫동안 가비지 수집 때문에 멈춘 사이 주키퍼 사이의 연결이 끊어질 수 있기 때문에 중요하다. 이전 컨트롤러가 작업을 재개할 경우, 새로운 컨트롤러가 선출 되었다는 것을 알지 못한 채 브로커에 메시지를 보낼 수 있다. **이러한 컨트롤러를 좀비**라고 부르는데, 컨트롤러가 전송하는 메시지에 컨트롤러 에포크를 포함하면 브로커는 예전 컨트롤러가 보내온 메시지를 무시할 수 있다.

---

- 브로커가 컨트롤러가 되면, 클로스터 메타데이터 관리와 리더 선출을 시작하기 전에 먼저 주키퍼로부터 최신 레플리카 상태 맵을 읽어옴
- 이 적재 작업은 <span style='color:#f7b731'>비동기 API를 사용해서 수행</span>되며, 지연을 줄이기 위해 읽기 요청을 여러 단계로 나눠서 주키퍼로 보냄

---

- 브로커가 클러스터를 나갔다는 사실을 컨트롤러가 알아차리면, <span style='color:#f7b731'>컨트롤러는 해당 브로커가 리더를 맡고 있었던 모든 파티션에 대해 새로운 브로커를 할당</span>해줌 
- 컨트롤러는 새로운 리더가 필요한<span style='color:#f7b731'> 모든 파티션을 순회해 가면서 새로운 리더가 될 브로커를 결정</span>
- 그러고 나서 새로운 상태를 주키퍼에 쓴 뒤, 새로 리더가 할당된 파티션의 레플리카를 포함하는 모든 브로커에 **LeaderAndISR 요청을 보냄**

---

> LeaderAndISR 요청은 각 파티션의 데이터의 리더쉽과 복제를 관리하기 위해 Kafka 브로커들 사이에서 사용되는 통신 메커니즘
> 
> Apache Kafka와 LeaderAndISR 요청의 맥락에서 설명되는 **"팔로워"는 현재 리더가 아닌 파티션의 복제본을 의미**한다. Kafka에서 각 파티션은 하나의 리더와 여러 개의 팔로워를 가진다. **리더는** <span style='color:#f7b731'>읽기와 쓰기를 처리하는 역할</span>을 맡고, **팔로워**는 <span style='color:#f7b731'>오류 허용 및 고가용성을 위해 리더의 데이터를 복제</span>한다.  
> 
> 브로커가 특정 파티션에 대한 팔로워가 될 때는 파티션의 데이터 복사본을 유지하고 리더와 동기화 상태를 유지하는 것을 의미한다. 팔로워는 리더가 실패했을 때 데이터 내구성과 가용성을 보장하기 위해 중요하다. 리더를 사용할 수 없게 되면 **팔로워 중 한 명이 새로운 리더가 되도록 승격**시켜 <span style='color:#f7b731'>서비스의 지속성을 보장</span>할 수 있다.

---

#### 요약
- 컨트롤러는 브로커가 클러스터에 추가되거나 재거될때 <span style='color:#f7b731'>파티션과 레플리카 중에서 리더를 선출할 책임</span>을 짐 
- 컨트롤러는 <span style='color:#f7b731'>서로 다른 2개의 브로커가 자신이 현재 컨트롤러라 생각하는 스플릿 브레인 현상을 방지</span>하기 위해 **에포크 번호를 사용**
- https://www.owl-dev.me/blog/73

---

### Kraft
- https://www.confluent.io/ko-kr/blog/removing-zookeeper-dependency-in-kafka/
---

아파치 카프카 커뮤니티는 19년도 부터 <span style='color:#f7b731'>기존의 주키퍼 기반 컨트롤러의 한계점을 극복</span>하기위해 <span style='color:#f7b731'>래프트 기반 컨트롤러 쿼럼</span>의 **KRaft**를 개발하였으며 3.3버전 이후 정식으로 출시
- https://raft.github.io/
- 
> Raft 알고리즘이란?
> - Paxos(팩소스) 알고리즘의 대안으로 설계된 분산합의 알고리즘
> - 노드 간에 안전하게 로그를 복제하고, 리더를 선출하여 클라이언트 요청을 처리함으로써 합의를 달성하는 것이 주요 목표
> - [Algorithm Raft Algorithm 소개](https://t3guild.wordpress.com/2020/03/28/algorithm-raft-algorithm/)

---

#### Kraft 개발 이유 
- 컨트롤러가 주키퍼에 메타데이터를 쓰는 작업은 동기적으로 이루어지지만, <span style='color:#f7b731'>브로커에 메시지를 보내는 작업</span>과 <span style='color:#f7b731'>주키퍼로부터 업데이트를 받는 과정이 비동기</span>이기 때문에 **브로커, 컨트롤러, 주키퍼 간에 메타데이터 불일치 발생** 가능
- 컨트롤러가 재시작될 때마다 모든 브로커와 파티션에 대한 메타데이터를 읽어와야함 
> [컨플루언트의 블로그](https://www.confluent.io/ko-kr/blog/removing-zookeeper-dependency-in-kafka/)에서는  다음과 같이 말하고 있음
   Actually, the problem is not with ZooKeeper itself but with the concept of external metadata management.(사실 문제는 ZooKeeper 자체가 아니라 외부 메타데이터 관리 개념에 있습니다.)

---

- **Kraft** 설계의 핵심 아이디어는 **카프카 그 자체**에 <span style='color:#f7b731'>사용자가 상태를 이벤트 스트림으로 나타 낼 수 있도록 하는 로그 기반 아키텍처를 도입</span>하는 것

---
#### 장점
- 다수의 컨슈머를 사용하여 이벤트를 재생함으로써 **최신 상태를 빠르게 따라 잡을 수 있음**
- 로그는 이벤트 사이에 명확한 순서를 부여하고 **컨슈머들이 항상 하나의 타임라인을 따라 움직이도록 보장**해줌
- 컨트롤러 노드들은 메타데이터 이벤트 로그를 관리하는 래프트 쿼럽이 되는데, 이 로그는 메타 데이터의 변경 내역을 저장함
	- 토픽, 파티션 등 주키퍼에 저장되어 있는 모든 정보

---

- 래프트 알고리즘을 사용함으로써 컨트롤러 노드들은 외부 시스템에 의존하지 않고 **자체적으로 리더를 선출** 가능
- 메타데이터 로그의 리더 역할을 맡고 있는 컨트롤러를 **액티브 컨트롤러**라고 부름
	- 액티브 컨트롤러는 모든 RPC(remote procedure call) 호출을 처리함
- **팔로워 컨트롤러**들은 <span style='color:#f7b731'>액티브 컨트롤러에 쓰여진 데이터들을 복제</span>하며, <span style='color:#f7b731'>액티브 콘트롤러에 장애가 발생했을 시 즉시 투입될 수 있도록 준비상태를 유지</span>
 ➡️ <mark style='background:#8854d0'>컨트롤러들이 모두 최신 상태를 가지고 있기 때문에, 컨트롤러 장애 복구는 모든 상태를 새 컨트롤러로 이전하는 리로드 기간을 필요로 하지 않음</mark>

---

- 컨틀롤러가 브로커에 <span style='color:#f7b731'>변경 사항을 push 하는 것이 아닌</span> 브로커들이 액티브 컨틀롤러로부터 **변경 사항을 pull**함
- 컨트롤러 쿼럼으로의 마이그레이션 작업의 일부로서, 기존에는 <span style='color:#f7b731'>주키퍼와 직접 통신하던 모든 클라이언트, 브로커 작업들은 이제 컨트롤러로 보내지게 됨</span>
➡️ <mark style='background:#8854d0'>이렇게 함으로써<span style='color:#f7b731'> 브로커 쪽에는 아무것도 바꿀 필요 없이</span>, 컨트롤러만을 바꿔 주는 것만으로 매끄러운 마이그레이션이 가능해짐</mark>

쿼럼이란? 위 내용을 그냥 복사해서 싹다 gpt에 물어보자

---

#### 어떻게 Kraft로 옮겨갈 수 있을까?
- chapter 6 155 p.

---