---
order: 21
layout: post
title: "내가 사용했던 MariaDB 공간함수"
subtitle: "나중에 보려고 정리한 공간함수들과 사용 예시들"
tag: Tech Notes
type: tech-notes
blog: true
text: true
author: JIYUN LEE
post-header: true
header-img: img/01_1.png
next-link: "../mariadb-sql-mode-oracle/"
prev-link: "../spring-cloud-netflix/"
---

```java
@Query(value = "SELECT * FROM station " + 
    "WHERE MBR_Contains(ST_GeomFromText('" + ":wkt" + "'), point)", 
    nativeQuery = true)
List<StationEntity> findByAreaMbr(String wkt);
```

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