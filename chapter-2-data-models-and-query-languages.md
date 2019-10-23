---
description: Part 1. Foundations of Data Systems
---

# CHAPTER 2 Data Models and Query Languages

대부분의 애플리케이션은 하나의 데이터 모델을 다른 데이터 모델 위에 계층화하여 빌드된다. 예를 들어,

1. Application Developer는 실제 세계를 보며 객체 / Data Structure / API의 모델을 만든다. 보통 이 단계에서의 Application에 Specific하다.
2. 1번의 Data Structure를 저장하려고 할때 그것을 범용적인 데이터 모델로 표현한다. Json / XML / RDB의 Table / Graph model 등으로 표현된다.
3. Database를 구축할 엔지니어는 2번에서 이야기한 데이터 모델들을 어떻게 표현할지 고민한다. 예를 들어, 메모리 / 디스크 / 네트워크 등 어디에 데이터 Bytes들을 표현할지 결정하게 된다.
4. 하드웨어 엔지니어들은 해당하는 electrical currents, pulses of light, magnetic fields 등 Bytes들을 어떻게 표현할지 결정한다.

이러한 Data Model은 소프트웨어에 깊은 영향을 끼치기 때문에 Application에 맞는 적절한 것을 고르는 것이 중요하다.

## Relational Model Versus Document Model

Relational Model은 초기에는 Business Data Processing\(Transaction Processing\)을 목적으로 만들어졌고 사용되었지만 지금은 거의 모든 것들인 RDB를 바탕으로 만들어지고 있다.

### The Birth of NoSQL

2010년대에 RDB를 위협하는 NoSQL이 나타났다. "NoSQL"이라는 이름은 기술적인 것을 뜻하는 것이 아니고 distributed, nonrelational databases와 관련된 Open Source Meetup을 위한 Twitter Hashtag로 쓰이던 것이다. 시간이 지나 "Not Only SQL"이라는 뜻으로 다시 해석되었다.

NoSQL이 인기를 끌게 된 이유.

* RDB보다 쉽게 확장할 수 있고 아주 큰 데이터셋을 저장할 수 있고 큰 Write Throughput을 처리할 수 있는 DB의 필요성
* 무료 오픈 소스 소프트웨어에 대한 선호
* RDB에서는 지원하지 않는 특정 Query Operation
* Relational Schema의 제한적인 부분에 대한 절망 / Dynamic하고 표현적인 데이터 모델에 대한 열망

Non-relational DB가 인기를 끌고는 있지만 Application마다 요구사항이 다르고 적절한 DB가 다르기 때문에 RDB 또한 계속 사용될 것으로 보인다.

### The Object-Relational Mismatch

RDB의 모델과 Programming language의 모델 사이에 차이가 발생하는 문제점을 **Impedance mismatch**라고 한다. ORM\(Object-relational mapping\)이 이 둘 사이의 Transition Layer를 위한 Boilerplate Code의 양을 줄이는 것에는 도움이 되었지만 완벽하게 두 모델 사이의 차이를 없애지는 못한다.

예시로 LinkedIn Profile같은 Resume 모델을 표현해보겠다.

Resume의 대부분은 여러가지 직업과 다양한 교육 기간 등 수많은 one-to-many relationship이 있다. 이로 인해 RDB로 구성하였을 경우에 하나의 프로필을 가져오기 위해 많은 join과 많은 쿼리를 사용해야 한다. 하지만 이를 JSON으로 구성했을 때는 하나의 쿼리로 충분하다.

user profile의 one-to-many relationships은 tree 구조로 표현할 수 있고 JSON은 이 tree 구조를 명시적으로 만들어준다.

### Many-to-One and Many-to-Many Relationships

Resume의 Region이나 Industries와 같이 free-text field로도 나타낼 수 있고 ID로 나타낼 수 있는 필드가 있다. 물론 free-text로 표현해도 되지만 ID로 했을때의 장점들이 있다.

1. Consistent style and spelling across profiles 여러 프로필들 사이에서 같은 스타일과 스펠링을 유지할 수 있다.
2. Avoiding ambiguity 같은 이름을 가진 여러 도시가 있을 수 있으므로 그런 모호함을 피할 수 있다.
3. Ease of updating 이름이 한곳에 저장되어있기 때문에 업데이트가 쉽다.
4. Localization support
5. Better search
6. No meaning to humans 사람에게 어떠한 의미도 없기때문에 바뀔 일이 없다.

하지만 이렇게 ID로 Normalizing할 수 있는 건 Many-to-One 관계일때만 가능하다. 또 이러한 관계는 Document Model에는 잘 맞지 않는다. 또 RDB에서는 이렇게 field 값을 ID로 표현하는 것이 일반적이지만 Document DB에서는 이런 일이 필요없기도 하고 Join을 잘 지원하지 않는다. 만약 Join을 지원하지 않는 DB를 사용한다면 여러번의 Query를 통해 데이터를 만들어야 한다.

