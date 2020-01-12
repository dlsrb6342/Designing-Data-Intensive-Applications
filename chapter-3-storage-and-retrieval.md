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

### B-Trees

log-structured index는 쓸만하지만 가장 흔한 index type은 아니다. 가장 많이 사용되는 indexing structure는 B-Tree 이다.

SSTables 처럼, B-Tree도 key-value lookup과 range query를 위해 key-value pair를 key 값으로 정렬해둔다. 하지만 이 부분을 제외한 모든 것들이 SSTable과 다르다.

log-structured index는 Database를 가변적인 size의 segments로 나눈다. 하지만 B-tree는 고정된 size의 **Blocks or Pages** 로 나눈다. 보통 4KB의 크기이며 한번에 한 page만 읽고 쓴다. 이러한 Design은 하드웨어와 조금 더 비슷한 면이 있다. \(Disk가 fixed-size blocks를 가지는 것처럼\)

각각의 page들은 address\(location\)으로 식별된다. pointer 처럼, 한 page가 다른 page를 가리키고 있다. 이러한 page references 를 사용하면 page들은 tree 형태로 표현될 수 있다. 

![](.gitbook/assets/image%20%286%29.png)

page는 여러개의 key들과 child page로 향하는 reference들을 가지고 있다. 무언가 Database에서 찾고자 할때, B-tree의 root인 어떤 한 page에서 시작된다. page의 child들은 각각 연속된 key의 range를 담당한다. B-tree의 leaf node인 page에는 각각의 key들만 가지고 있다. 

B-tree의 page가 가지는 reference의 수를 **branching factor** 라고 한다. 보통 몇 백개 정도 된다.

* Update value 
  * leaf page까지 찾아가서 value를 바꿔주고 disk에 써준다. 
* Add new key
  * 어느 범위에 속하는지 따라 내려가서 해당 page에 추가해준다.
  * 만약 page가 꽉 찼다면, page를 2개로 나누고 parent page에 reference를 update해준다.

대부분의 Database는 4 level의 B-Tree면 충분하기 때문에 page reference를 그렇게 많이 따라 내려가지는 않는다. \(4KB page, 500 branching factor, four-level B-Tree는 256TB를 저장할 수 있다.\)

#### Making B-trees reliable

B-Tree의 write 작업은 disk의 page에 있는 data를 overwrite 한다. 이 작업은 page의 reference를 변하지 않게 유지해준다. 이런 overwriting 작업은 실제 hardware operation이라고 생각할 수 있다. HDD에서는, spinning platter의 맞는 위치를 찾고 그곳에 새로운 data를 쓴다. SSD에서는 지우고 다시 쓰기 때문에 조금 더 복잡하다.

게다가, 만약 해당하는 page가 꽉 찼다면 여러 page를 고쳐야 할 수도 있다. 이러한 작업은 write 작업 중간에 database crash가 발생할 수 있기 때문에 굉장히 위험한 작업이다. 

Database가 crash에 회복할 수 있도록 만들기 위해, 보통 B-tree 구현체에 additional data structure를 포함시킨다. \(**WAL** : write-ahead log, redo log라고도 한다.\) WAL은 append-only file로 B-Tree의 모든 변경사항을 page에 적용하기 전에 작성되어지는 곳이다. Database가 crash 이후에 다시 실행되었을 때, 이 로그를 통해 다시 B-Tree를 consistent하게 만든다.

여러 thread가 B-tree에 접근하고 서로 같은 page를 수정하려고 할 수 있기 때문에 concurrency control이 중요하다. 이는 보통 **latches\(lightweight locks\)** 로 해결한다. 

### Comparing B-Trees and LSM-Trees

