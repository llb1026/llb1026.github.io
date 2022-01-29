---
order: 18
layout: post
title: "Transaction과 Lock 기본내용 정리"
subtitle: "@Version을 써봅시다"
tag: Tech Notes
type: tech-notes
blog: true
text: true
author: JIYUN LEE
post-header: true
header-img: img/transaction-001.jpeg
carousels:
  - images: 
    - image: /blog/transaction-and-lock/img/transaction-001.jpeg
    - image: /blog/transaction-and-lock/img/transaction-002.jpeg
    - image: /blog/transaction-and-lock/img/transaction-003.jpeg
    - image: /blog/transaction-and-lock/img/transaction-004.jpeg
    - image: /blog/transaction-and-lock/img/transaction-005.jpeg
    - image: /blog/transaction-and-lock/img/transaction-006.jpeg
    - image: /blog/transaction-and-lock/img/transaction-007.jpeg
    - image: /blog/transaction-and-lock/img/transaction-008.jpeg
    - image: /blog/transaction-and-lock/img/transaction-009.jpeg
    - image: /blog/transaction-and-lock/img/transaction-010.jpeg
    - image: /blog/transaction-and-lock/img/transaction-011.jpeg
    - image: /blog/transaction-and-lock/img/transaction-012.jpeg

---

https://github.com/llb1026/naver-intern/wiki/%EC%8A%A4%ED%84%B0%EB%94%94-Transaction

# 16. 트랜잭션과 락, 2차 캐시

## 사전지식
트랜잭션은 Atomicity(*원자성*), Consistency(*일관성*), Isolation(*독립성*), Durability(*지속성*)을 보장해야 한다. [^1]

- 원자성: 하나의 트랜잭션 안에서 실행한 일련의 작업들은 전부 다 성공하거나 전부 다 실패해야 한다 → 트랜잭션 안의 작업들이 부분적으로 실행되다가 중단되지 않는 것을 보장한다는 의미로, 예를 들어 '이체'라는 하나의 트랜잭션 안에서 송금만 성공하고 입금은 실패할 수 없어야 한다
- 일관성: 트랜잭션이 실행을 성공적으로 완료하면 언제나 일관성 있는 DB 상태를 유지해야 한다 → DB에서 정한 무결성 제약 조건을 항상 만족해야 하며, 예를 들어 무결성 제약이 '모든 계좌는 잔고가 있어야 한다' 라면 이를 위반하는 트랜잭션은 중단된다
- 독립성: 동시에 실행되는 트랜잭션들이 서로의 연산 작업에 영향을 미치지 않도록 격리하는 것으로, 동시성과 관련된 성능 이슈로 인해 격리 수준을 별도로 설정해줄 수 있다 → 예를 들어 가장 엄격한 격리 수준(SERIALIZABLE)이 설정되어 있다면, '이체' 트랜잭션이 진행되는 동안 다른 트랜잭션들('잔고 조회', '출금' 등)은 잔고 데이터에 접근할 수 없어 '이체'가 끝날 때까지 계속 기다려야 한다 (동시성X)
- 지속성: 트랜잭션이 성공적으로 끝나면 그 결과가 영원히 반영되어야 한다 → 중간에 시스템에 장애가 발생해도 DB 로그에 남은 트랜잭션들을 가지고 와서 장애 발생 전으로 되돌릴 수 있다 (트랜잭션은 로그에 모든 내역이 남은 후여야 commit 상태라고 할 수 있다)

---

## 트랜잭션과 격리 수준
격리 수준(Isolation level)은 동시에 여러 트랜잭션이 처리될 때 특정 트랜잭션이 다른 트랜잭션에서 변경/조회하고 있는 데이터를 볼 수 있도록 허용할지 말지를 결정하는 것이다.
일반적으로 격리 수준과 동시성은 반비례 관계이다.

1. ==READ UNCOMMITED==
	- 하나의 트랜잭션에서 작업중인 변경 내용이 commit/rollback 되지 않은 상태에서도 다른 트랜잭션에서 보여진다 → DIRTY READ
	- 데이터 정합성에 문제가 많아 RDBMS 표준에서는 격리 수준으로 인정하지 않고 있다
