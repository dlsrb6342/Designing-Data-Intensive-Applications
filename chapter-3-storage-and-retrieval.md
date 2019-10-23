---
description: Part 1. Foundations of Data Systems
---

# CHAPTER 3 Storage and Retrieval

Chapter 2에서는 application developer로써 어떻게 저장하고\(data model\) 어떻게 읽어올 것인지\(query\)에 대해 공부했다. Chapter 3에서는 database의 입장에서 같은 고민에 대해 이야기할 것이다.

Application Developer로써 DB Engine을 직접 구현하지는 않을 것이지만 적절한 Engine을 고르고 사용하기 위해서는 DB가 어떻게 storage를 handle하는지 알 필요가 있다.

## Data Structures That Power Your Database

대부분의 DB는 concurrency control이나 disk space 등 여러 복잡한 문제들이 있긴 하지만 내부에서는 기본적으로 `log`를 사용하고 있다. 

> log란 보통 application log를 떠올리기 마련이지만, 사실 append-only sequence of recrds를 뜻하는 말이다. 항상 human-readable할 필요는 없고 어떤 하나의 프로그램을 위한 방식으로 작성될 수 있다.

또 data를 효율적으로 찾기 위해 `index`라는 key를 사용한다. index는 additional metadata에 관리되며 어디에 data를 저장할지에 대한 지표로 사용된다. index를 추가하거나 제거하는 일은 실제 data에는 영향을 주지 않는다. 하지만 이런 additional structure를 사용하면 write 작업의 overhead가 증가하게 된다. 그러므로 잘 골라진 index들은 Read 속도를 증가시키지만 너무 많은 index는 write 속도를 감소시킨다.

### Hash Indexes







