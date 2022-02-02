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
header-img: img/01-1.png
next-link: "../how-transactional-works/"
prev-link: "../java-serializable/"
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


## 1. 사전지식

트랜잭션은 Atomicity(*원자성*), Consistency(*일관성*), Isolation(*독립성*), Durability(*지속성*)을 보장해야 한다.

- *원자성*: 하나의 트랜잭션 안에서 실행한 일련의 작업들은 전부 다 성공하거나 전부 다 실패해야 한다 → 트랜잭션 안의 작업들이 부분적으로 실행되다가 중단되지 않는 것을 보장한다는 의미로, 예를 들어 '이체'라는 하나의 트랜잭션 안에서 송금만 성공하고 입금은 실패할 수 없어야 한다
- *일관성*: 트랜잭션이 실행을 성공적으로 완료하면 언제나 일관성 있는 DB 상태를 유지해야 한다 → DB에서 정한 무결성 제약 조건을 항상 만족해야 하며, 예를 들어 무결성 제약이 '모든 계좌는 잔고가 있어야 한다' 라면 이를 위반하는 트랜잭션은 중단된다
- *독립성*: 동시에 실행되는 트랜잭션들이 서로의 연산 작업에 영향을 미치지 않도록 격리하는 것으로, 동시성과 관련된 성능 이슈로 인해 격리 수준을 별도로 설정해줄 수 있다 → 예를 들어 가장 엄격한 격리 수준(SERIALIZABLE)이 설정되어 있다면, '이체' 트랜잭션이 진행되는 동안 다른 트랜잭션들('잔고 조회', '출금' 등)은 잔고 데이터에 접근할 수 없어 '이체'가 끝날 때까지 계속 기다려야 한다 (동시성X)
- *지속성*: 트랜잭션이 성공적으로 끝나면 그 결과가 영원히 반영되어야 한다 → 중간에 시스템에 장애가 발생해도 DB 로그에 남은 트랜잭션들을 가지고 와서 장애 발생 전으로 되돌릴 수 있다 (트랜잭션은 로그에 모든 내역이 남은 후여야 commit 상태라고 할 수 있다)

### 1-2. 트랜잭션의 4가지 속성

#### 1-2-1. Propagation

동시에 여러 트랜잭션이 처리될 때 특정 트랜잭션이 다른 트랜잭션에서 변경/조회하고 있는 데이터를 볼 수 있도록 허용할지 말지를 결정하는 것이다. 일반적으로 격리 수준과 동시성은 반비례 관계이다.

- `REQUIRED`: 디폴트 속성이다. 이미 시작된 트랜잭션이 있으면 참여하고, 없으면 새로 시작한다.
- `SUPPORTS`: 이미 시작된 트랜잭션이 있으면 참여하고, 없으면 트랜잭션 없이 진행한다. 트랜잭션이 없긴 하지만 해당 경계 안에서 커넥션이나 Hibernate 세션을 공유할 수 있다.
- `MANDATORY`: 이미 시작된 트랜잭션이 있으면 참여하고, 없으면 새로 시작하는 대신 예외를 발생시킨다. 혼자서는 독립적으로 트랜잭션을 진행하면 안 되는 경우에 사용한다.
- `REQUIRED_NEW`: 항상 새로운 트랜잭션을 시작한다. 이미 진행중인 트랜잭션이 있으면 트랜잭션을 잠시 보류시킨다.
- `NOT_SUPPORTED`: 트랜잭션을 사용하지 않게 한다. 이미 진행중인 트랜잭션이 있으면 보류시킨다.
- `NEVER`: 트랜잭션을 사용하지 않도록 강제한다. 이미 진행중인 트랜잭션도 존재하면 안되고, 존재할 경우 예외를 발생시킨다.
- `NESTED`: 이미 진행중인 트랜잭션이 있으면 중첩 트랜잭션을 시작한다. 독립적인 트랜잭션을 만드는 REQUIRED_NEW랑은 다르다. 중복된 트랜잭션은 먼저 시작한 부모 트랜잭션의 커밋과 롤백에는 영향을 받지만, 자신의 커밋과 롤백은 부모 트랜잭션에 영향을 주지 않는다. 쉽게 예를 들어서 작업 안에 로깅 트랜잭션을 중첩하는 경우가 있다. 로깅 하나 실패한다고 해서 작업 전체를 취소할 수는 없는 그런 경우.

