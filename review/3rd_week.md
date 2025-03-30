# 8장 프록시와 연관관계 정리

## 프록시
엔티티를 조회할 때 항상 연관된 엔티티까지 모두 조회하는 것은 비효율적이다
<br>예를 들어서 하나의 팀과 10개의 멤버가 있다고 했을 때
하나의 팀을 조회하면서 10개의 멤버를 조회하게 된다면 낭비란 말

이 문제를 해결하기 위해 실제로 엔티티가 사용되기 전까지 조회를 지연하는 지연 로딩이 있다

지연 로딩의 사용법 
```java
Member member = em.getReference(Member.class, "member1");
```
이렇게 조회한 member 는 실제 객체가 아닌 프록시 객체이다
프록시 객체는 실제 객체를 상속받아 만들어지기 때문에 사용하는 입장에서는 겉 모양이 같음

이때 member.getName() 을 호출하게 되면 프록시 객체를 초기화한다

프록시 객체의 초기화 과정
1. member.getName() 을 호출해서 데이터 조회
2. 프록시 객체는 실제 엔티티가 생성되어 있지 않으면 영속성 컨텍스트에 실제 엔티티 생성을 요청하는데 이것을 초기화라고 한다
3. 영속성 컨텍스트는 데이터베이스를 조회해서 실제 엔티티 객체를 생성한다.
4. 프록시 객체는 생성된 실제 엔티티 객체의 참조를 Member target 멤버 변수에 보관한다
5. 프록시 객체는 실제 엔티티 객체의 getName()을 호출해서 결과를 반환

> 초기화는 영속성 컨텍스트의 도움을 받아야 가능하다
> 따라서 준영속 상태의 프록시를 초기화하면 LazyInitializationException 이 발생

## 즉시 로딩과 지연 로딩

### 즉시 로딩
즉시 로딩 사용법 -> @ManyToOne(fetch = FetchType.EAGER)<br>
엔티티를 조회할 때 연관된 엔티티를 즉시 조회하는 방법

### 지연 로딩
지연 로딩 사용법 -> @ManyToOne(fetch = FetchType.LAZY)<br>
```java
Member member = em.find(Member.class, "member1");
Team team = member.getTeam(); // 반환된 객체는 프록시 객체
team.getName(); // 실제 팀 객체 사용 시에 데이터베이스 조회, 프록시 초기화
```

### JPA 기본 페치 전략

- @ManyToOne, @OneToOne : 즉시 로딩
- @OneToMany, @ManyToMany : 지연 로딩

## 영속성 전이 CASCADE
```java
Parent parent = new Parent();
em.persist(parent);

Child child1 = new Child();
child1.setParent(parent);
parent.getChildren().add(child1);
em.persist(child1);

Child child2 = new Child();
child2.setParent(parent);
parent.getChildren().add(child2);
em.persist(child2);

```
영속성 전이를 사용하지 않으면 각 Child 를 영속화 시켜줘야한다
영속화 시키지 않으면 DB에 저장되지 않을 것
```java
Parent parent = new Parent();
em.persist(parent);

Child child1 = new Child();
child1.setParent(parent);

Child child2 = new Child();
child2.setParent(parent);

parent.getChildren().add(child1);
parent.getChildren().add(child2);
```
영속성 전이를 사용하면 setParent 만 해도 영속화됨

## 고아 객체
```java
// 연관관계 설정 어노테이션 속성을 orphanRemoval = true 로 설정
Parent parent1 = em.find(Parent.class, id);
parent1.getChildren().remove(0); // 자식 엔티티를 컬렉션에서 제거
```

고아 객체 제거가 활성화 되면 Parent 에서 참조하는 Child 컬렉션에서
제거해주기만해도 DB에서 삭제된다

# 9장 값 타입

## 기본 값 타입
말 그대로 기본 타입
- 자바 기본 타입(int, double)
- 래퍼 클래스 (Integer, Long)
- String

## 임베디드 타입(복합 값 타입)
```java
public class Member {
    private Long id;
    private String name;
    
    // 집주소 표현
    private String city;
    private String street;
    private String zipcode;
}
```
위 코드를 말로 설명하려면
- 회원 엔티티는 이름, 도시, 도로명, 우편 번호를 가진다
라고 할 것이다
이것은 단순히 정보를 풀어둔 것으로 다음과 설명하는 것이 더 명확하다
- 회원 엔티티는 이름, 집 주소를 가진다 / 예시는 아래와 같다
```java
public class Member {
    private Long id;
    private String name;
    
    private Address address;
}
```
코드가 훨씬 간결하고 객체지향적이다
이를 임베디드 타입이라고 한다
> @Embeddable: 값 타입을 정의하는 곳에 표시<br>
> @Embedded: 값 타입을 사용하는 곳에 표시

- 임베디드 타입은 기본 생성자가 필수
- 임베디드 타입이 null 이면 매핑한 컬럼 값은 모두 null이 된다

## 값 타입 비교
```java
int a  = 10;
int b  = 10;

Address c = new Address("서울시", "종로구");
Address d = new Address("서울시", "종로구");

// 동등성 비교, 값이 같은지
a.equals(b); 
c.equals(d);

// 동일성 비교
c == d // 다른 인스턴스이기에 거짓
```
- 동일성 비교: 인스턴스의 참조 값을 비교
- 동등성 비교: 인스턴스의 값을 비교, equals() 사용

