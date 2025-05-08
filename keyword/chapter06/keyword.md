### 지연로딩과 즉시로딩의 차이
둘의 핵심 차이점은 연관된 데이터를 불러오는 시점

1. 즉시로딩
    - 연관된 데이터를 즉시 함께 가져온다.
   ```java
    @Entity
    public class Member {
        @ManyToOne(fetch = FetchType.EAGER)
        private Team team;
    }
    ```
   - Member를 가져오면 연관된 Team도 즉시 가져온다. SQL은 조인을 포함한 쿼리로 실행된다.
   - 장점 
     - 데이터를 미리 로딩하므로 추가 쿼리가 발생하지 않는다.  
     - 쿼리 수 예측이 가능하다.
   - 단점
     - 항상 연관 데이터를 불러오므로 성능 낭비가 될 수 있다.
     - 불필요한 데이터까지 함께 가져올 수 있음.

2. 지연로딩
    - 연관된 데이터를 필요할 때 가져온다.
    - 프록시 객체로 대기하다가 실제 사용 시 쿼리 실행
    ```java
    @Entity
    public class Member { 
        @ManyToOne(fetch = FetchType.LAZY)
        private Team team;
    }
    ```
    member.getTeam()을 호출하는 순간 Team을 로딩한다.
    - 장점
        - 필요할 때만 쿼리 실행 -> 성능 효율적
        - 초기 로딩시 빠름
    - 단점
      - N+1 문제 발생 가능
      - 트랜잭션 범위 밖에서 접근하면 LazyInitializationException 발생 가능


### Fetch Join
- Fetch Join은 JPA에서 지연 로딩되는 연관 엔티티를 SQL의 Join을 통해 한 번에 로딩하는 방법이다.
    지연 로딩으로 설정되어 있더라도, JPQL에서 Fetch Join을 사용하면 연관 엔티티를 즉시 로딩한다.


- 예시
    ```java
    @Entity
    public class Member {
        @Id
        private Long id;
  
        @ManyToOne(fetch = FetchType.LAZY)
        private Team team;  
    }
    ```

    일반 JPQL
    ```java
     List<Member> members = em.createQuery("SELECT m FROM Member m", Member.class).getResultList();
    ```
    Members.get(0).getTeam().getName() 호출 시 그때 Team 쿼리가 발생 (지연 로딩)

    <br>
  
    Fetch Join 이용
    ```java
    List<Member> members = em.createQuery("SELECT m FROM Member m JOIN FETCH m.team", Member.class).getResultList();
    ```
    Member와 Team을 한 번의 쿼리로 로딩하고, 이후에는 Team을 따로 조회하지 않음



### EntityGraph
@EntityGraph 는 JPA에서 N+1 문제를 해결하거나 Fetch Join을 대체하기 위해 사용하는 JPQL 성능 최적화 기능이다.  
주로 연관된 엔티티를 Fetch Join처럼 함께 로딩하고 싶을 때 사용되며, 특히 동적 쿼리를 쓰지 않고도 즉시 로딩을 유도할 수 있다는 점에서 유용하다.

- 예시
  ```java
    @Entity
    public class Member {
        @Id
        private Long id;

        @ManyToOne(fetch = FetchType.LAZY)
        private Team team;
    }
  ```
  ```java
    public interface MemberRepository extends JpaRepository<Member, Long> {
        @EntityGraph(attributePaths = {"team"})
        @Query("SELECT m FROM Member m WHERE m.name = :name")
        List<Member> findByName(@Param("name") String name);
    }
  ```

- 위의 예시는 JOIN FETCH m.team처럼 동작한다.
- Join Fetch를 많이 쓰다보면 JPQL이 복잡해지고 가독성이 떨어진다. 이 때 @EntityGraph를 사용해서 리팩토링 가능


### JPQL
JPQL은 SQL과 비슷하지만 DB 테이블이 아니라 엔티티와 필드를 대상으로 쿼리

SQL : SELECT * FROM member WHERE name = '홍길동'

JPQL : SELECT m FROM Member m WHERE m.name = '홍길동'

JPQL에서 Member는 테이블명이 아닌 자바 클래스, m.name은 클래스의 필드명


### QueryDSL
QueryDSL은 타입 안전한 SQL/JPA 쿼리를 자바 코드로 작성할 수 있도록 해주는 프레임워크.

컴파일 시점에 문법 오류를 잡아주며 가독성이 좋고 유지보수하기 쉬운 쿼리 작성을 도와준다.

사용하기 위해서는 Q 클래스를 미리 생성하는 등의 세팅이 필요하다