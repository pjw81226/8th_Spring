# AOP를 이용한 Pageable 보정

실무에서 그렇게 많이 쓰이는 방식은 아니다. 하지만 단순하게 “이런 방법도 있구나” 하고 넘어갈만한 부분

Pageable은 Client로부터 받는다. 내부의 값은 음수 페이지, 존재하지 않는 column을 이용한 정렬 등과 같이 예측불가능한 값이 들어올 가능성이 존재한다.

이러한 값들을 Controller에서 일을 처리하기 이전에 AOP와 Around를 이용해서 어느정도 보정이 가능하다.

```java
    private final ConcurrentMap<Class<?>, Set<String>> validFieldsCache = new ConcurrentHashMap<>();

    @Around("@annotation(pageableConstraint)")
    public Object pageableValidation(@NonNull ProceedingJoinPoint joinPoint, @NonNull PageableConstraint pageableConstraint) throws Throwable {
        Object[] args = joinPoint.getArgs();
        Class<?>[] entityClasses = pageableConstraint.value();

        for (int i = 0; i < args.length; i++) {
            if (args[i] instanceof Pageable pageable) {
                Sort sort = pageable.getSort();

                int pageNumber = pageable.getPageNumber() <= 0 ? 1 : pageable.getPageNumber();

                // 기본 정렬 조건 설정
                if (sort.isUnsorted()) {
                    sort = Sort.by(Sort.Direction.DESC, "id");
                }

                if (sort.isSorted() && entityClasses.length > 0) {
                    List<Sort.Order> validOrders = sort.stream()
                            .filter(order -> isValidOrder(order, entityClasses))
                            .toList();
                    args[i] = PageRequest.of(pageNumber - 1, pageable.getPageSize(), Sort.by(validOrders));
                } else {
                    args[i] = PageRequest.of(pageNumber - 1, pageable.getPageSize(), sort);
                }
            }
        }
        return joinPoint.proceed(args);  // 수정된 인자를 전달하고 결과를 반환
    }

    private boolean isValidOrder(@NonNull Order order,  Class<?> @NonNull [] entityClasses) {
        String field = order.getProperty();
        for (Class<?> entityClass : entityClasses) {
            if (isValidField(entityClass, field)) {
                return true;
            }
        }
        return false;
    }

    private boolean isValidField(Class<?> entityClass, String field) {
        Set<String> validFields = validFieldsCache.computeIfAbsent(entityClass, this::getEntityFields);
        return validFields.contains(field);
    }

    private @NotNull Set<String> getEntityFields(@NonNull Class<?> entityClass) {
        Set<String> fields = new HashSet<>();
        for (Field field : entityClass.getDeclaredFields()) {
            fields.add(field.getName());
        }
        return fields;
    }
```

간단하게 작성해본 예시다. 정렬 조건이 설정되어있지 않은경우  primary key를 이용해서 정렬한다.

⇒ 하지만 이부분은 복합키를 primary key로 사용하는 부분도 존재하므로 문제 가능성이 있다.

음수의 페이지 값이 들어오면 강제로 1로 만들어준다.

별다른 처리 없이 음수의 페이지 값이 들어올 경우 500에러가 발생하거나 취약점이 드러날 수 있다.

하지만 실무에서 사용하지 않는 이유가 있다.

체크할 내용이 page, size, sort 범위 제한 뿐인데 프록시까지 쓰면서 해결해야할 부분은 아니기때문.

뿐만아니라  AOP로 예외를 던지면 스택트레이스가 프록시 경유 경로라 처음 보는 개발자가 원인을 추적하기가 까다롭다

# Spring Data JPA의 Paging

대량의 데이터를 테이블 전체 조회 없이 한 번에 n 건만 가져오도록 도와주는 기능.  
Spring Data JPA는 페이징을 위해 Pageable, Page, Slice 인터페이스를 제공한다.