2. ==READ COMMITTED==
	- 하나의 트랜잭션에서 작업중인 변경 내용이 commit/rollback 완료된 경우에만 다른 트랜잭션에서 보여진다
	- 아직 commit/rollback 전이라면, 다른 트랜잭션은 undo[^REDO 영역에는 수행했던 모든 작업들이 순차적으로 기록되고, UNDO 영역에는 각 데이터 변경 작업이 일어나기 이전의 데이터들이 담긴다. DB에 장애가 발생했을 때에는 REDO를 사용해 복구하고, 작업 단위로 롤백하고자 할 때에는 UNDO를 사용한다.] 영역에 백업된 기존 값을 참조한다
	- Oracle DBMS에서 디폴트 격리 수준이다 (Shared Lock[^Exclusive Lock(= Write Lock)은 하나의 트랜잭션에서 데이터 변경 작업을 수행할 시 트랜잭션이 완료될 때까지 다른 트랜잭션이 해당 데이터를 읽고 쓰지 못하게 하고, Shared Lock(= Read Lock)은 트랜잭션이 완료될 때까지 다른 트랜잭션이 해당 데이터를 읽을 수만 있게 한다. 하나의 데이터에 Exclusive Lock이 걸리면 다른 트랜잭션이 해당 데이터에 또다른 Exclusive Lock이나 Shared Lock을 걸 수 없으나, 데이터에 Shared Lock이 걸리면 다른 트잭션은 해당 데이터에 또다른 Shared Lock을 걸 수 있다. 그러나 Shared Lock이 걸린 데이터에 Exclusive Lock을 거는 것은 불가능하다.] 사용)
	- 하나의 트랜잭션에서 데이터를 조회중인데 다른 트랜잭션이 해당 데이터를 변경하고 commit하면 다시 해당 데이터를 조회했을 때 변경된 값이 조회되므로 데이터 정합성이 깨진다. 예를 들어, 앱에서 이체를 위해 계좌 잔고를 조회하는 도중 ATM으로 돈을 전부 이체했는데 앱에서는 여전히 이체가 가능한 문제가 발생한다 → NON-REPEATABLE READ[^REPEATABLE READ는 하나의 트랜잭션 안에서 동일한 SELECT문을 실행했을 때 항상 같은 결과를 가져오는 경우이다.]
	- NON-REPEATABLE READ는 하나의 트랜잭션에서 동일한 쿼리를 두 번 이상 실행했을 때 다른 결과값이 조회되는 경우로, DIRTY READ에 비해 비교적 발생할 확률이 적다
3. ==REPEATABLE READ==
	- READ COMMITTED와 달리, commit한 데이터만 접근할 수 있어 하나의 트랜잭션에서 한 번 조회한 데이터에 대해 여러 번 같은 조회 쿼리를 날려도 동일한 결과값이 나온다 → NON-REPEATABLE READ 발생하지 않음
	- READ COMMITTED와의 또다른 차이는 UNDO 영역에 기록된 이전 변경내역을 몇번째 이전 버전까지 찾아 들어가는지에 있음 → 데이터를 변경중인 트랜잭션 안에서 변경이 많아지고 종료되지 않으면 UNDO 영역이 무한정으로 커질 수 있고, 이로 인해 성능이 떨어질 수 있다
	- MySQL InnoDB에서 디폴트 격리 수준이다
	- Binary Log를 가진 MySQL DB에서는 최소 REPEATABLE READ 이상의 격리 수준을 사용해야 한다
	- `SELECT ~ FOR UPDATE` 쿼리의 경우, 다른 트랜잭션에서 commit 하기 전에 수행한 변경 작업에 의해 데이터들이 보였다가 안보였다가 할 수 있다 → PHANTOM READ
	- PHANTOM READ는 UNDO 영역에 Lock을 걸 수 없으므로 트랜잭션 시작 전의 데이터가 아니라 UNDO 영역에 기록된 변경된 레코드 버전을 읽어오기 때문에 발생한다
4. ==SERIALIZABLE==
	- 가장 엄격한 트랜잭션 격리 수준으로, 데이터 변경뿐만 아니라 조회 작업에도 Shared Lock이 필요하며 다른 트랜잭션이 해당 데이터를 변경할 수 없다
	- 한 트랜잭션에서 읽고 쓰는 데이터를 다른 트랜잭션에서는 절대 접근할 수 없다 → 동시성 처리 성능 저하

격리 수준에 따라 발생하는 문제점들들을 다시 정리하면 다음과 같다.

