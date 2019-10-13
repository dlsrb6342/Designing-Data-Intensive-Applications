---
description: Part 1. Foundations of Data Systems
---

# CHAPTER 1 Reliable, Scalable, and Maintainable Applications

## Reliable, Scalable, and Maintainable Applications

오늘날의 Application의 대부분은 `compute-intensive`가 아닌 `data-intensive`로 변화하였다. CPU Power가 아닌 데이터의 양이나 복잡성 또는 변경 속도가 가장 중요한 이슈가 된것이다.

data-intensive application들은 일반적으로 **필요로 하는 몇가지 기능**들을 제공하는 식으로 만들어진다.

* Database - 필요시에 다시 찾을 수 있도록 데이터를 저장한다.
* Caches - read 속도를 높이기 위해 큰 연산이 필요한 작업은 결과를 기억해둔다.
* Search Indexes - keyword나 여러가지 방법으로 데이터를 필터링하여 검색할 수 있다.
* Stream Processing - 비동기적으로 동작할 수 있도록 다른 프로세스에 메시지를 전달한다.
* Batch Processing - 주기적으로 대량의 데이터 분석/처리한다.

이 사항들이 굉장히 당연하다고 느껴지지만 실상은 그렇게 간단하지 않다. 수많은 종류의 Database가 있고 여러 가지의 indexing 방법이 있다. 이중에서 자신의 상황에 맞는 솔루션을 고르는 일은 어려운 일이다.

이 챕터에서는 Reliable, Scalable, Maintainable한 Data System을 만들기 위한 방법을 확인할 것이다.

### Thinking About Data Systems

우리는 Database, Queue, Cache 등등을 서로 굉장히 다른 것이라 생각한다. 데이터를 일정시간동안 저장한다는 점에서 비슷한 것들이지만 서로 아주 다른 특징을 가지고 있고 그렇기 때문에 구현방법도 굉장히 다르다.

**그렇게 다른점이 분명한데 왜 이러한 것들을 모두 Data System이라고 묶어서 불러야 할까?**

1. 최근 몇년간 데이터 저장 및 처리에 새로운 도구들이 등장했다. 예를 들어, Message Queue로도 사용되는 Datastore인 **Redis**, Database처럼 내구성이 보장되는 Message Queue인 **Apache Kafka**가 있다. 이렇게 서로의 Boundary가 불투명해진 것이다.
2. 많은 Application들의 데이터 처리와 저장에 대한 요구사항의 범위가 넓어져서 더이상 한가지 tool로는 만족시킬 수 없게 되었다. 따라서, 그러한 작업을 여러개로 쪼개어 여러 다른 tool에서 처리할 수 있도록 변화하였다.

하나의 서비스에서 다양한 tool을 합쳐서 제공하려 할때, 보통 이러한 구현은 Client에게 드러내지 않고 API를 제공한다. **Application Developer는 이제 단순히 application만 만드는 것이 아닌 Data System Engineer의 역할도 하게 되었다.**

Data System / Service를 설계하다보면 여러가지 문제들이 떠오른다.

1. 만약 내부적으로 문제가 생겼다하더라도, 어떻게 데이터를 정확하고 완벽하게 유지할 것인가?
2. 어떻게 항상 Client에게 좋은 퍼포먼스의 API를 제공할 것인가?
3. 만약 load가 높아진다면 어떻게 대처할 것인가?
4. Service를 좋아보이게 하는 API는 어떤 모습인가?

이 책에서 우린 3가지 Factor에 대해 집중할 것이다.

1. **Reliability**

   Hardware Fault, Software Fault 혹은 Human error가 있다고 해도, 시스템은 항상 정확하게 동작해야 한다.

2. **Scalability**

   데이터의 양이나 트래픽, 복잡성이 증가했을때, 이를 대처할 수 있는 합리적인 방법이 있어야 한다.

3. **Maintainability**

   시간이 흐름에 따라, 다양한 사람이 이 시스템과 일을 할 것인데 이 모든 사람이 생산적으로 일할 수 있어야 한다.

### Reliability

Software에게 Reliable은 다음과 같은 의미들을 포함한다.