LSM-Tree는 write 작업에서 조금 더 빠르고, B-Tree는 read 작업에서 더 빠르다. 하지만 이런 benchmark는 종종 결정적이지 않으며 워크로드의 세부 사항에 민감하다. 따라서 각자의 환경, workload로 system을 test 해야 한다. 이 섹션에서는 Storage Engine의 성능을 측정할 때, 고려해야할 몇가지 사항에 대해 다룰 것이다.

#### Advantages of LSM-Trees

B-Tree는 하나의 write 작업에서 적어도 2개의 write\(WAL에 쓰는 작업 + 실제 page에 쓰는 작업\)가 일어난다. 또한 작은 bytes의 변화에도 page의 전체를 다시 써야하는 overhead도 있다.

하지만 LSM-Tree도 compaction 때문에 data를 여러번 rewrite 하게 된다. 이런 현상을 write amplification이라고 부른다. 제한된 횟수만큼만 overwrite할 수 있는 SSD에서 문제가 된다. write 작업이 많은 application에게는 write amplification이 중요한 사항이 될 수 있다. 

LSM-Tree는 보통 B-Tree보다 write throughput 이 높게 나온다. 이것은 LSM-Tree가 낮은 write amplification을 가지기 때문이기도 하고 SSTable file들을 sequential하게 쓰기 때문이기도 하다. Sequentially Write는 magnetic hard drive일 때 특히 중요하게 작용한다.

LSM-Tree는 더 compress되기 때문에 B-Tree보다 적은 수의 file을 가진다. 또한 B-Tree는 fragmentation이 발생할 수 있기 때문에 disk space를 낭비하기도 한다.

#### Downsides of LSM-Trees

LSM-Tree의 compaction process가 read / write performance에 방해가 될 수 있다. disk 자원은 한정되어있기 때문에, disk가 compaction process를 끝내기를 기다려야 할 수도 있다.

Disk의 한정된 write throughput이 initial write\(logging and flushing memtable to disk\)와 compaction 사이에 공유되어야 한다. database가 커지면 커질수록 compaction에 필요한 disk bandwidth가 증가하게 된다. 

LSM-Tree가 여러 다른 segment들 안에 같은 key의 다양한 복사본을 들고 있을 수 있는 반면에, B-Tree는 하나의 key는 딱 한 곳에만 저장되어 있다. 이는 transaction 관리에 큰 장점이 있다. 보통 key의 한 범위에 lock을 구현해서 관리하는데 B-Tree에서는 tree에 이 lock을 붙여주면 된다.

### Other Indexing Structures

Secondary Index는 굉장히 흔하게 사용된다. RDB에서는, CREATE INDEX command를 통해 join을 효율적으로 할 수 있도록 index를 생성할 수 있다. Secondary Index는 key-value index로 구성되는데 지금까지 봤던 index와 가장 다른 점은 서로 다른 row가 같은 key 값을 가질 수 있다는 점이다. 

#### Storing values within the index

index의 value는 실제 row 혹은 row가 저장된 곳의 참조값 이렇게 2가지 값이 될 수 있다. 참조값일 경우, heap file의 어딘가를 나타내고 특정한 순서없이 저장되어 있다. heap file의 위치를 저장하는 방법이 secondary index가 있을때 데이터의 중복을 피할 수 있기 때문에 실제 row를 저장하는 방법보다 더 common하게 사용된다. 

key는 변하지 않고 값만 바꾸는 경우엔 heap file 위치 저장이 효율적이다. 새로운 값이 이전 값보다 작은 경우에는 그 값을 바로 덮어쓸 수 있다. 만약 새로운 값이 더 클 경우에는 약간 복잡해지는데 충분한 공간이 있는 heap으로 옮겨야 하기 때문에 모든 index를 새로운 heap을 가리키도록 업데이트해야 한다.

종종, index에서 heap, row까지의 hop이 read 작업 성능에 영향을 줄 수 있다. 이럴 땐 clustered index라는 것을 사용한다. 예를 들어 MySQL의 InnoDB storage engine은 primary key의 table는 항상 clustered index이고 secondary index는 heap file의 위치를 보기 보다는 primary key를 참조한다. 

