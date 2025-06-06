# 웹 애플리케이션과 영속성 관리

## 트랜잭션 범위의 영속성 컨텍스트
스프링 컨테이너는 트랜잭션 범위의 영속성 컨텍스트 전략을 기본으로 사용한다
- 트랜잭션을 시작할 때 영속성 컨텍스트를 생성하고 트랜잭션이 끝날 때 영속성 컨텍스트를 종료한다
- 같은 트랜잭션 안에서는 항상 같은 영속성 컨텍스트에 접근한다

## 준영속 상태와 지연 로딩

준영속 상태에서는 지연 로딩이 작동하지 않는다
예를 들면 컨트롤러 레이어에서 뷰를 렌더링할 때 엔티티를 지연로딩 설정해서 프록시 객체로 조회 -> 초기화하지 않은 프록시 객체 사용 시 실제 데이터를 불러오려고 초기화 기도
<br> 하지만 준영속 상태는 영속성 컨텍스트가 없으므로 지연로딩 불가

<br> 준영속 상태의 지연 로딩 문제를 해결하는 방법은 2가지
- 뷰가 필요한 엔티티를 미리 로딩해두는 방법
- OSIV 를 사용해서 엔티티를 항상 영속 상태로 유지하는 방법

### 뷰가 필요한 엔티티를 미리 로딩해두는 방법
- 글로벌 페치 전략 수정
- JPQL 페치 조인
- 강제로 초기화

#### 글로벌 페치 전략
글로벌 페치 전략의 단점 : 사용하지 않는 엔티티 로딩, N+1 문제 발생

#### JPQL 페치 조인
무분별하게 사용 시 화면에 맞춘 리포지토리 메소드가 증가한다
결국 프리젠테이션 계층이 데이터 접근 계층을 침범하는 것임
#### 강제로 초기화

```java
@Transactional
public Order findOrder(id) {
    Order order = orderRepository.findOrder(id);
    order.getMember().getName(); // 프록시 객체 강제로 초기화
    return order;
}
```

이처럼 프록시 초기화 역할을 서비스 계층이 담당하면 뷰가 필요한 엔티티에 따라 서비스 계층의 로직을 변경해야한다 <br>
프리젠테이션 계층이 서비스 계층을 침범하는 상황.

따라서 비즈니스 로직을 담당하는 서비스 계층에서 프록시 객체를 초기화하는 역할을 분리해야함 <br>
FACADE 계층이 그 역할을 담당

#### FACADE 계층 추가

### OSIV
OSIV(Open Session in View) 는 영속성 컨텍스트를 뷰까지 열어둔다는 뜻
영속성 컨텍스트가 살아있으면 엔티티는 영속 상태로 유지된다 따라서 뷰에서도 지연 로딩을 사용할 수 있다

#### 과거 OSIV: 요청당 트랜잭션
요청이 들어오면 필터나 인터셉터에서 영속성 컨텍스트를 만들면서 트랜잭션 시작, 요청이 끝날 때 트랜잭션과 영속성 컨텍스트를 함께 종료
FACADE 계층 없이도 뷰에 독립적인 서비스 계층 유지 가능

**요청 당 트랜잭션 방식의 OSIV 문제점**
프리젠테이션 계층이 엔티티를 변경할 수 있다는 점

#### Spring OSIV
요청 당 트랜잭션 방식의 문제점인 프리젠테이션 계층에서 엔티티를 변경할 수 있다는 점을 보완.
요청 시에 영속성 컨텍스트를 만들고 트랜잭션은 서비스 계층에서 시작, 비즈니스 로직 수행 후 트랜잭션이 종료되면
프리젠테이션 계층으로 오는데 여기서는 엔티티의 수정이 불가능. 트랜잭션이 끝났기 때문에.

**엔티티간 복잡한 조인을 통해 데이터를 전달해야하는 경우 DTO 를 반환하는 것이 더 나을 수 있다**

- 프리젠테이션 계층에서 엔티티를 수정하고나서 비즈니스 로직 수행 시 엔티티가 수정될 수 있다
- 같은 영속성 컨텍스트를 여러 트랜잭션이 공유할 수 있다