1. User가 예상한대로 행동해야 한다.
2. User가 실수를 하거나 예상치 못한 방법으로 사용할 때도 견딜 수 있어야 한다.
3. 예상되는 로드와 데이터 양에 대해서는 사용하기에 좋은 퍼포먼스를 보여야 한다.
4. 허용되지 않은 접근이나 Abusing을 막아야 한다.

Faults란 무언가 잘못될 수 있는 상황을 뜻하며 fault-tolerance or resilient는 시스템이 faults를 예상하고 대응할 수 있음을 뜻한다.

여기서, **Faults를 Failure와 같게 봐서는 안된다.** Faults란 어떤 한 컴포넌트가 그것의 스펙과 다르게 행동할 때를 뜻하는데 반해, Failure는 user에게 제공해야 할 서비스가 전부 멈췄을 때를 의미한다.

Faults가 발생하는 것을 완전히 막을 수는 없다. 따라서 우리는 Faults로 인해 Failure가 발생하는 상황을 막을 수 있도록 System을 fault-tolerance하게 설계해야 한다.

중대한 버그의 대부분은 오류 처리를 미흡하게 해서 발생한다. 고의적으로 결함을 발생시켜 System의 fault-tolerance를 훈련해볼 수 있다\(ref: Netflix의 Chaos Monkey\).

#### Hardware Faults

우리가 System Failure의 원인을 떠울렸을때 가장 먼저 생각나는 것은 Hardware Faults일 것이다. 수많은 machine을 사용하고 있다면 이런 Hardware Faults는 언제든 일어날 수 있다.

우리는 각각의 Hardware Component에 여분의 장비를 투입하여 System의 Failure Rate를 줄일 수 있다. 이 방법은 Failure를 발생시키는 Hardware 문제를 완벽히 막을 수 있는 것은 아니지만, 몇년 동안은 장비가 잘 동작하도록 할 수 있다.

최근까지는 하나의 machine이 완전히 장애를 일으키는 일은 드물기 때문에 대부분의 Application들이 이 방법으로 충분했다. 그러나 데이터 양과 Application의 computing 필요성이 증가함에 따라 많은 수의 장비를 사용하게 되었다. 이로 인해 Hardware Faults의 비율도 따라서 증가하게 되었다. 게다가 클라우드 환경에서는 VM이 경고없이 사용할 수 없게 되는 것이 흔하다.

따라서 software fault-tolerance 기술이나 하드웨어 중복성을 더해 전체 장비의 손실에 견딜 수 있는 System을 만들게 되었다. 이러한 시스템에는 rolling-update를 할 수 있는 이점도 있다.

#### Software Faults

Software Faults는 System 내부에서 발생하는 에러로, 더 예상하기가 힘들고 Hardward Faults와 다르게 다른 노드들과 연관성이 깊기 때문에 많은 System에 fault를 발생시킬 수 있다.

이러한 Software Faults를 유발하는 Bug들은 특정한 비정상적인 상황에서만 트리거되기 때문에 그전까지는 오랫동안 숨어있게 된다.

Software Faults를 빠르게 해결할 수 있는 해결책은 없다. 아래와 같은 것들에 주의를 기울이다 보면 도움이 될 것이다.

1. 시스템 안에서의 가정과 상호작용에 대해 주의깊게 생각한다.
2. 철저한 테스트를 한다.
3. 프로세스를 격리시킨다.
4. 프로세스가 크래시를 발생시키고 재시작될 수 있게 한다.
5. Production 환경에서 시스템의 상태를 모니터링하고 분석한다.

#### Human Faults

시스템을 설계하고 운영하는 사람은 신뢰할 수 없는 존재이다. 이러한 "사람"들을 데리고 신뢰할 수 있는 시스템을 만들 수 있는 방법은 무엇일까?

* 에러가 발생할 수 있는 기회를 최소화하는 방식으로 시스템을 설계한다.
* 사람들이 실수를 많이 하는 지점에서 장애를 발생시킬 수 있는 지점을 분리한다. 특히 non-production **sandbox** 환경을 제공해야 해서 사람들이 쉽고 안전하게 실험해 볼 수 있게 해야 한다.
* Unit test, Integration test, Manual test까지 모든 레벨에서 철저한 테스트를 한다.
* Human Faults로부터의 빠르고 쉬운 recovery를 제공해야 한다. 이를 통해 장애 상황의 영향을 최소화할 수 있다.
* Performance metrics나 error rates를 자세하고 명확하게 모니터링해야 한다. 이를 공학 분야에서는 **telemetry**라고 부른다. 모니터링은 문제가 발생했을때 빠른 경고를 줄 수 있고 가정이나 제약사항이 잘못되었는지 판단할 수 있게 해준다.
* 훌륭한 관리 관행과 교육을 제공한다.

