# 10장 객체지향 쿼리 소개

## 객체지향 쿼리 종류
- JPQL
- Criteria : JPQL 을 편하게 작성해주는 API. 빌더 클래스 모음
- QueryDSL : Criteria 쿼리처럼 JPQL 을 편하게 작성하도록 도와주는 빌더 클래스 모음
- 네이티브 SQL : JPA 에서 JPQL 대신 직접 SQL 을 사용할 수 있다
- 객체지향 쿼리 심화

- ### JPQL
JPQL 은 엔티티 객체를 조회하는 객체지향 쿼리이다
JPQL 은 SQL 을 추상화해서 특정 데이터베이스에 의존하지 않는다

- ### Criteria
프로그래밍 코드로 JPQL 을 작성할 수 있다(오타 방지)
고로 컴파일 시점에 오류를 발견할 수 있다
너무 복잡하고 장황하다 사용하기 불편하다 

- ### QueryDSL
Criteria 처럼 JPQL 빌더 역할을 한다
어노테이션 프로세서를 사용해서 쿼리 전용 클래스를 만들어야한다
QClass <- 쿼리 전용 클래스

- ### 네이티브 SQL
JPA 는 SQL 을 직접 사용할 수 있는 기능을 지원하는데 이것을 네이티브 SQL 이라한다.
JPQL 이 지원하지 않는 기능을 사용하기 위해서 네이티브 SQL 을 사용한다
단점은 특정 데이터베이스에 의존하는 SQL 을 작성해야 한다는 것

- ### JDBC 직접 사용, 마이바티스 같은 SQL 매퍼 프레임워크 사용

## JPQL 
앞서 다양한 방법이 있었지만 **어떤 방법을 사용하든 JPQL 에서 모든 것이 시작한다**
- JPQL 은 객체지향 쿼리 언어이다. 테이블을 대상으로 쿼리하는 것이아니라 엔티티 객체를 대상으로 한다
- 특정 데이터베이스에 의존하지 않는다
- JPQL 은 결국 SQL 변환된다
- 대소문자 구분하지 않는다
- 엔티티 명은 기본값인 클래스 명을 엔티티 명으로 사용하는 것을 추천한다
- 별칭은 필수

### 조인
- 내부 조인
INNER JOIN 을 사용한다 INNER 는 생략가능
- 외부 조인
LEFT (OUTER) JOIN - OUTER 생략 가능
- 세타 조인
```java
// 회원 이름이 팀 이름과 똑같은 사람 수를 구하는 예

// JPQL
select count(m) from Member m, Team t
where m.username = t.name

// SQL
SELECT COUNT(M.ID)
FROM
    MEMBER M CROSS JOIN TEAM T
WHERE
    M.USERNAME=T.NAME
```

### JOIN ON 절
```java
// JOIN ON 사용 예

// JPQL
select m,t from Member m
left join m.team t on t.name = 'A'

// SQL
SELECT m.*, t.* FROM Member m
LEFT JOIN Team t ON m.TEAM_ID=t.id and t.name='A'
```
연관관계가 필요없다.
- 일반 조인은 객체지향적이며 연관관계 기반
- JOIN ON 절은 연관관계 없이 사용, 복잡할 때 사용


### 페치 조인
SQL 에서의 조인 종류가 아님 JPQL 에서 성능 최적화를 위한 기능
연관된 엔티티나 컬렉션을 한 번에 같이 조회하는 기능이다
```java
// JPQL
select m
from Member m join fetch m.team

// SQL
SELECT M.*, T.*
FROM MEMBER T
INNER JOIN TEAM T ON M.TEAM_ID=T.ID
```

멤버와 팀이 지연로딩 설정된 상태에서 페치 조인하면 연관된 엔티티 모두 로딩
영속성 컨텍스트에 들어감

### 페치 조인과 일반 조인의 차이
```java
// 내부조인 JPQL
select t
from Team t join t.members m
        where t.name = '팀A'

// SQL
SELECT T.*
FROM TEAM T
INNER JOIN MEMBER M ON T.ID=M.TEAM_ID
WHERE T.NAME = '팀A'
```
일반 조인은 결과를 반환할 때 연관관계까지 고려하지 않는다
단지 SELECT 절에 지정한 엔티티만 조회할 뿐이다

반면 페치 조인은 연관된 엔티티를 전부 조회한다
- 페치 조인 대상에는 별칭을 줄 수 없다
- 둘 이상의 컬렉션을 페치할 수 없다

### 경로 표현식
 > . 을 찍어 객체 그래프를 탐색하는 것

용어정리
- 상태 필드 - String, int 등, 경로 탐색의 끝
- 연관 필드
  - 단일 값 연관 필드(엔티티) - 묵시적으로 내부 조인, 단일 연관 경로는 계속 탐색 가능
  - 컬렉션 값 연관 필드(컬렉션) - 묵시적으로 내부 조인, 더는 탐색 불가

```java
// 명시적 조인 : JOIN 을 직접 적어주는 것
SELECT m FROM Member m JOIN m.team t

// 묵시적 조인 : 경로 표현식에 의해 묵시적으로 조인이 일어나는 것, 내부 조인만 할 수 있음
SELECT m.team FROM Member m
```

경로 탐색을 사용한 묵시적 조인 시 주의사항
- 항상 내부 조인
- 컬렉션은 경로 탐색의 끝이다 컬렉션에서 경로 탐색을 하려면 명시적으로 조인해서 별칭을 얻어야 한다

> 묵시적 조인은 조인이 일어나는 상황을 한눈에 파악하기 어렵다
> 단순하고 성능에 이슈가 없으면 괜찮지만 성능이 중요하면 분석하기 쉽게 묵시적 조인보다 **명시적 조인을 사용**하자

### 서브 쿼리
JPQL 은 SQL 과 다르게 WHERE, HAVING 절에서만 사용가능

- EXISTS : 서브쿼리에 결과 존재하면 참
- (ALL/ANY/SOME) : ALL - 조건 모두 만족 시 참, ANY 혹은 SOME - 하나라도 만족하면 참
- IN : 서브쿼리의 결과 중 하나라도 같은 것이 있으면 참

### 엔티티 직접 사용
JPQL 에서 엔티티 객체를 직접 사용하면 SQL 에서는 해당 엔티티의 기본 키 값을 사용한다
```java
select count(m.id) from Member m // 엔티티의 아이디를 사용
select count(m) from Member m    // 엔티티를 직접 사용
```

SQL 로 변환될 때 해당 엔티티의 기본 키를 사용하기 때문에 실제 실행된 둘의 SQL 은 같다

엔티티를 파라미터로 직접 받는 코드
```java
String qlString = "select m from Member m where m = :member";
List resultList = em.createQuery(qlString)
        .setParameter("member", member)
        .getResultList();
```
실행된 SQL 은 다음과 같다
```
select m.*
from Member m
where m.id=?
```