개발 초기에는 Join이 필요없는 model일지 몰라도, 점점 데이터는 서로 엮이고 관계가 생기게 된다.

Resume에서 하나의 유저에 대한 정보는 하나의 Document로 표현할 수 있지만 만약 school, organization 등 새로운 feature들이 추가될수록 다른 유저들과 이 feature의 값들을 서로 참조를 통해 나타내게 된다. 이런 식으로 feature들이 추가되면 Many-to-Many 관계가 필요해지게 된다.

### Are Document Databases Repeating History?

위에서 얘기했던 Many-to-Many 관계나 Table Join과 같은 데이터의 관계에 대한 논쟁은 이전부터 있었다. 60~70년대에 IBM의 IMS라는 DB에서는 데이터를 _hierarchical model_로 표현했었는데 이는 지금의 Document DB의 Json 모델과 비슷하다. 70년대에도 똑같이 Data 중복이나 서로간의 참조에 대한 고민을 해왔었다. 이를 위한 해결책으로 나왔던 것이 Relational model\(SQL\)과 Network model이다.

#### Network model 

hierarchical model을 일반화한 모양이다. hierarchical model은 tree구조로 각각 하나의 부모만을 갖지만 Network model은 여러 부모를 가질 수 있다. RDB에서의 외래키는 아니고 programming언어의 pointer와 비슷한 개념이다.   
Network model에서는 하나의 record를 찾으려면 root record부터 링크를 따라가며 경로를 따라가야 하는데 이를 access path라 한다. 이를 application developer가 구성하고 알고 있어야 한다.

#### Relational model

a collection of tuples. Query Optimizer가 자동으로 어떤 순서로 어떤 index를 사용할지 결정하고 사용한다.

#### Comparison to document databases

Document DB는 nested record를 다른 테이블에 따로 저장하는 것이 아닌 부모 record에 함께 저장한다는 점에서는 hierarchical model의 모습을 보인다.  
그러나 Many-to-One, Many-to-Many 관계를 표현할 때는 Relational DB와 근본적으로 다르지 않다. 둘다 unique identifier를 통해 서로 관련있는 아이템들을 참조하고 있다.   
\(RDB : foreign key / Document DB : document reference\)

### Relational Versus Document Databases Today

여러 차이점이 있지만 이 챕터에서는 Data model의 차이를 중점적으로 보겠다. 

* Document DB 
  * Schema의 유연성 -&gt; application code와 비슷한 data 구조 
  * Locality로 인한 좋은 퍼포먼스
* RDB
  * join에 대한 지원
  * many-to-one and many-to-many 관계에 대한 지원

#### Which data model leads to simpler application code?

* Document DB는 Document의 nested item에 대해 직접적으로 참조하지 못한다. hierarchical model의 access path처럼 타고타고 들어가서 참조해야 한다. -&gt; Document가 deeply nested가 아니라면 문제될 것 없다.
* Document DB는 join에 대한 지원이 좋지 않다. -&gt; application에 따라 문가 안될 수도 있다.
* many-to-many relationships을 사용한다면 RDB에서는 간단히 Join으로 해결되는 점이 Document DB에서는 복잡한 application code를 만들고 퍼포먼스를 안좋게 만든다.

#### Schema flexibility in the document model

**No schema**란 document에 어떤 key와 value가 들어올지 쓰여질지 모른다는 의미와 document를 읽을 때 어떤 필드들이 포함되어 오는지 보장되지 않는다는 의미이다. 

하지만 보통 application code를 통해 document를 읽을때 어떤 Structure를 가지고 데이터를 읽기 때문에 Document DB는 _**schema-on-read**_라고 할 수 있다. \(Relational DB는 _**schema-on-write**_\)

이 차이는 Data의 구조를 바꾸게 되었을때 극명하게 보인다.   
만약 `name`을 `firstName, lastName`으로 나누게 되면,   
  
`if (user && user.name && !user.first_name) {  
    user.first_name = user.name.split(" ")[0];  
}`

Document DB는 이런식으로 application code에서 대응할 수 있지만 RDB는 `ALTER TABLE`이 필요하다. 이 작업은 DB를 느리게 하고 downtime이 필요할 수도 있다.

한 collection안에 있는 item들이 모두 같은 구조를 가질 필요가 없을때는 Document DB의 schema-on-read가 유리하다.

#### Data locality for queries

Document는 보통 json / xml / birany 등으로 하나의 연속된 문자열로 저장된다. 만약 전체 document에 접근하는일이 잦다면 이러한 _**storage locality**_가 이점이 될 수 있다. 

storage locality가 이점을 보이는 것은 사실 전체 document를 읽어올 때뿐이다. 만약 document의 한 부분만 필요한 경우, 혹은 작은 부분만 수정했을 경우에 Document DB는 전체 document를 **다 읽고 다 다시 쓰게 된다.** 이러한 성능상의 제한 때문에 Document DB가 유용한 상황은 많지 않다. 심지어 RDB에서도 서로 관련있는 item들을 묶어서 locality를 증가시킬 수 있다. 