clustered index\(index에 row data를 저장하는 것\)과 nonclustered index\(index에 data의 reference만 저장하는 것\)의 절충안을 `covering index` 혹은 `index with included columns` 라고 한다. 이것은 table의 특정 column만 index에 저장하고 index만으로도 특정 query에 답을 줄 수 있다. 

#### Multi-column indexes

지금까지 이야기한 key-value map 형태의 index는 테이블의 여러 column으로 query를 할때에는 부족한 점이 있다. 

가장 흔한 type의 multi-column index은 concatenated index이다. Concatenated index란 여러 field를 하나의 key로 합쳐서 표현하는 index이다. 오래된 방식이고 만약 하나의 field로 query하고 싶을때는 사용할 수 없는 index이다.

Multi-dimensional index가 여러 column을 한번에 조회하는 가장 일반적인 방법이다. 공간적 데이터를 다룰때 특히 중요하다. latitude의 범위와 longitude의 범위로 조회할 경우, Multi-dimensional index를 활용할 수 있다.

#### Full-text search and fuzzy indexes

지금까지의 index는 정확한 data와 정확한 key 값으로 조회할 경우만을 가정하고 생각했다. 하지만 불명확한\(fuzzy\) query를 할 때는 아주 다른 기술이 사용된다.

Full-text search engine은 동의어도 포함해야 하고 문법적 다양성도 고려해야 하고 여러가지 언어적인 고려사항이 많다. Lucene은 이를 edit distance로 조회가 가능하게 해준다.

#### Keeping everything in memory

RAM이 점점 싸지면서 cost-per-gigabyte는 점점 고려대상에서 제외되고 있다. 많은 dataset들이 그렇게 크지 않기 때문에 여러 장비에 분산하여 data 전부를 memory에 저장할만 해졌다. 이로 인해 in-memory database들이 발전할 수 있었다.

캐시로만 사용하기 위한 in-memory database도 있지만 batter-powered RAM과 같은 특수한 hardware를 사용하거나 disk에 log를 남기는 등 durability를 목적으로 하는 in-memory database도 있다. 

VoltDB, MemSQL, and Oracle TimesTen와 같은 Relational Model의 in-memory database들도 있다. 이들은 on-disk data관리의 overhead가 없어 큰 성능 향상이 있다. durability를 신경쓴 key-value store인 RAMCloud도 있고 Redis나 Couchbase같이 durability가 약한 key-value store도 있다.

in-memory database의 performance 이점은 disk에서 읽어올 필요가 없기 때문에 발생하는 것은 아니다.  disk-based engine이더라도, 큰 memory를 가지고 있다면 최근 사용된 disk block을 전부 memory에 저장하고 읽어올 수 있다. 그러나 in-memory database는 in-memory data structure를 disk에 write할 형태로 encoding하는 overhead가 없기 때문에 더 빠를 수 있는 것이다.

Redis는 priority queue나 set같은 다양한 data structure를 제공하는데 이는 모든 데이터를 memory에 저장하기 때문에 가능한 이점이다. 이러한 data structure는 disk based index에는 구현하기 어렵기 때문에 in-memory database만 제공할 수 있는 것이다.

최근 연구 결과, in-memory database는 사용 가능한 memory의 양보다 더 큰 dataset를 제공할 수 있다고 한다. Operating System이 virtual memory를 다루는 것과 file들을 swap하는 것처럼 가장 예전에 접근한 data는 memory에서 disk로 내리고 나중에 접근할 때에 disk에서 꺼내오는 방식으로 구현이 가능하다. 이러한 방식을 anti-caching 방식이라고 한다. 데이터베이스는 전체 메모리 페이지가 아닌 개별 레코드 단위로 작업 할 수 있으므로 OS보다 메모리를 보다 효율적으로 관리 할 수 있다. 

