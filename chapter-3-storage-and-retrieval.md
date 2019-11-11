---
description: Part 1. Foundations of Data Systems
---

# CHAPTER 3 Storage and Retrieval

Chapter 2에서는 application developer로써 어떻게 저장하고\(data model\) 어떻게 읽어올 것인지\(query\)에 대해 공부했다. Chapter 3에서는 database의 입장에서 같은 고민에 대해 이야기할 것이다.

Application Developer로써 DB Engine을 직접 구현하지는 않을 것이지만 적절한 Engine을 고르고 사용하기 위해서는 DB가 어떻게 storage를 handle 하는지 알 필요가 있다.

## Data Structures That Power Your Database

대부분의 DB는 concurrency control이나 disk space 등 여러 복잡한 문제들이 있긴 하지만 내부에서는 기본적으로 `log`를 사용하고 있다. 

> log란 보통 application log를 떠올리기 마련이지만, 사실 append-only sequence of recrds를 뜻하는 말이다. 항상 human-readable 할 필요는 없고 어떤 하나의 프로그램을 위한 방식으로 작성될 수 있다.

또 data를 효율적으로 찾기 위해 `index`라는 key를 사용한다. index는 additional metadata에 관리되며 어디에 data를 저장할지에 대한 지표로 사용된다. index를 추가하거나 제거하는 일은 실제 data에는 영향을 주지 않는다. 하지만 이런 additional structure를 사용하면 write 작업의 overhead가 증가하게 된다. 그러므로 잘 골라진 index들은 Read 속도를 증가시키지만 너무 많은 index는 write 속도를 감소시킨다.

### Hash Indexes

#### In-memory Hash Map Index

Append Only File Data Storage에서 사용할 수 있는 가장 간단한 indexing strategy는 In-memory Hash Map Index이다. 모든 key의 offset을 Hash Map에 저장하는 방법으로 새로운 key가 저장될 때마다 Hash Map에 offset을 저장하는 방식이다.  새로운 Data 추가보다는 Value에 대한 update가 많을 때 유용한 indexing 방식이다. 

Append Only이기 때문에 Log로 인해 disk space가 꽉찰 수 있다는 점을 주의해야 한다. 이를 해결하기 위해 compaction을 사용한다. Compaction이란 하나의 Data file segment에 있는 중복된 key들을 마지막으로 update된 값만 남기고 다 삭제하는 것을 의미한다. 

```text
# before compaction
foo: 1, bar: 2, baz:3, bar: 10, foo: 12, foo: 1, baz: 5, bar: 4, foo: 10

# after compaction
foo: 10, bar: 4, baz: 5
```

#### Hash Map Index 실제 구현 시에 고려해야 할 사항

* File format 
  * binary가 빠르고 사용하기 쉽다.
* Deleting records
  * 만약 record 삭제를 하고 싶다면, special deletion record를 data file에 넣어야한다.
  * tombstone이라고 불리기도 한다.
  * Segment가 merge\(compact\)될 때 tombstone을 확인하고 이전 값들은 모두 무시하게 된다.
* Crash Recovery
  * DB가 재시작되면 메모리에 있던 Hash Map은 모두 사라지게 된다. index를 다시 생성하기 위해 모든 Data Segment를 확인해야 하는데 이는 너무 오래 걸리는 작업이 될 수 있다.
  * 이를 극복하기 위해 hash map의 snapshot을 따로 저장해두기도 한다.
* Partially written records
  * DB는 언제든 Crash가 발생할 수 있기 때문에 broken record를 감지할 수 있어야 한다. 데이터 파일에 checksum을 같이 저장하여 broken record를 감지하고 무시하기도 한다.
* Concurrency Control
  * write 작업은 순차적으로 일어나야 하기 때문에 writer thread를 하나만 두고 사용한다. 이를 통해 Read 작업은 여러 thread에서 동시에 수행될 수 있게 한다. 

#### Append Only File의 장단점

* 장점
  * log appending과 Segment Merge가 동시에 일어날 수 있다.
  * Random writing보다 훨씬 빠르다.
  * Concurrency와 Crash Recovery 구현이 
  * Merging 작업을 통해 Data Fragment 를 방지할 수 있다.
* 단점
  * Hash Map은 메모리에 적재되기 때문에 key가 너무 많다면 적절하지 않다. 물론 디스크에 Hash Map을 저장할 수 있지만 performance가 잘 나오지 않는다. 
  * Range Query를 사용하고자 하면 비효율적이다. key 하나 하나 따로 lookup 해야 한다.

### SSTables and LSM-Trees

이전 예시에서 Append만 하던 Data Storage 구조에서 조금만 변경하여 key-value pair를 key 값으로 Sorting해서 저장할 수 있다. 이러한 구조를 **SSTables \(Sorted String Table\)** 이라고 부른다. SSTables은 Hash Indexes 보다 훌륭한 여러가지 장점을 가지고 있다.

