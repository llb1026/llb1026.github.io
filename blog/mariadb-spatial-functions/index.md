---
order: 17
layout: post
title: "내가 사용했던 MariaDB 공간함수"
subtitle: "나중에 보려고 정리한 공간함수들과 사용 예시들"
tag: Tech Notes
type: tech-notes
blog: true
text: true
visible: true
author: JIYUN LEE
post-header: true
header-img: img/01_1.png
next-link: "../mariadb-sql-mode-oracle/"
prev-link: "../spring-cloud-netflix/"
---

## 1. 구현 조건

카풀 서비스를 개발하면서 라이더(탑승자)가 출근 시 자신의 집과 가까운 카풀 탑승 장소를, 퇴근 시 자신의 집과 가까운 카풀 하차 장소를 찾는 기능을 구현하게 되었다.

라이더의 집 주소, 회사 주소, 탑승 및 하차 가능한 모든 카풀 장소는 DB에 저장되어 있고, 앱에서 검색 필터로 반경 몇 미터 내의 카풀장소를 검색할 것인지에 대한 숫자값이 넘어온다고 했을 때 아래의 2가지 방법으로 접근해보기로 했다.

1. 라이더의 좌표를 중심으로 반경 N 미터의 폴리곤을 그리고, 해당 폴리곤 범위 내에 속한 카풀 장소를 골라낸다
2. 각 카풀 장소와 라이더의 좌표 사이의 거리를 재고, 해당 거리가 N미터 이하인 카풀 장소를 골라낸다

1번 방식을 `MBR_Contains()`로, 2번 방식을 `ST_DISTANCE_SPHERE()`로 구현해보았다.

---

## 2. MBR_Contains()

**MBR**<sup> Minimum Bounding Rectangle </sup>이란 '요소를 포함하는 가장 작은 사각형'이라는 뜻이다. MariaDB에서는 공간 검색 시 MBR들의 그룹을 더 큰 MBR이 묶고, 이런 그룹들을 또다시 더 큰 MBR로 묶는 R-Tree 구조의 인덱스를 사용한다.

R-Tree 인덱스를 사용하는 이유는 아래와 같다고 한다.

- X좌표값, Y좌표값 2개 컬럼으로 B-Tree composite 인덱스를 걸면 Left-most 특성땜에 특정 영역 범위 검색 시 한 쪽밖에 인덱스를 타지 못 함 (데이터 분포에 따라 거의 풀 스캔을 해야 할 수 있음)
- 위치 정보 검색 시 R-Tree 인덱스가 B-Tree와 비교해서 전반적으로 검색이 빠름 (단, 검색영역이 너무 넓으면 느림)

아래 예시 코드에서 `wkt`는  Well Known Text의 약자로, 쉽게 이해할 수 있는 문자열로 데이터를 표현하는 방식이다. 예시로 `'POLYGON((175 150, 20 40, 50 60, 125 100, 175 150))'` 이런 식의 스트링 값이다.

```java
@Query(value = "SELECT * FROM station " + 
    "WHERE MBR_Contains(ST_GeomFromText('" + ":wkt" + "'), point)", 
    nativeQuery = true)
List<StationEntity> findByAreaMbr(String wkt);
```

---

## 3. ST_DISTANCE_SPHERE()

`MBR_Contains()`로 몇 번 테스트를 돌려보았는데, 실행 속도가 그렇게 빠르다고 느껴지지 않았다. 이에 2번 방식을 시도했으나, 개발할 당시 프로젝트가 사용하고 있던 MariaDB 버전은 `ST_DISTANCE_SPHERE()`가 없었다 (10.2.38 버전부터 공식 지원한다고 한다 ([링크](https://mariadb.com/kb/en/st_distance_sphere/))). 

따라서 직접 MySQL의 함수를 가져다 MariaDB에 만들어 사용했다.

정확히는 `MBR_Contains()`가 spatial index를 타고, `ST_DISTANCE_SPHERE()`는 풀스캔을 하기 때문에 함수 자체로만 놓고 보면 `MBR_Contains()`가 빠르게 처리되는 것이 맞다. 제대로 된 비교를 위해서는 나머지 소스들은 동일하게 가져가고 사용하는 함수만 갈아끼워 테스트 했어야 했는데, `MBR_Contains()`에서 `ST_DISTANCE_SPHERE()`로 변경하며 spatial index를 태운 범위필터링을 앞단에 한 번 추가로 거쳤고, 쿼리 전후의 Java단 로직이 변경되어 제대로 된 비교가 되지 않았다. 개발계의 테스트용 더미데이터가 적었던 것도 하나의 원인이었던 것 같다.

```sql
CREATE FUNCTION `st_distance_sphere`(`pt1` POINT, `pt2` POINT) RETURNS decimal(10,2)
    BEGIN
    return 6371000 * 2 * ASIN(SQRT(
       POWER(SIN((ST_Y(pt2) - ST_Y(pt1)) * pi()/180 / 2),
       2) + COS(ST_Y(pt1) * pi()/180 ) * COS(ST_Y(pt2) *
       pi()/180) * POWER(SIN((ST_X(pt2) - ST_X(pt1)) *
       pi()/180 / 2), 2) ));
    END
```

참고로 `ST_DISTANCE()`는 평면 거리를 재는 함수이기 때문에, 우리가 살고 있는 구체의 지구에서 잰 거리와는 다른 값을 뱉는다.

사용 예시는 아래와 같다.

```java
@Query(value = "SELECT * from station " + 
    "WHERE ST_DISTANCE_SPHERE(ST_GeomFromText(CONCAT('Point(', :lon, ' ', :lat, ')')), point) <= :range", 
    nativeQuery = true)
List<StationEntity> findByAreaDistance(double range, double lon, double lat);
```

```java
@Query(value = "SELECT *, ST_DISTANCE_SPHERE(Point(:lon, :lat), point) AS distance FROM station " + 
    "WHERE name = :#{#station.name()} " + 
    "HAVING distance <= :range " + 
    "ORDER BY distance " + 
    "LIMIT 0, :limit", 
    nativeQuery = true)
List<StationEntity> findByDistance(double range, rouble lon, double lat, StationObject station, int limit);
```

---

## References

- [R-Tree 인덱스](https://jangjunha.github.io/blog/mysql-mariadb-spatial-index/)
- [ST_DISTANCE_SPHERE 함수 생성 쿼리](https://stackoverflow.com/questions/44409012/function-st-distance-sphere-does-not-exist-in-mariadb)