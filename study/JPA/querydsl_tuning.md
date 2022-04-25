이 글의 내용은 [우아콘2020]의 '수십억건에서 querydsl 사용하기' 영상을 보고 정리하였습니다.

www.youtube.com/watch?v=zMAX7g6rO_Y


1. 워밍업

- repository에서 extends/implements 사용하지 않기.

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