#### 1-2-2. Isolation

동시에 여러 트랜잭션이 진행될 때 트랜잭션의 작업 결과를 여타 트랜잭션에게 어떻게 노출할 것인지를 결정하는 기준이 된다.

- `DEFAULT`
	- 사용하는 DB 디폴트 설정을 따른다.
	- 간혹 일부 DB는 디폴트값이 다른 경우가 있으니 드라이버와 DB문서를 잘 확인해서 디폴트 isolation 수준을 확인해야 한다.
- `READ_UNCOMMITTED`
	- 가장 낮은 isolation 수준이다.
	- 하나의 트랜잭션에서 작업중인 변경 내용이 commit/rollback 되지 않은 상태에서도 다른 트랜잭션에서 보여진다. → **DIRTY READ**
	- 데이터 정합성에 문제가 많아 RDBMS 표준에서는 격리 수준으로 인정하지 않고 있다.
	- 그러나 속도가 가장 빠르기 때문에 데이터의 일관성이 좀 떨어지더라도 성능을 극대화할 때 의도적으로 사용하기도 한다.
- `READ_COMMITTED`
	- 가장 많이 사용되는 isolation 수준이다.
	- `READ_UNCOMMITTED`와 달리 다른 트랜잭션이 커밋하지 않은 정보는 읽을 수 없다. 대신 하나의 트랜잭션이 읽은 row를 다른 트랜잭션이 수행할 수 있다. 때문에 처음 트랜잭션이 같은 row를 읽어도 다른 내용이 발견될 수 있다.
	- 하나의 트랜잭션에서 데이터를 조회중인데 다른 트랜잭션이 해당 데이터를 변경하고 commit하면 다시 해당 데이터를 조회했을 때 변경된 값이 조회되므로 데이터 정합성이 깨진다. 예를 들어, 앱에서 이체를 위해 계좌 잔고를 조회하는 도중 ATM으로 돈을 전부 이체했는데 앱에서는 여전히 이체가 가능한 문제가 발생한다. → **NON-REPEATABLE READ** (하나의 트랜잭션에서 동일한 쿼리를 두 번 이상 실행했을 때 다른 결과값이 조회되는 경우로, DIRTY READ에 비해 비교적 발생할 확률이 적다)
	- Oracle DBMS에서 디폴트 격리 수준이다.
- `REPEATABLE_READ`
	- 하나의 트랜잭션이 읽은 row를 다른 트랜잭션이 수정하는 것을 막아준다. `READ COMMITTED`와 달리, commit한 데이터만 접근할 수 있어 하나의 트랜잭션에서 한 번 조회한 데이터에 대해 여러 번 같은 조회 쿼리를 날려도 동일한 결과값이 나온다 → NON-REPEATABLE READ 발생하지 않음
	- 그렇지만 새로운 row를 추가하는 것은 제한하지 않는다. 따라서 `SELECT ~ FOR UPDATE` 쿼리의 경우, 다른 트랜잭션에서 commit 하기 전에 수행한 변경 작업에 의해 데이터들이 보였다가 안보였다가 할 수 있다. → **PHANTOM READ**
	- MySQL InnoDB에서 디폴트 격리 수준이다.
	- Binary Log를 가진 MySQL DB에서는 최소 REPEATABLE READ 이상의 격리 수준을 사용해야 한다.
- `SERIALIZABLE`
	- 가장 강력한 isolation 수준이다. 
	- 이름 그대로 트랜잭션을 순차적으로 진행시켜 주기 때문에 여러 트랜잭션이 동시에 같은 테이블의 정보에 접근하지 못한다. 가장 안전하지만 성능도 가장 떨어지므로 엄청나게 안전해야 하는 작업이 아니면 잘 안 쓴다.