- ==DIRTY READ==: 하나의 트랜잭션에서 변경된 데이터가 commit되기 이전에도 다른 트랜잭션에서 변경된 데이터에 접근할 수 있는 문제
- ==NON-REPEATABLE READ==: 하나의 트랜잭션에서 변경중인 데이터에 대해 Shared Lock을 획득한 다른 트랜잭션이 같은 조회 쿼리를 두 번 수행했는데 그 사이에 해당 데이터가 변경되어 처음 조회 결과값과 다른 결과값이 나올 수 있는 문제
- ==PHANTOM READ==: 하나의 트랜잭션에서 변경중인 데이터에 대해 Shared Lock을 획득한 다른 트랜잭션이 같은 쿼리를 두 번 수행했는데 그 사이에 신규로 삽입된 데이터에 의해 첫번째에서는 없던 데이터가 두번째에서 조회되는 문제

| 격리 수준 | DIRTY READ | NON-REPEATABLE READ | PHANTOM READ |
|:--|:-:|:-:| :-: |
| READ UNCOMMITTED | O | O | O |
| READ COMMITTED | | O | O |
| REPEATABLE READ | | | O |
| SERIALIZABLE | | | |

---

## 낙관적 Lock과 비관적 Lock
- 낙관적 Lock
	- 트랜잭션이 충돌 없이 수행될 것이라 가정하고, DB에서 제공하는 Lock이 아니라 JPA(어플리케이션)에서 제공하는 버전 관리 기능을 사용한다
	- 예를 들어, 앱에서 이체하기 위해 잔고를 조회하는 중에도 ATM에서 전액 출금을 할 수 있으며, 출금이 진행된 이후에도 앱에서 동일한 트랜잭션으로 이체를 시도할 경우 어플리케이션 단에서 Exception을 발생시킬 수 있다
- 비관적 Lock
	- 트랜잭션에 충돌이 일어날 것을 가정하고 DB단에서 우선 Lock을 거는 방식으로, 대표적으로 `SELECT FOR UPDATE` 구문이 있다
	- 그러나 SECOND LOST UPDATES 문제가 발생할 수 있다
	- 예를 들어, 두 사람이 동시에 하나의 글을 수정할 때 마지막으로 수정완료 버튼을 누른 사람의 수정사항만 남고 먼저 수정완료한 사람의 데이터는 사라진다
	- SECOND LOST UPDATES 문제는 DB 트랜잭션의 범위를 넘어서므로, 다음과 같이 3가지 선택방법으로 해결해야 한다
		1. 마지막 commit만 인정 (default)
		2. 최초 commit만 인정
		3. 충돌하는 갱신 내용 병합

---

## @Version 소개
Entity에 `@Version` 어노테이션을 붙이면 Entity를 수정할 때마다 버전이 하나씩 자동으로 증가한다. 그래서 Entity를 수정할 때의 버전과 수정완료할 때의 버전이 다르면 Exception이 발생한다 (디폴트로 최초 commit만 인정)

`@Version`으로 추가한 버전 관리 필드는 JPA가 직접 관리하며, 임의로 수정할 수 없다. 

버전 값을 강제로 증가시키거나, 버전 관리 필드를 무시하는 Bulk 연산의 경우 JPA에서 제공하는 Lock을 사용해야 한다.

JPA Lock의 종류는 아래와 같다.

| Lock Mode | Type | Description |
|:--|:--|:--|
| 낙관적 Lock | OPTIMISTIC | 낙관적 Lock을 사용 |
| 낙관적 Lock | OPTIMISTIC_FORCE_INCREMENT | 낙관적 Lock을 사용하며 버전 정보를 강제로 증가 |
| 비관적 Lock | PESSIMISTIC_READ | 비관적 Lock, Shared Lock을 사용 |
| 비관적 Lock | PESSIMISTIC_WRITE | 비관적 Lock, Exclusive Lock을 사용 |
| 비관적 Lock | PESSIMISTIC_FORCE_INCREMENT | 비관적 Lock을 사용하며 버전 정보를 강제로 증가 |
| 기타 | NONE | Lock을 걸지 않는다 |
| 기타 | READ | OPTIMISTIC과 동일 |
| 기타 | WRITE | OPTIMISTIC_FORCE_INCREMENT와 동일 |

> JPA를 사용할 때에는 READ COMMITTED 격리 수준 + 낙관적 버전 관리를 적용하는 것이 일반적이다

---

{% include carousel.html height="55" unit="%" duration="20" number="1" %}

---

## Appendix
- [위키백과 ACID](https://ko.wikipedia.org/wiki/ACID)
- [MySQL Isolation Level 종류 및 특징](https://hyunki1019.tistory.com/111)
- [트랜잭션의 격리 수준이란](https://nesoy.github.io/articles/2019-05/Database-Transaction-isolation)

---

[^1]: 풋노트 샘플