#### How Important Is Reliability?

신뢰성은 거의 모든 평범한 Application에게도 중요한 요소이다. Business Application에 있는 Bug는 생산성을 잃게 할 수 있고 Ecommerce의 장애는 매출 손실 및 평판 손상 측면에서 막대한 비용이 발생할 수 있다. "noncritical"한 Application에서도 우리는 사용자에게 좋은 서비스를 제공할 책임이 있다. 신뢰성 대신에 개발 비용이나 운영 비용을 줄이는 방향을 택해야 하는 상황도 있다. 하지만 이러한 상황에서 우리는 우리가 포기한 신뢰성의 부분에 대해서 정확히 자각하고 있어야 한다.

### Scalability

지금 현재에 시스템이 reliable하다고 하더라도 미래에도 그럴 것이라고 확신할 수 없다. 유저가 늘어나거나 데이터의 양이 증가하는 식의 로드 증가가 발생한다면 더이상 시스템은 Reliability를 유지할 수 없다.

Scalability는 늘어나는 시스템의 로드에 대해 잘 대처하는 능력을 뜻하는 말이다. 하지만 이것이 일차원적인 것만을 뜻하는 것이 아닌 시스템이 어떤 특정한 방법으로 인해 차차 커졌을때 이 성장을 대처할 수 있는 옵션이 있느냐를 묻는 것이다.

#### Describing Load

첫번째, 시스템의 현재 로드를 간결하게 표현할 수 있어야 한다. 로드를 표현할 수 있는 **load parmeters**는 각각의 시스템 구조에 따라 상이하다.

#### Describing Performance

시스템의 로드를 표현할 수 있게 되면, 이제 로드가 증가했을때 무슨 일이 발생하는지 조사할 수 있게 된다. 이때 2가지 방법으로 이를 확인할 수 있다.

1. Load Parameter는 증가하지만 시스템 리소스\(CPU, Memory, Network Bandwidth...\)는 변경하지 않았을 때, 시스템의 performance는 어떤 영향이 있을까?
2. Load Parameter가 증가했을 때, 시스템의 performance를 유지시키기 위해서는 시스템 리소스를 얼마나 증가시켜야 할까?

이러한 질문의 답을 구하려면 **performance number**가 필요하다. Hadoop과 같은 Batch System에서는 **Throughput**일 것이고 Web Service에서는 **Response Time**이 될 것이다.

실제 Production 환경에서는 다양한 요청들이 들어올 것이고 따라서 response time도 모두 다 다를 것이다. 그러므로 Performance를 볼때, 딱 하나의 값만 보는 것이 아니라 Performance number의 분포를 확인해야 한다.

Response Time을 볼때, 보통 average\(arithmetic mean\)을 보려고 하지만 그렇게 좋은 수단이 아니다. 그대신 **percentiles**를 확인하는 것이 좋다. 예시로 95th percentiles은 100개의 요청 중에 가장 느렸던 5개의 요청의 평균을 뜻한다.

#### Approaches for Coping with Load

위에서 우린 load를 표현할 수 있는 Load Parameter와 performance를 계산할 수 있는 metrics에 대해 알아보았다. 이를 통해 우린 **scalability**에 대해 이야기해 볼 것이다.

* scaling up - vertical scaling이라고도 부르며, 더 powerful한 머신으로 바꾸는 것이다.
* scaling out - horizontal scaling이라고도 부르며, load를 여러 작은 머신으로 분산시키는 것이다.

Elastic System이란 시스템이 load의 증가를 감지하여 자동으로 computing resource를 추가하는 시스템을 말한다. 반면에 Elastic System이 아닌 시스템은 사람이 직접 capacity를 계산하여 머신을 추가할지 결정하고 직접 추가해야 한다. Elastic System은 load를 예측하기 어려운 환경에서 사용하기 좋지만 단점으로는 구성하기 복잡하고 operational surprises가 많을 수 있다.