* Segment를 Merge할 때 훨씬 간단하고 효율적이다. Mergesort algorithm을 생각하면 된다. 이미 정렬되어있기 때문에 여러 Segment를 처음부터 하나씩 읽어가고 가장 나중에 쓰여진 key를 Merged Segment에 쓰면 된다.
* 모든 키의 index를 메모리에 다 저장할 필요가 없다. 이미 key는 정렬되어 있기 때문에 index 또한 정렬된 순서로 저장할 수 있다. index에 저장되어있지 않은 key의 값을 찾고자 한다면 그 key가 순서 상 어디에 위치해 있는지 찾고 인접한 key의 index를 따라가서 찾으면 된다.
* Records를 하나의 그룹으로 묶고 compress할 수 있다. in-memory index에는 이 compressed block의 시작점을 저장해두고 사용할 수 있다.

#### Constructing and maintaining SSTables

B-Tree로 Disk에 sorted structure을 유지할 수 있지만 다른 red-black tree나 AVL tree로 memory에 관리하는 것이 더 쉽다. 

Storage Engine을 아래와 같이 동작하도록 만들 수 있다.

* Write 요청이 들어오면, in-memory에 있는 balanced tree에 넣어준다. 이때 이 balanced tree를 memtable이라고 한다.
* 만약 이 memtable의 크기가 어떤 threshold를 넘어섰을 때는 SSTable file로  디스크에 써준다. 이미 key로 sorting되어있기 때문에 이 작업은 효율적으로 수행될 수 있다. 이때 이 SSTable은 Database의 가장 최근 segment가 되고 write 작업은 새로운 memtable에 진행한다.
* read 작업은, 먼저 memtable에서 찾아보고 그다음은 DB의 segment를 순서대로 찾아본다.
* Background 작업으로, segment file들을 merge / compaction은 계속 진행되어 중복되거나 삭제된 값들은 버려지게 된다.

이같은 동작은 하나의 문제점을 가지고 있다. DB Crash가 일어났을 때, 가장 최근 값들\(memtable에 있는\)을 잃게 된다. 이 문제점을 해결하기 위해 write 작업에 대해서는 디스크에 log를 즉시 작성한다. 물론 이 log는 정렬되어있지는 않지만 이 log는 crash 있을 때에만 사용되기 때문에 문제가 되지 않는다. 또 memtable이 SSTable로 작성될 때마다 이 log는 다 버려지고 다시 작성된다.

#### Making an LSM-tree out of SSTables

지금까지 설명했던 알고리즘은 LevelDB와 RocksDB와 같은 다른 application에 embedded 되도록 설계된 DB에서 사용중인 알고리즘이다. Cassandra와 HBase에서도 비슷한 Storage Engine이 사용된다.  
\(Cassandra, HBase는 SSTable과 memtable이 소개된 Google의 Bigtable 논문에 토대로 만들어졌다.\)

이 indexing structure는 처음엔 Log-Structured Merge-Tree라는 이름으로 소개되었었다. 그래서 이러한 Storage Engine을 LSM Storage Engine이라고 불리기도 한다.

ElasticSearch나 Solr에서 사용 중인 Lucene도 비슷한 방법을 사용한다. 물론 full-text index가 훨씬 복잡하지만 베이스는 비슷하다. word\(a term\)를 포함하는 Document ID의 List\(postings list\)를 key-value로 저장한다. Lucene에서는 이 key-value를 SSTable과 비슷하게 sorted files에 저장하고 이를 background에서 merge한다.

#### Performance optimization

* Bloom Filters
  * 만약 DB에 없는 key를 찾고자 한다면, 모든 Segments를 다 돌게 된다. 이를 막기 위해, Bloom Filter를 사용한다.
  * Bloom filter is a memory-efficient data structure for approximating the contents of a set.
  * Bloom filter는 key가 DB에 존재하는지 알려주어 쓸모없는 Disk IO를 줄일 수 있다.
* Order and Timing of Compact and Merge
  * 가장 일반적인 option은 size-tiered와 leveled가 있다.
  * LevelDB와 RocksDB는 leveled를 사용하고 HBase는 size-tiered, Cassandra는 두가지 option 모두 지원한다.
  * Size-tiered Compaction : 새로운 그리고 작은 SSTables가 오래되고 큰 SSTables에 merge된다.
  * Leveled Compaction : key range를 작은 SSTables로 쪼개고\(level\) 이전 Data들을 나눠진 level에 나눠넣는다. size-tiered보다 더 점진적으로 진행되고 더 적은 Disk를 사용한다.

LSM-Tree는 약간의 미묘함들이 있긴 하지만 기본 아이디어는 간단하고 효율적이다. Dataset이 허용가능한 Memory보다 커졌을 때도 충분히 잘 동작하고, 이미 Data가 정렬되어 저장됐기 때문에 range query도 효율적으로 수행할 수 있다. 또 Disk Write 작업이 순차적으로 수행되어 write throughput이 상당히 좋다.