격리 수준에 따라 발생하는 문제점들들을 다시 정리하면 다음과 같다.

- **DIRTY READ**: 하나의 트랜잭션에서 변경된 데이터가 commit되기 이전에도 다른 트랜잭션에서 변경된 데이터에 접근할 수 있는 문제
- **NON-REPEATABLE READ**: 하나의 트랜잭션에서 변경중인 데이터에 대해 Shared Lock을 획득한 다른 트랜잭션이 같은 조회 쿼리를 두 번 수행했는데 그 사이에 해당 데이터가 변경되어 처음 조회 결과값과 다른 결과값이 나올 수 있는 문제
- **PHANTOM READ**: 하나의 트랜잭션에서 변경중인 데이터에 대해 Shared Lock을 획득한 다른 트랜잭션이 같은 쿼리를 두 번 수행했는데 그 사이에 신규로 삽입된 데이터에 의해 첫번째에서는 없던 데이터가 두번째에서 조회되는 문제

<table class="uk-table-small uk-table style-2 uk-table-striped uk-text-center">
    <thead>
        <tr>
            <th style="text-align: center;"> \ </th>
            <th style="text-align: center;">DIRTY READ</th>
            <th style="text-align: center;">NON-REPEATABLE READ</th>
            <th style="text-align: center;">PHANTOM READ</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td><em>READ_UNCOMMITTED</em></td>
            <td>O</td>
            <td>O</td>
            <td>O</td>
        </tr>
        <tr>
            <td><em>READ_COMMITTED</em></td>
            <td>X</td>
            <td>O</td>
            <td>O</td>
        </tr>
        <tr>
            <td><em>REPEATABLE_READ</em></td>
            <td>X</td>
            <td>X</td>
            <td>O</td>
        </tr>
        <tr>
            <td><em>SERIALIZABLE</em></td>
            <td>X</td>
            <td>X</td>
            <td>X</td>
        </tr>
    </tbody>
</table>

#### 1-2-3. Timeout

트랜잭션에 초 단위로 제한시간을 걸 수 있다. 디폴트는 트랜잭션 시스템의 제한시간을 따르는 것.

#### 1-2-4. Read-Only

트랜잭션을 읽기 전용으로 설정할 수 있다. 성능을 최적화하기 위해 사용할 수도 있고, 특정 트랜잭션 작업 안에서 쓰기 작업이 일어나는 것을 의도적으로 방지하기 위해 사용할 수도 있다.

트랜잭션을 준비하면서 read-only 속성이 트랜잭션 매니저에게 전달되고, 이에 따라 매니저가 작업을 수행하는데 일부 매니저의 경우 이 속성을 무시하고 쓰기 작업을 허용할 수도 있기 때문에 주의해야 한다.

일반적으로 read-only 트랜잭션이 시작된 이후 INSERT, UPDATE, DELETE같은 쓰기 작업이 진행되면 예외가 발생한다.

---

## 2. 낙관적 Lock과 비관적 Lock

- **낙관적 Lock**
	- 트랜잭션이 충돌 없이 수행될 것이라 가정하고, DB에서 제공하는 Lock이 아니라 JPA(어플리케이션)에서 제공하는 버전 관리 기능을 사용한다
	- 예를 들어, 앱에서 이체하기 위해 잔고를 조회하는 중에도 ATM에서 전액 출금을 할 수 있으며, 출금이 진행된 이후에도 앱에서 동일한 트랜잭션으로 이체를 시도할 경우 어플리케이션 단에서 Exception을 발생시킬 수 있다
- **비관적 Lock**
	- 트랜잭션에 충돌이 일어날 것을 가정하고 DB단에서 우선 Lock을 거는 방식으로, 대표적으로 `SELECT FOR UPDATE` 구문이 있다
	- 그러나 SECOND LOST UPDATES 문제가 발생할 수 있다
	- 예를 들어, 두 사람이 동시에 하나의 글을 수정할 때 마지막으로 수정완료 버튼을 누른 사람의 수정사항만 남고 먼저 수정완료한 사람의 데이터는 사라진다
	- SECOND LOST UPDATES 문제는 DB 트랜잭션의 범위를 넘어서므로, 다음과 같이 3가지 선택방법으로 해결해야 한다
		1. 마지막 commit만 인정 (default)
		2. 최초 commit만 인정
		3. 충돌하는 갱신 내용 병합