Scalable한 시스템의 구조는 하나의 generic한 답은 없다. Application마다 각각의 솔루션을 가지고 있다. 만약 잘못된 가정을 가지고 Scalable한 시스템을 설계했을 때는 운좋으면 그저 낭비되는것이지만 운이 나쁘다면 오히려 역효과를 낳을 수 있다.

### Maintainability

Software 개발의 비용은 초기 개발 비용만이 아니라 그것을 유지보수하는 비용도 포함이다. 대부분의 사람들이 _legacy system_의 \*\*유지보수를 싫어하기에 우리는 유지보수의 고통을 줄이는 방향으로 시스템을 설계해야 한다.

* Operability
  * Operation team이 시스템을 사용하기 쉽게 만든다.
* Simplicity
  * 새로 온 엔지니어가 이 시스템을 쉽게 이해할 수 있도록 복잡성을 제거하여 만든다.
  * User Interface의 Simplicity와는 다른 의미이다.
* Evolvability
  * 요구사항의 변화로 인한 예상하지 못한 Use Case에 맞게 시스템을 쉽게 변화시킬 수 있도록 만든다.
  * extensibility, modifiability, plasticity 라고도 불린다.

#### Operability: Making Life Easy for Operations

좋은 Operation Team의 역할

* 시스템의 상태를 모니터링하고 문제가 생기면 빠르게 복구한다.
* System Failure나 Performance 저하와 같은 문제의 원인을 분석한다.
* 보안 패치를 포함하여 Software와 Platform을 최신 상태로 유지한다.
* 다른 시스템들이 서로에게 어떤 영향을 끼치는지 파악하고 문제점이 있는 변화들을 미리 방지한다.
* 어떤 문제가 생길지 예상하고 미리 해결한다.
* 배포를 위한 툴, Configuration 관리에 관한 좋은 Practice를 만든다.
* 시스템 Configuration이 변경될때 시스템의 보안을 유지한다.
* 예상가능하고 Production 환경을 안정적이게 유지할 수 있는 Process를 정의한다.
* 시스템에 대한 조직의 지식을 잘 보존한다.

Good operability의 의미

* 모니터링을 통해, 시스템의 내부나 실행 중의 모습을 보여줄 수 있도록 한다.
* 표준 툴을 통해 자동화와 통합을 지원한다.
* 개별 머신에 의존성을 제거한다.
* 좋은 문서와 이해하기 쉬운 운영 모델을 제공한다.
* 기본적인 동작을 좋게 제공하되, 관리자가 원한다면 쉽게 재정의할 수 있도록 한다.
* 적절한 경우에는 Self-healing을 하되, 관리자가 수동으로 시스템의 상태를 컨트롤할 수 있도록 한다.
* 예측가능한 동작을 하고 이슈를 최소화한다.

#### Simplicity: Managing Complexity

작은 프로젝트에서는 코드를 간단하고 명시적으로 유지하기 쉽지만 프로젝트가 커질수록 점점 복잡해지고 어려워진다. 그럴수록 생산성을 낮추고 유지보수 비용을 높인다.

시스템을 간단하게 만든다고 해서 반드시 그것의 기능이 줄어든다는 것은 아니다. **Accidental Complexity**\(우발적 복잡성\)을 줄인다는 의미도 포함하는 것이다. Accidental Complexity란 소프트웨어가 해결하는 문제가 아니라 구현에서 발생한 문제를 의미한다.

**Abstraction**은 Accidental Complexity를 줄이는 가장 좋은 방법 중 하나이다. 좋은 추상화는 자세한 구현 방법은 뒤에 숨겨둔채 간단하고 이해하기 쉬운 façade를 제공할 수 있다.

#### Evolvability: Making Change Easy

시스템의 요구사항이 영원히 변하지 않을 수는 없다. **Agile working patterns**은 이런 변화를 쉽게 적용할 수 있도록 도와주는 **TDD**나 **Refactoring** 같은 \*\*\*\*technical tools과 patterns을 제공한다.

Agile techniques은 대부분 작은 코드에 집중하여 이야기하고 있지만 이 책에서는 큰 Data System에서도 **agility**를 높이는 방향에 대해 찾아볼 것이다.

