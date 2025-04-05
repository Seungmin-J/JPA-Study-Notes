# 2025-03-30 스터디 일지

## 컬렉션 조회할 때
컬렉션 즉시로딩은 항상 외부 조인 사용

---

## 권장 영속성 전이 / 고아 객체 설정
영속성 전이 cascade remove / orphanRemoval 
Cascade.ALL + orphanRemoval = true 쓰는거 권장

---

## 즉시 로딩을 사용하는 경우
N+1 을 즉시 로딩으로 해결하는 방법있다<br>
대부분 지연 로딩 사용하되 필요시 즉시 로딩

---

## CascadeType.REMOVE 와 orphanRemoval = true 차이
- CascadeType.REMOVE : 부모 객체가 삭제되었을 때 자식 객체 삭제
- orphanRemoval = true : 부모 객체와 연관관계가 끊어졌을 때 자식 객체 삭제


