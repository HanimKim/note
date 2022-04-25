이 글의 내용은 [우아콘2020]의 '수십억건에서 querydsl 사용하기' 영상을 보고 정리하였습니다.
제 글을 보신다면, 제글의 정리된 내용만 보는 것도 좋지만, 본 내용을 제공해준 유튜브를 한 번 꼭 보시는 것을 추천드립니다.

[출처]
* [수십억건에서 querydsl 사용하기(우아콘2020) 동영상](https://www.youtube.com/watch?v=zMAX7g6rO_Y)
* ['수십억건에서 QUERYDSL 사용하기' 를 보고.. 블로그](https://hyune-c.tistory.com/entry/%EC%88%98%EC%8B%AD%EC%96%B5%EA%B1%B4%EC%97%90%EC%84%9C-QUERYDSL-%EC%82%AC%EC%9A%A9%ED%95%98%EA%B8%B0-%ED%9B%84%EA%B8%B0)

--------


### 1. 워밍업

### 1.1 repository에서 extends/implements 사용하지 않기.

변경 전,
```java
@Repository
public class AlarmManagerRepositorySupport extends QuerydslRepositorySupport {

    private final JPAQueryFactory queryFactory;

    public AlarmManagerRepositorySupport(JPAQueryFactory queryFactory) {
        super(AlarmSettings.class);
        this.queryFactory = queryFactory;
    }
```

변경 후,
```java
@RequiredArgsConstructor
@Repository
public class AlarmManagerRepositorySupport{

    private final JPAQueryFactory queryFactory;

```
**@RequiredArgsConstructor** 은 final이 붙거나 @NotNull 이 붙은 필드의 생성자를 자동 생성해주는 롬복 어노테이션으로,
**@RequiredArgsConstructor** 사용 시, 따로 JPAQueryFactory를 주입하는 생성자를 만들지 않아도 됩니다.

**주의 사항**
```
repository에 대한 implements를 제거 하면, 기본 repository에서 제공해주는 기본 메서드들은 사용 할 수 없습니다.
Entity에 대한 기본 repository의 함수를 사용하고 싶다면 따로 만들어서 사용해야 된다는 단점이 존재합니다.
따라서, 위와 같은 방법은 단일 Entity에 구속 받지 않고, 여러 Entity를 사용하는 서비스 로직에 초점을 둔 repository들 사용시에 용이하다고 생각 됩니다.
```


### 1.2 조건절에 동적쿼리 사용 시, 가독성을 위해 BooleanExpression 사용하는 함수 활용

변경 전,
```java
public List<Book> findBooks(String name, String author) {
    	
    BooleanBuilder builder = new BooleanBuilder();

    if (!StringUtils.isEmpty(name)) {
        builder.and(book.name.eq(name));
    }

    if (!StringUtils.isEmpty(idx)) {
        builder.and(book.author.eq(author));
    }

    return queryFactory
        .selectFrom(book)
            .where(builder)
            .fetch();
}
```

변경 후,
```java
public Page<Book> findBooks(Stirng name, String author, Pageable pageable) {
    QueryResults<Book> book = queryFactory
            .selectFrom(review)
            .where(
        eqName(name),
                eqAuthor(author)
            )
            .orderBy(Book.updatedAt.desc(), Book.idx.desc())
            .offset(pageable.getOffset())
            .limit(pageable.getPageSize())
            .fetchResults();

    return new PageImpl<>(result.getResults(), pageable, result.getTotal());
}

private BooleanExpression eqName(String name) {
    if (StringUtils.isEmpty(name)) {
        return null;
    }

    return book.name.eq(name);
}
```
**변경 전** 과 같이 builder 객체를 생성 후 조건절을 넣는 방식으로 코딩을 한다면, 조건절이
수십건이 되고 많아 진다면 소스코드의 가독성이 낮아집니다.
따라서, **변경 후** 와 같이 BooleanExpression을 활용한 함수를 만들어서 where 조건절에 한번에 넣는다면,
작성되어 있는 쿼리가 어떻한 내용의 쿼리인지 가독성이 높아집니다.

### 2. 성능개선 - Select

### 2.1 querydsl의 exist 매소드 사용 금지.

원래 native query의 exist 매소드는 조건에 맞는 데이터가 발견시 종료되어 count 매소드 보다 성능이 뛰어납니다.
그런데 querydsl의 exist 매소드는 ```count > 0``` 를 사용하게 구성되어 있어, 전체 행을 모두 조회하여 검색 대상이 많을 수록 성능이 떨어집니다.
따라서 exist 매소드를 사용하려면, querydsl의 exist 매소드 말고 따로 exist 매소드를 만들어서 사용하는 것을 권장합니다.

### 2.2 명시적 Join을 사용하여 Cross Join 회피하기.

변경 전,
```java
@Transactional(readOnly = true)
public List<Book> crossJoin() {
    return queryFactory
            .selectFrom(book)
            .where(
                    book.idx.gt(book.lender.idx)
            )
            .fetch();
}
```

변경 후,
```java
@Transactional(readOnly = true)
public List<Book> innerJoin() {
    return queryFactory
            .selectFrom(book)
            .innerJoin(book.lender, lender)
            .where(
                    book.idx.gt(lender.idx)
            )
            .fetch();
}
```
**변경 전** 과 같이 Join을 선언 안해주고, Entity에 포함 되어있는 것으로 조건절을 사용하면, 묵시적 Join으로 Cross Join이 발생한다.


### 2.3 Entity 보다는 Dto를 우선 사용.

Entity를 직접 조회하면 단순 조회 기능에서의 성능 이슈 요소가 있습니다.

* 하이버네이트 1차, 2차 캐시 문제 발생
* 불필요한 칼럼을 조회합니다.
* @OneToOne N+1 쿼리 발생

결론적으로 **실시간으로 Entity 변경이 필요한 경우엔 Entity** 를, **성능 개선이나 대량의 데이터 조회가 필요한 경우엔 Dto** 를 조회하는 것을 추천합니다.

```
+ dto 사용 시, 패러미터로 받아서 이미 알고 있는 값은, as 표현식을 사용하여 조회하는 컬럼에서 제외 시켜줍니다.
+ N+1 문제는 @OneToOne(fetch = FetchType.LAZY) 설정으로도 해결되지 않으며, Fetch Join 의 사용을 추천합니다. \
(### 공부 필요!! 참고 : https://1-7171771.tistory.com/143)
```


### 2.4 Group By 최적화 하기.

MySQL에서 Group By를 실행 할 경우, Index를 타지 않으면 Filesort가 필수로 발생합니다.
order by null 을 통해 Filesort 를 제거할 수 있지만 Querydsl 에서는 지원되지 않습니다.
OrderByNull 을 직접 구현함으로서 해결 할 수 있습니다. 

