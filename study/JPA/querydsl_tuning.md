이 글의 내용은 [우아콘2020]의 '수십억건에서 querydsl 사용하기' 영상을 보고 정리하였습니다.
제 글을 보신다면, 제글의 정리된 내용만 보는 것도 좋지만, 본 내용을 제공해준 유튜브를 한 번 꼭 보시는 것을 추천드립니다.

[출처]
* [수십억건에서 querydsl 사용하기(우아콘2020) 동영상](https://www.youtube.com/watch?v=zMAX7g6rO_Y)
* ['수십억건에서 QUERYDSL 사용하기' 를 보고.. 블로그](https://hyune-c.tistory.com/entry/%EC%88%98%EC%8B%AD%EC%96%B5%EA%B1%B4%EC%97%90%EC%84%9C-QUERYDSL-%EC%82%AC%EC%9A%A9%ED%95%98%EA%B8%B0-%ED%9B%84%EA%B8%B0)

--------


### 1. 워밍업

### 1.1 repository에서 extends/implements 사용하지 않기.

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
**@RequiredArgsConstructor** 은 final이 붙거나 @NotNull 이 붙은 필드의 생성자를 자동 생성해주는 롬복 어노테이션으로,
**@RequiredArgsConstructor** 사용 시, 따로 JPAQueryFactory를 주입하는 생성자를 만들지 않아도 됩니다.

**주의 사항**
```
repository에 대한 implements를 제거 하면, 기본 repository에서 제공해주는 기본 메서드들은 사용 할 수 없습니다.
Entity에 대한 기본 repository의 함수를 사용하고 싶다면 따로 만들어서 사용해야 된다는 단점이 존재합니다.
따라서, 위와 같은 방법은 단일 Entity에 구속 받지 않고, 여러 Entity를 사용하는 서비스 로직에 초점을 둔 repository들 사용시에 용이하다고 생각 됩니다.
```


### 1.2 조건절에 동적쿼리 사용 시, 가독성을 위해 BooleanExpression 사용하는 함수 활용

변경 전
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

변경 후
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
