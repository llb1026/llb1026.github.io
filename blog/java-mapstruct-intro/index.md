---
order: 23
layout: post
title: "MapStruct를 써야 하는 이유"
subtitle: "자주 쓰는 @Mapping 사용법 정리"
tag: Tech Notes
type: tech-notes
blog: true
text: true
author: JIYUN LEE
post-header: true
header-img: img/01_1.png
next-link: "../spring-webflux-intro/"
prev-link: "../mariadb-sql-mode-oracle/"
---

## 1. 인트로

프로젝트 초반, 함께 같은 파트를 개발하게 된 동료에게 JPA를 쓰자고 설득하는 과정에서 MapStruct 라이브러리도 함께 설명했었다. 기억을 되살려 MapStruct를 썼을 때의 이점, 그리고 내가 자주 쓰는 예시들을 정리해보았다.

### 1-1. MapStruct 소개

MapStruct는 Java Bean Mapper 라이브러리다. 주로 Entity와 DTO 간, DTO와 DTO 간 형변환 시 사용한다. 개발자가 인터페이스만 명시해주면, MapStruct가 빌드 시 자동으로 변환하는 코드를 생성한다. ([Baeldung 문서](https://www.baeldung.com/mapstruct))

MapStruct 이외에도 다른 매핑 프레임워크들이 있는데, [Baeldung 문서](https://www.baeldung.com/java-performance-mapping-frameworks)에 따르면 JMapper와 MapStruct가 가장 좋은 성능을 보인다고 한다. modelMapper와 비교했을 때만 해도 압도적인 성능 차이가 있으니, 웬만하면 MapStruct 쓰자.

---

## 2. MapStruct 사용법

예시로 아래와 같은 데이터 구조를 생각해보자.

- `user` 테이블에는 사용자 정보 - 이름, 이메일, 우편번호, 도로명주소, 상세주소 데이터가 들어있다.
- `vehicle` 테이블에는 사용자가 소유한 차량의 정보 - 번호판, 대표차량여부 데이터가 들어있다.
- 사용자는 여러 대의 차량을 보유할 수 있고, 그 중 하나의 차량을 대표차량으로 설정할 수 있다.

```sql
create table user (
    id             bigint auto_increment primary key,
    entity_version bigint       null,
    name           varchar(50)  null,
    email          varchar(100) null,
    zip_code       varchar(10)  null,
    road_address   varchar(500) null,
    address_detail varchar(500) null,
    created_at     timestamp    null,
    updated_at     timestamp    null
)
```

```sql
create table vehicle(
    id             bigint auto_increment primary key,
    entity_version bigint               null,
    user_id        bigint               null,
    number_plate   varchar(36)          null,
    represented    tinyint(1) default 1 null,
    created_at     timestamp            null,
    updated_at     timestamp            null,
    constraint vehicle_user_id_fk
        foreign key (user_id) references user (id)
)
```

이렇게 만든 테이블을 토대로 Entity는 아래와 같이 만들 수 있다. Entity마다 공통된 컬럼들(`id`, `entity_version`, `created_at`, `updated_at`)은 별도의 BaseEntity로 분리했다.

```java
public class UserEntity extends BaseEntity implements Serializable {

    @Column(name = "name")
    private String name;

    @Column(name = "email")
    private String email;

    @Column(name = "zip_code")
    private String zipCode;

    @Column(name = "road_address")
    private String roadAddress;

    @Column(name = "address_detail")
    private String addressDetail;

}
```

```java
public class VehicleEntity extends BaseEntity implements Serializable {

    @Column(name = "user_id")
    private Long userId;

    @Column(name = "number_plate")
    private String numberPlate;

    @Column(name = "represented")
    private Boolean represented;

}
```

그리고 Service 로직에서 사용할 DTO 구조는 아래와 같다고 가정한다. (Entity와 동일한 필드를 갖고 있는 BaseDTO는 본문에서 생략한다)

```java
public class UserCustomDto implements Serializable {

    private Long id;

    private String name;

    private String email;

    private AddressDto address;

    private List<VehicleCustomDto> vehicleList;

}
```

```java
public class AddressDto implements Serializable {

    private String zipCode;

    private String roadAddress;

    private String addressDetail;

}
```

```java
public class VehicleCustomDto implements Serializable {

    private Long id;

    private String numberPlate;

    private String represented;

}
```

### 2-1. MapStruct를 사용하지 않았을 때

사용자의 아이디로 사용자 정보와 사용자의 차량정보를 리턴하는 API를 만든다고 하자. `user` 테이블에서는 `findById`로, `vehicle` 테이블에서는 `findByUserId`로 원하는 데이터 Entity들을 확보한다. 

이제 응답값으로 `UserCustomDto`를 리턴해야 하는데, 별도의 매퍼를 사용하지 않는다면 개발자가 직접 Entity에서 DTO로 변환하는 코드를 작성해줘야 한다.

```java
private UserCustomDto convert(UserEntity user, List<VehicleEntity> vehicleList) {
    return UserCustomDto.builder()
            .id(user.getId())
            .name(user.getName())
            .email(user.getEmail())
            .address(AddressDto.builder()
                    .roadAddress(user.getRoadAddress())
                    .addressDetail(user.getAddressDetail())
                    .zipCode(user.getZipCode())
                    .build())
            .vehicleList(vehicleList.stream().map(vehicle -> VehicleCustomDto.builder()
                    .id(vehicle.getId())
                    .numberPlate(vehicle.getNumberPlate())
                    .represented(vehicle.getRepresented() ? "대표차량" : "일반차량")
                    .build()).collect(Collectors.toList()))
            .build();
}
```

필요한 모든 객체 간에 변환메소드를 일일히 작성해야 하는 것도 귀찮은데, 데이터 타입이 변경되거나 데이터 depth가 더 깊어지면 훨씬 더 복잡하고 귀찮아진다. 또한 저 엄청나게 길어지는 변환메소드를 둘 곳도 마땅치않고, 누군가 알게모르게 수정할 지 모르는 소스코드로 관리하는 것도 찜찜하다.

### 2-2. MapStruct를 사용할 때

위와 같은 노가다를 MapStruct에게 맡길 수 있다. 아래와 같이 인터페이스에 어떻게 변환해야 하는지를 명시해주면 MapStruct가 변환코드를 자동으로 생성한다.

먼저 가장 기본적으로, Entity에서 BaseDTO, BaseDTO에서 Entity로의 변환은 `GenericMapper`에서 정의한 메소드를 오버라이딩하면 끝이다.

```java
public interface GenericMapper<D, E> {

    D toDto(E entity);
    E toEntity(D dto);

    @BeanMapping(nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
    void updateFromDto(D dto, @MappingTarget E entity);

    List<D> getDtoList(List<E> entityList);

}
```

```java
@Mapper(componentModel = "spring",
        unmappedTargetPolicy = ReportingPolicy.IGNORE,
        unmappedSourcePolicy = ReportingPolicy.IGNORE)
public interface UserMapper extends GenericMapper<UserBaseDto, UserEntity> {

    @Override
    UserBaseDto toDto(UserEntity entity);

    @Override
    UserEntity toEntity(UserBaseDto dto);
}
```

이렇게 선언해주고 빌드하면, 프로젝트의 `build/generated/source/annotationProcessor/` 하위에 `~MapperImpl`가 생성된 것을 확인할 수 있다.

MapStruct가 위의 인터페이스를 토대로 자동생성한 코드는 아래와 같다.

```java
@Override
public UserBaseDto toDto(UserEntity entity) {
    if ( entity == null ) {
        return null;
    }

    UserBaseDtoBuilder userBaseDto = UserBaseDto.builder();

    userBaseDto.id( entity.getId() );
    userBaseDto.entityVersion( entity.getEntityVersion() );
    userBaseDto.name( entity.getName() );
    userBaseDto.email( entity.getEmail() );
    userBaseDto.zipCode( entity.getZipCode() );
    userBaseDto.roadAddress( entity.getRoadAddress() );
    userBaseDto.addressDetail( entity.getAddressDetail() );
    userBaseDto.createdAt( entity.getCreatedAt() );
    userBaseDto.updatedAt( entity.getUpdatedAt() );

    return userBaseDto.build();
}

@Override
public UserEntity toEntity(UserBaseDto dto) {
    if ( dto == null ) {
        return null;
    }

    UserEntityBuilder userEntity = UserEntity.builder();

    userEntity.name( dto.getName() );
    userEntity.email( dto.getEmail() );
    userEntity.zipCode( dto.getZipCode() );
    userEntity.roadAddress( dto.getRoadAddress() );
    userEntity.addressDetail( dto.getAddressDetail() );

    return userEntity.build();
}
```

---

### 2-3. 자주 쓰는 @Mapping 예시

필드 구조가 똑같은 Entity와 BaseDTO 간의 매핑은 확인했으니, 이제 CustomDTO로 세부 매핑하는 방법을 알아보자.

우선 인터페이스 전문을 먼저 올리고, 밑에 하나씩 설명을 달도록 하겠다.

```java
@Mapper(componentModel = "spring",
        unmappedTargetPolicy = ReportingPolicy.IGNORE,
        unmappedSourcePolicy = ReportingPolicy.IGNORE,
        uses = { VehicleMapper.class },
        imports = { AddressDto.class, UserCustomDto.class, VehicleCustomDto.class })
public interface UserMapper extends GenericMapper<UserBaseDto, UserEntity> {

    @Override
    UserBaseDto toDto(UserEntity entity);

    @Override
    UserEntity toEntity(UserBaseDto dto);

    @Named("toCustomUserDto")
    @Mapping(target = "address", expression = "java(AddressDto.builder()" +
            ".zipCode(userBaseDto.getZipCode())" +
            ".roadAddress(userBaseDto.getRoadAddress())" +
            ".addressDetail(userBaseDto.getAddressDetail()).build())")
    @Mapping(target = "vehicleList", source = "vehicleBaseDtoList", qualifiedByName = "toCustomVehicleDto")
    UserCustomDto toCustomDto(UserBaseDto userBaseDto, List<VehicleBaseDto> vehicleBaseDtoList);

}
```

```java
@Mapper(componentModel = "spring",
        unmappedTargetPolicy = ReportingPolicy.IGNORE,
        unmappedSourcePolicy = ReportingPolicy.IGNORE,
        imports = { VehicleCustomDto.class })
public interface VehicleMapper extends GenericMapper<VehicleBaseDto, VehicleEntity> {

    @Override
    VehicleBaseDto toDto(VehicleEntity entity);

    @Override
    VehicleEntity toEntity(VehicleBaseDto dto);

    @Named("toCustomVehicleDto")
    @Mapping(target = "represented", expression = "java(baseDto.getRepresented() ? \"대표차량\" : \"일반차량\")")
    VehicleCustomDto toCustomDto(VehicleBaseDto baseDto);

}
```

- `@Mapper` 어노테이션으로 본 인터페이스가 매퍼라고 명시한다.
- `unmappedTargetPolicy`는 변환대상의 어떤 컬럼이 매핑되지 않았을 때 처리방식을 명시한다. `ReportingPolicy.IGNORE`은 매핑되지 않았어도 리포트 하지 않고 넘어간다.
- `unmappedSourcePolicy`는 위와 동일하게 변환소스가 매핑되지 않았을 때 처리방식을 명시한다.
- `@Named`는 매핑 메소드에 특정 qualifier name을 붙여줄 때 사용한다. 위에서의 예시로, `VehicleMapper`에서 `toCustomDto`에 `@Named("toCustomVehicleDto")`을 달아주고, `UserMapper`에서 `@Mapping(target = "vehicleList", source = "vehicleBaseDtoList", qualifiedByName = "toCustomVehicleDto")`와 같이 사용할 수 있다.
- `target`은 변환대상의 필드명이다.
- `source`는 매핑 메소드 파라미터의 데이터들로부터 직접 접근해 변환소스를 명시하는 방법이다.
- `expression`은 파라미터의 데이터들로부터 바로 데이터를 가져올 수 없을 경우, 추가 변환이 필요한 경우 `java()` 표현 안에 JAVA 문법으로 변환소스를 명시하는 방법이다.
- `uses`는 매핑 메소드에서 다른 매퍼를 사용할 때 해당 매퍼를 명시한다.
- `imports`는 매핑 메소드에서 `expression`으로 JAVA 문법 사용 시 필요한 추가 클래스들을 명시한다.

이렇게 명시한 인터페이스로 MapStruct가 자동생성한 변환코드는 아래와 같다.

```java
@Component
public class UserMapperImpl implements UserMapper {

    @Autowired
    private VehicleMapper vehicleMapper;

    @Override
    public void updateFromDto(UserBaseDto dto, UserEntity entity) {
        if ( dto == null ) {
            return;
        }

        if ( dto.getId() != null ) {
            entity.setId( dto.getId() );
        }
        if ( dto.getEntityVersion() != null ) {
            entity.setEntityVersion( dto.getEntityVersion() );
        }
        if ( dto.getCreatedAt() != null ) {
            entity.setCreatedAt( dto.getCreatedAt() );
        }
        if ( dto.getUpdatedAt() != null ) {
            entity.setUpdatedAt( dto.getUpdatedAt() );
        }
        if ( dto.getName() != null ) {
            entity.setName( dto.getName() );
        }
        if ( dto.getEmail() != null ) {
            entity.setEmail( dto.getEmail() );
        }
        if ( dto.getZipCode() != null ) {
            entity.setZipCode( dto.getZipCode() );
        }
        if ( dto.getRoadAddress() != null ) {
            entity.setRoadAddress( dto.getRoadAddress() );
        }
        if ( dto.getAddressDetail() != null ) {
            entity.setAddressDetail( dto.getAddressDetail() );
        }
    }

    @Override
    public List<UserBaseDto> getDtoList(List<UserEntity> entityList) {
        if ( entityList == null ) {
            return null;
        }

        List<UserBaseDto> list = new ArrayList<UserBaseDto>( entityList.size() );
        for ( UserEntity userEntity : entityList ) {
            list.add( toDto( userEntity ) );
        }

        return list;
    }

    @Override
    public UserBaseDto toDto(UserEntity entity) {
        if ( entity == null ) {
            return null;
        }

        UserBaseDtoBuilder userBaseDto = UserBaseDto.builder();

        userBaseDto.id( entity.getId() );
        userBaseDto.entityVersion( entity.getEntityVersion() );
        userBaseDto.name( entity.getName() );
        userBaseDto.email( entity.getEmail() );
        userBaseDto.zipCode( entity.getZipCode() );
        userBaseDto.roadAddress( entity.getRoadAddress() );
        userBaseDto.addressDetail( entity.getAddressDetail() );
        userBaseDto.createdAt( entity.getCreatedAt() );
        userBaseDto.updatedAt( entity.getUpdatedAt() );

        return userBaseDto.build();
    }

    @Override
    public UserEntity toEntity(UserBaseDto dto) {
        if ( dto == null ) {
            return null;
        }

        UserEntityBuilder userEntity = UserEntity.builder();

        userEntity.name( dto.getName() );
        userEntity.email( dto.getEmail() );
        userEntity.zipCode( dto.getZipCode() );
        userEntity.roadAddress( dto.getRoadAddress() );
        userEntity.addressDetail( dto.getAddressDetail() );

        return userEntity.build();
    }

    @Override
    public UserCustomDto toCustomDto(UserBaseDto userBaseDto, List<VehicleBaseDto> vehicleBaseDtoList) {
        if ( userBaseDto == null && vehicleBaseDtoList == null ) {
            return null;
        }

        UserCustomDtoBuilder userCustomDto = UserCustomDto.builder();

        if ( userBaseDto != null ) {
            userCustomDto.id( userBaseDto.getId() );
            userCustomDto.name( userBaseDto.getName() );
            userCustomDto.email( userBaseDto.getEmail() );
        }
        if ( vehicleBaseDtoList != null ) {
            userCustomDto.vehicleList( vehicleBaseDtoListToVehicleCustomDtoList( vehicleBaseDtoList ) );
        }
        userCustomDto.address( AddressDto.builder().zipCode(userBaseDto.getZipCode()).roadAddress(userBaseDto.getRoadAddress()).addressDetail(userBaseDto.getAddressDetail()).build() );

        return userCustomDto.build();
    }

    protected List<VehicleCustomDto> vehicleBaseDtoListToVehicleCustomDtoList(List<VehicleBaseDto> list) {
        if ( list == null ) {
            return null;
        }

        List<VehicleCustomDto> list1 = new ArrayList<VehicleCustomDto>( list.size() );
        for ( VehicleBaseDto vehicleBaseDto : list ) {
            list1.add( vehicleMapper.toCustomDto( vehicleBaseDto ) );
        }

        return list1;
    }
}
```

```java
@Component
public class VehicleMapperImpl implements VehicleMapper {

    @Override
    public void updateFromDto(VehicleBaseDto dto, VehicleEntity entity) {
        if ( dto == null ) {
            return;
        }

        if ( dto.getId() != null ) {
            entity.setId( dto.getId() );
        }
        if ( dto.getEntityVersion() != null ) {
            entity.setEntityVersion( dto.getEntityVersion() );
        }
        if ( dto.getCreatedAt() != null ) {
            entity.setCreatedAt( dto.getCreatedAt() );
        }
        if ( dto.getUpdatedAt() != null ) {
            entity.setUpdatedAt( dto.getUpdatedAt() );
        }
        if ( dto.getUserId() != null ) {
            entity.setUserId( dto.getUserId() );
        }
        if ( dto.getNumberPlate() != null ) {
            entity.setNumberPlate( dto.getNumberPlate() );
        }
        if ( dto.getRepresented() != null ) {
            entity.setRepresented( dto.getRepresented() );
        }
    }

    @Override
    public List<VehicleBaseDto> getDtoList(List<VehicleEntity> entityList) {
        if ( entityList == null ) {
            return null;
        }

        List<VehicleBaseDto> list = new ArrayList<VehicleBaseDto>( entityList.size() );
        for ( VehicleEntity vehicleEntity : entityList ) {
            list.add( toDto( vehicleEntity ) );
        }

        return list;
    }

    @Override
    public VehicleBaseDto toDto(VehicleEntity entity) {
        if ( entity == null ) {
            return null;
        }

        VehicleBaseDtoBuilder vehicleBaseDto = VehicleBaseDto.builder();

        vehicleBaseDto.id( entity.getId() );
        vehicleBaseDto.entityVersion( entity.getEntityVersion() );
        vehicleBaseDto.userId( entity.getUserId() );
        vehicleBaseDto.numberPlate( entity.getNumberPlate() );
        vehicleBaseDto.represented( entity.getRepresented() );
        vehicleBaseDto.createdAt( entity.getCreatedAt() );
        vehicleBaseDto.updatedAt( entity.getUpdatedAt() );

        return vehicleBaseDto.build();
    }

    @Override
    public VehicleEntity toEntity(VehicleBaseDto dto) {
        if ( dto == null ) {
            return null;
        }

        VehicleEntityBuilder vehicleEntity = VehicleEntity.builder();

        vehicleEntity.userId( dto.getUserId() );
        vehicleEntity.numberPlate( dto.getNumberPlate() );
        vehicleEntity.represented( dto.getRepresented() );

        return vehicleEntity.build();
    }

    @Override
    public VehicleCustomDto toCustomDto(VehicleBaseDto baseDto) {
        if ( baseDto == null ) {
            return null;
        }

        VehicleCustomDtoBuilder vehicleCustomDto = VehicleCustomDto.builder();

        vehicleCustomDto.id( baseDto.getId() );
        vehicleCustomDto.numberPlate( baseDto.getNumberPlate() );

        vehicleCustomDto.represented( baseDto.getRepresented() ? "대표차량" : "일반차량" );

        return vehicleCustomDto.build();
    }
}
```

---

### 2-4. 소스코드

위 케이스를 바탕으로 작성한 어플리케이션 소스코드는 [Github](https://github.com/llb1026/mapstruct_practice) 에서 확인할 수 있다.