#### Convergence of document and relational databases

대부분의 RDB들이 2000년대 이후로 XML을 지원하면서 RDB에서도 Document DB에서와 비슷한 Data model을 사용할 수 있게 되었다. PostgreSQL은 JSON type 또한 지원한다.  
Document DB에서는 RethinkDB가 RDB와 비슷한 join을 지원하고 MongoDB driver 중 몇몇은 자동으로 DB의 reference를 파악한다. 물론 RDB의 join 보다는 느리지만 client-side join을 효과적으로 수행해준다.

## Query Languages for Data

SQL과 같은 Declarative한 Query Language에서는 어떤 데이터를 원하는지 표현하기만 하면 된다. 어떻게 데이터를 찾아올지는 표현할 필요없다. 그건 Database System의 query optimizer가 어떻게 할 지 결정하고 결과를 만들어 준다. 

Declrartive Query Language는 DB 엔진의 구현을 숨길 수 있어 Query 변경 없이 성능 향상을 위한 수정을 할 수 있고 Parallel execution도 쉽게 구현 할 수 있다.

### MapReduce Querying

MapReduce는 대량의 data를 한번에 여러 머신에서 가공할때 사용하는 programming model이다. 함수형 프로그래밍 언어에서 많이 볼 수 있는 `map, reduce` 함수를 기반으로 한다. MongoDB와 같은 NoSQL에서도 많은 document 사이에서 read-only query를 수행할때 사용되기도 한다. 

MapReduce는 클러스터에서 distributed execution을 수행하기 위한 low-level 모델이다. SQL 또한 distributed execution 구현을 MapReduce로 할 수도 있지만 그렇지 않은 구현도 있다. MapReduce가 distributed query execution의 정답은 아니다.

MapReduce는 map 함수, reduce 함수 2가지를 작성해야 하는데 이를 극복하기 위해 MongoDB 2.2에서 aggregation pipeline을 추가했다. 

```text
db.observations.aggregate([
    { $match: { family: "Sharks" } },
    { $group: {
        _id: {
            year: { $year: "$observationTimestamp" },
            month: { $month: "$observationTimestamp" }
        },
        totalAnimals: { $sum: "$numAnimals" }
    } }
]);
```

## Graph-Like Data Models

간단한 many-to-many 관계에서는 Relational Model로도 표현 가능하지만 점점 복잡해 질수록 점점 그래프 모양의 data를 갖게 된다. Graph는 vertices와 edges로 이루어져있고 거의 모든 데이터를 표현할 수 있다. 

### Property Graphs

vertex 는

* unique id
* outgoing edges
* incoming edges
* properties \(key-value pairs\)

edge는

* unique id
* edge의 시작에 있는 vertex = tail vertex
* edge의 끝에 있는 vertex = head vertex
* tail vertex와 head vertex의 관계를 나타내는 label
* properties \(key-value pairs\)

Graph는 Vertex와 Edge 이 2개의 Relation table로 이루어져있다고 생각하면 된다.

1. 서로 다른 vertex를 연결하는 edge에는 schema같은 제약사항이 없다. 어떤 vertex든 상관없이 아무 vertex와 다 연결 가능하다. 
2. 어떤 vertex든, 그 vertex의 incoming과 outgoing을 알기 때문에 효율적으로 graph를 순회할 수 있다. 
3. 다양한 label을 사용해서 하나의 그래프에 다양한 종류의 정보를 저장할 수 있다. 

이러한 특징들이 Graph model에 유연성을 준다. 

### Graph Queries in SQL

위에서 graph data가 Relational DB로도 표현할 수 있다고 했는데 SQL에 또한 Graph DB에서 사용 가능하다. 

하지만 Graph DB에서는 원하는 vertex를 찾을때까지 계속 순회해야 하기 때문에 join의 횟수를 알 수 없다. 또 Graph DB를 위한 Query Language를 사용하면 단 몇줄로 가능한 Query를 상당히 많은 줄로 작성해야 한다. 따라서 Application에 맞는 DB를 선택하는 것이 중요하다. 

### Triple-Stores and SPARQL

Triple-Stores model은 비슷한 아이디어를 조금 다르게 표현했을뿐, Property Graph와 거의 같다. 

Triple-Stores에서는 Data를 3가지 파트\(`subject, predicate, object`\)로 저장한다. `subject`는 graph의 vertex와 같은 것이고 `object`는 `predicate`에 따라 vertex일때도 있고 property의 value일때도 있다. 

1. Object가 string이나 숫자같은 타입일 경우에는 property의 value이다. ex\) `Lucy(subject), age(predicate), 33(object)`
2. Object는 vertex가 될 수 있다. 이 경우에는 predicate가 edge가 되고 subject는 tail vertex, object는 head vertex가 된다. ex\) `Lucy(subject), marriedTo(predicate), alain(object)`



  
  




