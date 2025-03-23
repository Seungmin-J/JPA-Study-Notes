# 1장 - JPA 소개

## SQL을 직접 다룰 때 문제점
JPA 이전에는 JDBC API 를 통해서 작성한 SQL 을 데이터베이스에 전달하여 실행했다

이 방법의 문제점은 크게 3가지가 있다

1. 진정한 의미의 계층 분할이 어렵다.
2. 엔티티를 신뢰할 수 없다.
3. SQL에 의존적인 개발을 피하기 어렵다.

이게 무슨 말인가? 한 번 알아보자

---
```java
public class Member {
    private String memberId;
    private String name;
    
}

public class MemberDAO {
    public Member find(String memberId) {...}
}
```

위와 같은 회원 엔티티와 회원용 DAO(데이터 접근 객체)가 있다

DAO의 find 메서드로 회원을 조회하는 기능을 개발하려고 한다

순서는 다음과 같다

1. 회원 조회용 SQL을 작성한다

```sql
SELECT member_id, name FROM member m WHERE member_id = ?
```

2. JDBC API 를 사용해서 SQL 을 실행한다
```java
ResultSet rs = stmt.executeQuery(sql);
```
3. 조회 결과를 Member 객체로 매핑한다
```java
String memberId = rs.getString("member_id");
String name = rs.getString("name");

Member member = new Member();
member.setMemberId(memberId);
member.setName(name);
```

조회 기능을 만들었으니 회원 저장 기능도 만들어볼까?

```java

public class MemberDAO {
    public Member find(String memberId) {...}
    public void save(Member member) {...} // 회원 저장 기능 추가
}
```

1. 회원 등록용 SQL 작성
```java
String sql = "INSERT INTO member(member_id, name) VALUES(?, ?)";
```
2. 회원 객체의 값을 꺼내서 등록 SQL 에 전달
```java
pstmt.setString(1, member.getMemberId());
pstmt.setString(2, member.getName());
```
3. JDBC API 를 사용해서 SQL 을 실행한다
```java
pstmt.executeUpdate(sql);
```
자 이제 회원 조회, 저장 기능이 만들어졌다

만들고나니 회원의 연락처를 함께 저장해달라는 개발 요구사항이 생겼다

```java
public class Member {
    private String memberId;
    private String name;
    private String tel; // 추가됨
}

```

엔티티에 필드를 추가해줬다면 테이블에도 tel 컬럼을 추가해야한다

```sql
ALTER TABLE member
ADD COLUMN tel VARCHAR(255);
```

INSERT SQL 도 수정하고 회원 객체의 연락처 값을 SQL 에 매핑해야한다
```java
String sql = "INSERT INTO member(member_id, name, tel) VALUES(?,?,?)";
pstmt.setString(3, member.getTel());
```

회원 조회 코드도 수정해야한다
```java
String sql = "SELECT member_id, name, tel FROM member WHERE member_id = ? " 
        
String tel = rs.getString("tel");
member.setTel(tel);
```

여기서 만약 회원의 소속팀까지 추가해달라는 요구사항이 생긴다면 위의 과정과 비슷한 반복이 필요하다</br>
Team 이라는 참조형 객체가 추가되기 때문에 회원, 팀 테이블을 JOIN 해야하고 더 귀찮은 작업이 될 것이다

이처럼 위에서 말한 3가지

데이터 접근 계층을 사용하지만 직접 DAO 를 열어서 SQL 을 확인할 수 밖에 없는, <br>
비즈니스 요구사항을 모델링한 객체인 엔티티를 신뢰하지 못하는, <br>
SQL에 의존적인 개발을 피하기 어려운 <br>

문제가 발생한다

## 패러다임의 불일치

객체지향 프로그래밍은 추상화, 캡슐화, 정보은닉, 상속, 다형성 등 시스템의 복잡성을 제어할 수 있는<br>
다양한 장치들을 제공한다
비즈니스 요구사항을 정의한 도메인 모델도 객체로 모델링하면 객체지향 언어가 가진 장점들을 활용할 수 있다<br>

객체는 속성(필드)과 기능(메서드)를 가진다
객체의 기능은 클래스에 정의되어 있으므로 객체 인스턴스의 상태인 속성만 저장헀다가 필요할 때 불러와서 복구하면 된다<br>
하지만 객체를 상속받거나 다른 객체를 참조하고 있다면 객체의 상태를 저장하기는 쉽지 않다

데이터베이스는 데이터 중심으로 구조화되어있다. 그리고 객체지향의 추상화, 상속, 다형성 같은 개념이 없다<br>
객체와 관계형 데이터베이스는 지향하는 목적이 서로 다르므로 둘의 기능과 표현 방법도 다르다<br>
이것을 객체와 관계형 데이터베이스의 패러다임 불일치 라고 한다

개발자는 데이터베이스와 객체간에 이 패러다임의 불일치를 해결해야한다