---

## 3. @Version 소개

Entity에 `@Version` 어노테이션을 붙이면 Entity를 수정할 때마다 버전이 하나씩 자동으로 증가한다. 그래서 Entity를 수정할 때의 버전과 수정완료할 때의 버전이 다르면 Exception이 발생한다 (디폴트로 최초 commit만 인정)

`@Version`으로 추가한 버전 관리 필드는 JPA가 직접 관리하며, 임의로 수정할 수 없다. 

### 3-1. 적용 방법

정말 간단하다. DB에 버전관리를 할 테이블마다 `bigint()` 타입의 컬럼을 만들고, Entity에서 해당 필드 선언 위에 `@Version`을 붙여주면 끝이다.

```sql
CREATE TABLE temp (
	id	bigint auto_increment primary key,
	-- ...중략
	entity_version bigint default 0 not null,
);
```

```java
@Entity
@Getter
@Setter
@ToString
@NoArgsConstructor
@AllArgsConstructor
@Table(name = "temp")
public class TempEntity {

	@Id
	private Long id;
	
	// ...중략
	 
	@Version
	private Long entity_version;

}
```


### 3-2. JPA Lock의 종류

버전 값을 강제로 증가시키거나, 버전 관리 필드를 무시하는 Bulk 연산의 경우 JPA에서 제공하는 Lock을 사용해야 한다.

<table class="uk-table-small uk-table style-2 uk-table-striped uk-text-center">
    <thead>
        <tr>
            <th style="text-align: center;">Lock Mode</th>
            <th style="text-align: center;">Type</th>
            <th style="text-align: center;">Description</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td><em>낙관적 Lock</em></td>
            <td>OPTIMISTIC</td>
            <td>낙관적 Lock을 사용</td>
        </tr>
        <tr>
            <td><em>낙관적 Lock</em></td>
            <td>OPTIMISTIC_FORCE_INCREMENT</td>
            <td>낙관적 Lock을 사용하며 버전 정보를 강제로 증가</td>
        </tr>
        <tr>
            <td><em>비관적 Lock</em></td>
            <td>PESSIMISTIC_READ</td>
            <td>비관적 Lock, Shared Lock을 사용</td>
        </tr>
        <tr>
            <td><em>비관적 Lock</em></td>
            <td>PESSIMISTIC_WRITE</td>
            <td>비관적 Lock, Exclusive Lock을 사용</td>
        </tr>
        <tr>
            <td><em>비관적 Lock</em></td>
            <td>PESSIMISTIC_FORCE_INCREMENT</td>
            <td>비관적 Lock을 사용하며 버전 정보를 강제로 증가</td>
        </tr>
		<tr>
            <td><em>기타</em></td>
            <td>NONE</td>
            <td>Lock을 걸지 않는다</td>
        </tr>
        <tr>
            <td><em>기타</em></td>
            <td>READ</td>
            <td>OPTIMISTIC과 동일</td>
        </tr>
        <tr>
            <td><em>기타</em></td>
            <td>WRITE</td>
            <td>OPTIMISTIC_FORCE_INCREMENT와 동일</td>
        </tr>
    </tbody>
</table>

JPA를 사용할 때에는 *READ COMMITTED 격리 수준* + *낙관적 버전 관리*를 적용하는 것이 일반적이다

---

## 4. 발표자료

아래는 사내 스터디에서 본문 내용으로 발표를 준비하며 만든 자료이다.

{% include carousel.html height="55" unit="%" duration="20" number="1" %}

---

## Appendix
- [위키백과 ACID](https://ko.wikipedia.org/wiki/ACID)
- [MySQL Isolation Level 종류 및 특징](https://hyunki1019.tistory.com/111)
- [트랜잭션의 격리 수준이란](https://nesoy.github.io/articles/2019-05/Database-Transaction-isolation)
