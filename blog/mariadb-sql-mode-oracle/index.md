---
order: 18
layout: post
title: "죽어도 Datetime으로 바뀌지 않는 Date 타입 컬럼"
subtitle: "믿었던 인프라팀의 배신..?"
tag: Lesson Learned
type: lesson-learned
blog: true
text: true
visible: true
author: JIYUN LEE
post-header: true
header-img: img/01_1.png
next-link: "../java-mapstruct-intro/"
prev-link: "../mariadb-spatial-functions/"
---

## 1. 상황발생

프로젝트 진행 중, 검증계와 운영계 환경도 구축이 완료되어 개발계와 동일하게 DB에 테이블을 생성해야 하는 순간이 왔다. DDL 스크립트를 쭉 복사해서 각 환경에 테이블을 만드는데, 이상하게 검증계에서만 `DATETIME` 타입 컬럼들이 모두 `DATE` 타입으로 생성되었다.

내가 잘못 복붙했나 싶어 테이블 삭제와 DDL 복붙을 반복해봤지만 똑같았고, `ALTER TABLE`로 컬럼 타입을 수정하려고 해도 그대로였다. 쿼리 수행 결과에도 별다른 로그 없이 정상완료로 찍혀 도대체 어느 곳에서 `DATETIME`을 `DATE`로 갈아끼우는지 추측할 수가 없었다.

---

## 2. 범인은 SQL_MODE

아무리 구글링을 해 봐도 답을 찾지 못했고, 검증계에 MariaDB를 설치한 인프라 팀에 문의했다. 그리고 인프라 팀에서 추후 Oracle DB와의 마이그레이션할 가능성을 미리 고려하여 추가한 `SET SQL_MODE='ORACLE';`이 문제의 원인이라는 것을 찾을 수 있었다.

MariaDB 버전 10.3 이후부터 MariaDB에서 Oracle 문법을 쓸 수 있게 하는 옵션이 추가되었고 ([공식문서](https://mariadb.com/kb/en/sql_modeoracle/)), Oracle에서는 `DATE` 타입이 MariaDB에서의 `DATETIME`과 같이 시간값도 포함하고 있어 애초에 `DATETIME` 타입이 없으므로 `DATETIME` 타입의 컬럼을 자동으로 `DATE` 타입으로 변환한 것이다.

### 2-1. Synonyms for Basic SQL Types

<table class="uk-table-small uk-table style-2 uk-table-striped uk-text-center">
    <thead>
        <tr>
            <th style="text-align: center;">Oracle type</th>
            <th style="text-align: center;">MariaDB synonym</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>VARCHAR2</td>
            <td>VARCHAR</td>
        </tr>
        <tr>
            <td>NUMBER</td>
            <td>DECIMAL</td>
        </tr>
        <tr>
            <td>DATE (with time portion)</td>
            <td>MariaDB DATETIME</td>
        </tr>
        <tr>
            <td>RAW</td>
            <td>VARBINARY</td>
        </tr>
        <tr>
            <td>CLOB</td>
            <td>LONGTEXT</td>
        </tr>
        <tr>
            <td>BLOB</td>
            <td>LONGBLOB</td>
        </tr>
    </tbody>
</table>

`SHOW CREATE TABLE`로 DDL을 확인했을 때, 테이블에 MariaDB의 `DATE` 타입 컬럼이 포함되어 있다면 해당 컬럼은 `mariadb_shcema.date`로 표시된다고 한다.

---

## 3. 교훈

1. 프레임워크, 라이브러리, DB를 막론하고 **사용하는 버전과 해당 버전의 특이사항을 제대로 파악**할 수 있도록 노력하자.
2. 반나절정도 혼자서 온갖 노력을 총동원해도 실마리조차 보이지 않는다면, 빠르게 다른 동료나 유관부서에 도움을 요청하자. 이런 삽질은 어느정도 이상으로 시간을 많이 잡아먹게 되면 손해인 것 같다.