## 패러다임의 불일치 예시
### 1. 상속
가령 Item 객체와 Item 객체를 상속받는 Book 객체가 있다고 해보자<br>
Book 객체를 저장하기 위해서는 두 SQL 이 필요하다
### 2. 연관관계
객체는 참조를 통해 다른 객체에 접근해서 연관된 객체를 조회한다<br>
덕분에 자유롭게 객체 그래프를 탐색할 수 있다<br>
반면에 테이블은 외래 키를 사용해서 다른 테이블과 연관관계를 가지고 조인을 사용해서 연관된 테이블을 조회한다<br>
SQL 을 직접 다루면 실행하는 SQL 에 따라 객체 그래프의 탐색 범위가 정해진다<br>
객체 그래프의 범위를 확인하기 위해서는 DAO 에 작성된 SQL 을 열어봐야 알 수 있다

## JPA 을 사용해야하는 이유
JPA 는 이처럼 패러다임의 불일치를 해결하기 위한 개발자의 수많은 노력을 줄여준다

### 1. 생산성 - SQL, JDBC API 를 사용하지 않고 자바 컬렉션에 저장하듯이 저장, 그리고 조회가 가능, DDL 도 자동생성 
### 2. 유지보수 - 엔티티에 필드 하나만 추가되어도 CRUD 관련 SQL 을 모두 수정해야 했으나 JPA가 이를 대신함
### 3. 패러다임 불일치 해결
### 4. 성능 - 영속성 컨텍스트로 쿼리 실행 횟수를 줄임
### 5. 데이터 접근 추상화와 벤더 독립성 - 각 벤더사의 관계형 데이터베이스마다 사용법이 다른 문제를 <br> 추상화를 통해 특정 데이터베이스종속되지 않게됨 


# 3장 - 영속성 관리

## 영속성 컨텍스트의 장점
- 1차 캐시
- 동일성 보장
- 트랜잭션을 지원하는 **쓰기 지연**
- 변경 감지
- 지연 로딩

### 엔티티 조회
영속성 컨텍스트에는 Map 이 존재
Map 의 key 는 엔티티의 id, value 는 엔티티 객체다<br>
영속성 컨텍스트에 데이터를 조회하고 저장하는 기준은 데이터베이스의 기본 키 값 (ID)

| @Id | Entity |
|-----|--------|
| 1   | member |

### 1차 캐시에서 우선 조회
em.find() 를 호출하면 1차 캐시에서 식별자 값으로 엔티티를 찾음<br>
찾는 엔티티가 있으면 데이터베이스에서 조회하지 않고 1차 캐시의 엔티티를 조회<br>

1차 캐시에 찾는 엔티티가 없으면 데이터베이스에서 조회 후 1차 캐시에 저장한 뒤 엔티티를 반환함
<br> 데이터베이스에 접근하지 않아도 되니 성능상 이점을 얻음

### 동일성 보장
```java
Member a = em.find(Member.class, 1);
Member b = em.find(Member.class, 1);
System.out.println(a == b); // 동일성 비교
```
em.find(Member.class, 1) 를 반복하여도 1차 캐시에 저장된 같은 인스턴스를 반환한다<br>
따라서 동일성 비교의 결과는 true 

**영속성 컨텍스트는 성능상 이점과 엔티티의 동일성을 보장한다**

* 참고) 동등성은 다른 인스턴스이지만 같은 값을 가지는 특성을 의미 <br> ex) equals() 

### 쓰기 지연
트랜잭션을 커밋하기 전까지 SQL 을 쓰기 지연 저장소에 모아둔다<br>
엔티티 매니저가 flush 할 때 한 번에 모든 쿼리를 실행함으로<br>
성능을 최적화할 수 있다

### 엔티티 수정
멤버의 이름과 나이를 수정하는 기능을 개발했다. 이후에 등급을 바꾸는 기능이 추가되었다<br>
그러면 등급을 수정하는 SQL 도 따로 작성해야함, 결국 SQL 에 의존하는 개발을 하게됨

### 변경 감지<br>
JPA 는 엔티티를 영속성 컨텍스트에 저장할 때 최초 상태를 복사해서 저장, 이를 스냅샷 이라고함<br>

**변경 감지 순서**
1. 트랜잭션 커밋하면 플러시 호출
2. 엔티티와 스냅샷 비교해서 변경된 엔티티 조회
3. 변경된 엔티티가 있으면 수정 쿼리 생성 후 쓰기 지연 저장소에 보낸다
4. 쓰기 지연 저장소의 SQL 을 데이터베이스에 보낸다
5. 데이터베이스 트랜잭션을 커밋한다

**JPA 의 변경 감지 후 수정 쿼리 생성 기본 전략**<br>

엔티티의 모든 필드를 업데이트한다<br>
모든 필드를 업데이트하는 SQL 을 애플리케이션 실행 시점에 미리 생성해두고 재사용할 수 있다

