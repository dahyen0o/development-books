### CQRS 패턴

명령(Command) 모델과 조회(Query) 모델을 분리하는 패턴이다.

- 명령 모델: 상태를 변경하는 기능을 구현할 때 사용
- 조회 모델: 데이터를 조회하는 기능을 구현할 때 사용

<br>

### 조회 모델을 구현하는 방법

#### 스펙(Specification)

애그리거트가 특정 조건을 충족하는지를 검사할 때 사용하는 인터페이스이다. <br>
*검색 조건을 다양하게 조합*해야 할 때 사용한다.

```java
public interface Specification<T> {
  Predicate toPredicate(Root<T> root, CriteriaQuery<?> query, CriteriaBuilder builder);
}
```

Spring Data JPA에서 제공하는 위 인터페이스를 이용해 다음과 같이 사용한다.

```java
public class OrdererIdSpec implements Specification<OrderSummary> {
  private String ordererId;

  @Override
  public Predicate toPredicate(Root<OrderSummary> root, CriteriaQuery<?> query, CriteriaBuilder builder) {
    return builder.equal(
          root.get(OrderSummary_.ordererId), // OrderSummary_는 JPA 정적 메타 모델이다.
          ordererId
    );
}

public interface OrderSummaryDao extends Repository<OrderSummary, String> {
  List<OrderSummary> findAll(Specification<OrderSummary> spec);
}
```

##### 스펙 빌더 클래스

복잡한 조건에 따라 스펙을 조합해야 할 때 사용한다.

```java
Specification<MemberData> spec = SpecBuilder.builder(MemberData.class)
    .ifTrue(searchRequest.method1(), () -> MemberDataSpecs.method1())
    .ifHasText(searchRequest.method2(), name -> MemberDataSpecs.method2(name))
    .toSpec();

List<MemberData> result = memberDataDao.findAll(spec, PageRequest.of(0, 5));
```

메서드 호출 체인으로 연속된 변수 할당을 줄여 코드 가독성을 높이고 구조를 단순화시킨다.

#### 정렬

- 메서드 이름에 `OrderBy` 를 사용해 정렬 기준 지정
```java
public interface OrderSummaryDao extends Repository<OrderSummary, String> {
  List<OrderSummary> findByOrdererIdOrderByNumberDesc(String orderId);
}
```

- `Sort` 를 인자로 전달
```java
public class Service {
  // 생략

  public void method() {
    Sort sort = Sort.by("number").ascending();
    List<OrderSummary> results = orderSummaryDao.findAll(spec, sort);
  }
}

public interface OrderSummaryDao extends Repository<OrderSummary, String> {
  List<OrderSummary> findAll(Specification<OrderSummary> spec, Sort sort);
}
```

#### 페이징

전체 데이터 중 일부만 보여주는 처리다.

```java
public class Service {
  // 생략

  public void method() {
    Sort sort = Sort.by("number").ascending();
    PageRequest pageReq = PageRequest.of(1, 10, sort);
    List<MemberData> results = memberDataDao.findByNameLike("사용자%", pageReq);
  }
}

public interface MemberDataDao extends Repository<MemberData, String> {
  Page<MemberData> findByNameLike(String name, Pageable pageable);
}
```

레포지터리 메서드의 리턴 타입으로 Page를 사용하면 `추가 COUNT 쿼리` 를 실행해서 조건에 해당하는 데이터 개수를 구한다.

#### Hibernate의 @Subselect

쿼리 결과를 @Entity로 매핑할 수 있는 유용한 기능이다.

- @Immuatble: 해당 엔티티를 immutable(불변)하게 만든다.
- @Synchronize: 추가한 테이블에 변경이 발생하면 즉시 `flush` 를 하여, 변경 내역이 @Subselect의 엔티티에 반영된다.

```java
@Entity
@Immutable
@Subselect(
      """select o.order_number as number,
          o.version,
          o.orderer_id,
          o.orderer_name,
          o.total_amounts,
          p.product_id
          from purchase_order o inner join ... // 생략
      """
)
@Synchronize({"purchase_order", "order_line", "product"})
public class OrderSummary {
    @Id private String number;
    @Column private long version;
    @Column(name = "orderer_id") private String ordererId;
    // 생략
}
```
