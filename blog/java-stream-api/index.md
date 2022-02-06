---
order: 13
layout: post
title: "Java Stream API 사용 모음집"
subtitle: "자주 쓰는 메소드들 정리"
tag: Tech Notes
type: tech-notes
blog: true
text: true
visible: false
author: JIYUN LEE
post-header: true
header-img: img/01_1.png
next-link: "../java-serializable/"
prev-link: "../restful-api/"
---

## 1. filter()

if문처럼 특정 조건에 맞는 것들만 걸러낼 때 사용한다.

```java
List<TempEntity> result = tempRepository.findAll()
    .stream().filter(temp -> temp.getDistance() < 100)
    .collect(Collectors.toList());
```

## 2. map()

스트림 안의 각 항목들에 특정 함수를 적용시킨다. 주로 형 변환 시 자주 쓴다.

```java
List<TempEntity> result = tempRepository.findAll()
    .stream().map(temp -> tempMapper.toDto(temp))
    .collect(Collectors.toList());
```

## 3. distinct()

중복된 항목을 제거할 때 사용한다.

```java
List<TempEntity> duplicateList = List.of(/* 생략 */);
List<TempEntity> distinctList = duplicateList
    .stream().distinct().collect(Collectors.toList());
```

## 7. forEach()

## 4. sorted()

## 5. peek()

## 6. limit()

## 8. reduce()

## 9. collect()

## 10. min(), max()

## 11. count(), empty()

## 12. ~Match()

## 13. find~()

## 14. of()