## Page
Page는 요청한 페이지 데이터뿐 아니라, 전체 레코드 수, 전체 페이지 수, 현재 페이지 번호, 다음 페이지 여부 등 모든 페이징 관련 정보를 포함하는 객체이다.

```java
public interface Page<T> extends Slice<T> {
    int getTotalPages();          // 전체 페이지 수
    long getTotalElements();      // 전체 데이터 수
    <U> Page<U> map(Function<? super T, ? extends U> converter); // 변환
}
```

레포지토리의 메소드를 이용해서 DB를 조회하는 즉시 두개의 쿼리가 발생한다.
1. 데이터 조회 쿼리: LIMIT / OFFSET을 포함해서 해당 페이지의 데이터 10건을 가져오는 쿼리
2. 카운트 쿼리: 전체 데이터 개수를 구하는 SELECT COUNT(*) 쿼리

그렇다면 조회쿼리는 알겠는데 전체 데이터 개수를 구하는 쿼리는 왜 날아가는가?  
이유는 전체 페이지 수와 전체 데이터 수를 알아야 하기 때문이다.  
Page<T>는 단순히 현재 페이지의 데이터만 제공하는 것이 아니라, 다음 정보를 추가로 제공한다: 
- 총 데이터 수	getTotalElements()
- 전체 페이지 수	getTotalPages()
- 현재 페이지 번호	getNumber()
- 다음 페이지 존재 여부	hasNext() 
- 마지막 페이지 여부	isLast()  

이 중 getTotalElements()와 getTotalPages()를 계산하려면 DB에 전체 몇 건이 있는지를 반드시 알아한다.

## Slice
Slice<T>는 Spring Data JPA에서 페이징 처리 시 사용되는 결과 타입 중 하나로, 다음의 목적을 위해 사용된다.
> "전체 데이터 개수나 전체 페이지 수는 몰라도 되고, 현재 페이지의 데이터와 다음 페이지가 있는지만 알면 된다."

즉, 가볍고 빠른 페이징이 필요한 경우 사용한다..

특징

- 전체 데이터 수 제공 -> 제공하지 않음 (getTotalElements() 없음)
- 전체 페이지 수 제공 -> 제공하지 않음 (getTotalPages() 없음)
- 현재 페이지 데이터 제공 ->	getContent()
- 다음 페이지 존재 여부 -> hasNext()
- 쿼리 횟수 -> 1번 (데이터 조회 쿼리만 실행)

동작방식  
- 가져올 데이터보다 1개 더 가져와서 hasNext를 판단한다.
- 예를들어,  10개 요청했다고 하면, 내부적으로는 11개를 조회한다.
    - 11개 중 앞의 10개를 getContent()로 보여주고,
      11번째 데이터가 존재하면 → hasNext() == true 로 판단
- 무한 스크롤 UI에 매우 적합




# 객체 그래프 탐색
객체 간의 연관관계를 통해, 하나의 엔티티에서 다른 엔티티를 참조하며 탐색하는 것을 말한다.  
DB의 테이블을 직접 조인하는 대신, 객체 참조를 이용해 연관된 객체에 접근하는 방식이다.

## 예시
```java
@Entity
public class Member {
    @Id @GeneratedValue
    private Long id;

    private String name;

    @ManyToOne(fetch = FetchType.LAZY)
    private Team team; // 연관관계 필드
}
```

```java
@Entity
public class Team {
    @Id @GeneratedValue
    private Long id;

    private String name;
}
```

```java
Member member = em.find(Member.class, 1L);

// 객체 그래프 탐색
String teamName = member.getTeam().getName(); // member → team 참조 → team.name 접근
```
### 이때 member.getTeam()은?
JPA는 member.team 필드를 지연로딩 프록시로 가지고 있음  
getTeam()을 처음 호출하는 시점에 실제 DB 쿼리를 날림  
이를 **지연 로딩** 이라 하고, 이를 이용한 탐색을 객체 그래프 탐색이라 한다.

