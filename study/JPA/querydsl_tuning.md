이 글의 내용은 [우아콘2020]의 '수십억건에서 querydsl 사용하기' 영상을 보고 정리하였습니다.

--------


### 1. 워밍업

#### 1.1 repository에서 extends/implements 사용하지 않기.

변경 전
```java
@Repository
public class AlarmManagerRepositorySupport extends QuerydslRepositorySupport {

    private final JPAQueryFactory queryFactory;

    public AlarmManagerRepositorySupport(JPAQueryFactory queryFactory) {
        super(AlarmSettings.class);
        this.queryFactory = queryFactory;
    }
```

변경 후
```java
@RequiredArgsConstructor
@Repository
public class AlarmManagerRepositorySupport{

    private final JPAQueryFactory queryFactory;

```
**@RequiredArgsConstructor**은 final이 붙거나 @NotNull 이 붙은 필드의 생성자를 자동 생성해주는 롬복 어노테이션으로,
**@RequiredArgsConstructor** 사용 시, 따로 JPAQueryFactory를 주입하는 생성자를 만들지 않아도 됩니다.

**repository에 대한 implements를 제거 하면, 기본 repository에서 제공해주는 기본 메서드들은 사용 할 수 없습니다.
Entity에 대한 기본 repository의 함수를 사용하고 싶다면 따로 만들어서 사용해야 된다는 단점이 존재합니다.
따라서, 위와 같은 방법은 단일 Entity에 구속 받지 않고, 여러 Entity를 사용하는 서비스 로직에 초점을 둔 repository들 사용시에 용이하다고 생각 됩니다.**

#### 1.2 조건절에 동적쿼리 사용 시, 가독성을 위해 b 함수 만들어서 where절어
