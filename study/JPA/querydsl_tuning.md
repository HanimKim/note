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
+ N+1 문제는 @OneToOne(fetch = FetchType.LAZY) 설정으로도 해결되지 않으며, Fetch Join 의 사용을 추천합니다. 
```
(Fetch Join 공부 필요!! 참고 : https://1-7171771.tistory.com/143)


### 2.4 Group By 최적화 하기.

MySQL에서 Group By를 실행 할 경우, Index를 타지 않으면 Filesort가 필수로 발생합니다.
order by null 을 통해 Filesort 를 제거할 수 있지만 Querydsl 에서는 지원되지 않습니다.
OrderByNull 을 직접 구현함으로서 해결 할 수 있습니다. 

**Using filesort** 는 **정렬이 필요한 데이터를 메모리에 올리고 정렬 작업을 수행한다** 는 의미로, 이미 정렬된 인덱스를 사용함으로써 해결할 수 있습니다. 
영상에서는 우리가 사용하는 모든 Group By 가 index를 탄다는 보장이 없기에 말하고 있습니다.

하지만 8.0 이상을 사용하는 환경이라면 더 이상 고려하지 않아도 됩니다.
```
Previously (MySQL 5.7 and lower), GROUP BY sorted implicitly under certain conditions. In MySQL 8.0, that no longer occurs, so specifying ORDER BY NULL at the end to suppress implicit sorting (as was done previously) is no longer necessary. However, query results may differ from previous MySQL versions. To produce a given sort order, provide an ORDER BY clause.
```
https://dev.mysql.com/doc/refman/8.0/en/order-by-optimization.html

마지막으로 **Paging 이 필요하지 않고 정렬 건수가 적다면 WAS에서 정렬** 하는 것을 추천합니다.
일반적으로 WAS 리소스가 DB 리소스보다는 여유 있고 저렴하기 때문입니다.


### 2.5 커버링 인덱스

커버링 인덱스를 사용할 땐 inline view (from 절의 subQuery)에서 커버링 인덱스를 통해 필터를 하도록 하는 것이 일반적이지만,  
JPQL은 from 절의 서브 쿼리를 지원하지 않아 직접 구현해야 합니다.(서브 쿼리를 빼서, select 절을 두번 조회)

공부 필요!!!

### 3. 성능 개선 - Update/Insert

### 3.1 일괄 Update 최적화

* Dirty Checking - 실시간 비즈니스 처리, 실시간 단건 처리
* Querydsl.update - 대량의 데이터를 일괄로 Update 처리

주의할 점은 일괄 업데이트는 **영속성 컨택스트의 1차 캐시 갱신이 안됨으로 Cache Eviction 이 필요** 합니다.

공부 필요!!

### 3.2 JPA로 Bulk Insert는 자제한다.

JPA 에서는 **Insert 합치기가 적용되지 않는다.**

영상에서 JdbcTemplate를 말하지만 type safe 하지 않은 단점과 함께 EntityQL를 소개합니다. 하지만 이도 설정과 사용법에 단점을 가지고 있으므로 선택적으로 사용해야 된다고 말합니다.
다른 블로그에서는 Spring 진영에서 공식적으로 밀어주는 JOOQ를 추천합니다. starter 가 존재하기에 세팅도 쉽고, 사용 경험도 매우 좋았다고 합니다.

공부 필요!!!

