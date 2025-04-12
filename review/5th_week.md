# QueryDSL
```java
// QueryDSL 기본 쿼리 기능
JPAQuery query = new JPAQuery(em);
QItem item = Qitem.item;
List<Item> list = query.from(item)
        .where(item.name.eq("상품1").and(item.price.gt(20000)))
        .list(item);
```

where 절에는 and 나 or 을 사용할 수 있다.

```java
item.price.between(10000,20000);
item.name.contains("상품1");
item.name.startsWith("고급");
```

### 결과 조회
- uniqueResult() : 조회 결과가 한 건일 때 사용. 조회 결과가 없으면 null 반환, 두 개 이상이면 NonUniqueResultException 발생
- singleResult() : uniqueResult() 와 같지만 두 개 이상이면 처음 데이터 반환
- list() : 결과가 하나 이상일 때 사용, 결과 없으면 빈 컬렉션 반환

### 페이징과 정렬

```java
SearchResults<Item> result =
    query.from(item)
            .where(item.price.gt(10000))
            .offset(10).limit(20)
            .listResults(item);

long total = result.getTotal(); // 검색된 전체 데이터 수
long limit = result.getLimit();
long offset = result.getOffset();
List<Item> results = result.getResults(); // 조회된 데이터
```

listResults() 를 사용하면 전체 데이터 조회를 위한 count 쿼리를 한 번 더 실행함
그리고 SearchResults 를 반환

### 조인

```java
// inner join 예제
QMember member = QMember.member;
QTeam team = QTeam.team;

List<Member> result = queryFactory
        .selectFrom(member)
        .join(member.team, team)
        .where(team.name.eq("teamA"))
        .fetch();
```

### 서브쿼리
```java
// 나이가 가장 많은 사람 조회
QMember member = QMember.member;
QMember memberSub = new QMember("memberSub");

JPAQueryFactory queryFactory = new JPAQueryFactory(em);

List<Member> result = queryFactory
.selectFrom(member)
.where(member.age.eq(
JPAExpressions
.select(memberSub.age.max())
.from(memberSub)
))
.fetch();
```

### 네이티브 SQL

JPQL 은 데이터베이스에 종속적인 기능은 지원하지 않는다
- 특정 데이터베이스만 지원하는 함수, 문법, SQL 쿼리 힌트 : JPQL 에서 네이티브 SQL 함수 호출할 수 있다
- 인라인 뷰, UNION, INTERSECT : 하이버네이트를 포함한 몇몇 JPA 구현체들이 지원한다
- 스토어드 프로시저 : 몇몇 JPA 구현체들이 지원한다

네이티브 SQL 을 사용하면 엔티티를 직접 조회할 수 있고 JPA 가 지원하는 영속성 컨텍스트의 기능을 그대로 사용할 수 있다

#### 네이티브 SQL 사용
```java
// 결과 타입 정의
public Query createNativeQuery(String sqlString, Class resultClass);

// 결과 타입을 정의할 수 없을 때
public Query createNativeQuery(String sqlString);

// 결과 매핑 사용
public Query createNativeQuery(String sqlString, String resultSetMapping);
```

네이티브 SQL 은 관리하기 쉽지 않고 특정 DB 에 종속적이기 때문에 이식성이 떨어진다
가능한 JPA 구현체가 제공하는 기능을 사용할 것

### 객체지향 쿼리 심화

#### 벌크 연산
여러 엔티티를 수정할 때 save(), remove() 등 하나씩 처리하면 오래걸린다
그럴 땐 벌크연산을 사용하면 된다

- 참고 : 100건 이상 한 번에 많은 양을 수정할 때 필수적
- 영속성 컨텍스트를 사용하는 JPA 특성이 그 이유
```java
// UPDATE 벌크 연산
String qlString = 
    "update Book b " +
    "set b.price = b.price * 1.1 " +
    "where b.stockAmount < :stockAmount";

int resultCount = em.createQuery(qlString)
                    .setParameter("stockAmount", 10)
                    .executeUpdate();
```

#### 벌크 연산 시 주의점
벌크 계산은 영속성 컨텍스트를 무시하고 데이터베이스에 직접 쿼리한다
1. 영속성 컨텍스트에 올라가있는 ProductA 가 있고
2. 벌크 연산을 통해 모든 상품의 가격 10% 를 증가시켰다면
3. 벌크 연산은 데이터베이스에 직접 쿼리하기 때문에
4. ProductA 는 update 되지 않는

해결 방법 
- em.refresh(), em.clear()
- 벌크 연산 먼저 실행

#### JPQL 조회 시 엔티티와 영속성 컨텍스트 
member1 과 member2 를 조회하는 JPQL 을 실행하는 시나리오가 있다
1. JPQL 은 조회를 요청하면서 SQL 로 변환된다
2. 조회한 결과와 영속성 컨텍스트를 비교
3. 식별자 값을 기준으로 member1 은 이미 영속성 컨텍스트에 있으므로 버리고 기존에 컨텍스트에 존재하던 member1 을 반환
4. 식별자 값을 기준으로 member2 는 영속성 컨텍스트에 없으므로 영속성 컨텍스트에 추가
5. 영속성 컨텍스트에 있던 member1 과 DB 에 있던 member2 를 반환

- 알 수 있는 사실
1. JPQL 로 조회한 엔티티는 영속 상태다
2. 영속성 컨텍스트에 존재하는 엔티티가 있으면 기존 엔티티를 반환한다

- 영속성 컨텍스트에 수정중인 데이터가 사라질 수 있으므로 영속성 컨텍스트에 있던 기존 엔티티를 새로 검색한 엔티티로 대체하는 것은 위험하다
- 따라서 영속성 컨텍스트는 영속 상태인 엔티티의 동일성을 보장한다

#### find() vs JPQL


>find() 메서드는 영속성 컨텍스트에서 찾고 없으면 데이터베이스에서 찾는다<br>
따라서 찾으려는 엔티티가 영속성 컨텍스트에 있으면 바로 찾으므로 성능상 이점 있다

> - JPQL 은 항상 데이터베이스를 조회한다
> - JPQL 로 조회한 엔티티는 영속 상태다
> - 영속성 컨텍스트에 이미 존재하는 엔티티가 있으면 기존 엔티티를 반환한다

#### flush
JPA 는 플러시가 일어날 때 영속성 컨텍스트에 변경사항이 있는 엔티티를 찾아서 쿼리를 날린다

JPQL 은 영속성 컨텍스트에 있는 데이터를 고려하지 않고 데이터베이스에서 조회하기 때문에
엔티티가 변경되고나서 플러시하기전에 JPQL 로 조회하면 변경 후의 데이터를 기준으로 엔티티를 조회하게 되면
데이터베이스에는 아직 반영이 안된 엔티티의 데이터가 남아있을 것이기 때문에 변경된 데이터로는 조회할 수 없다

- 플러시 모드
    - FlushModeType.AUTO : 쿼리와 커밋할 때 플러시한다
    - FlushModeType.COMMIT : 커밋 시에만 1번 플러시한다

## 정리
- JPQL 은 SQL 을 추상화해서 특정 데이터베이스에 의존하지 않는다
- QueryDSL, Criteria 는 JPQL 을 만들어주는 빌더 역할을 할 뿐이다 핵심은 JPQL 을 잘 알아야 한다
- 네이티브 SQL 은 최대한 지양하고 가능한 JPQL 을 쓰자
- JPQL 은 대량의 데이터를 수정하거나 삭제하는 벌크연산을 지원한다