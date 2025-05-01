1. 실습 repository → fork 후에 프로젝트 build.gradle 구성
![1.png](imgs%2F1.png)
2. 해당 프로젝트에서 사용할 데이터베이스 생성및 intelij 데이터베이스 연결 (테이블들은 이전 과제의 테스트를 위해 미리 만들어둔 것)
![2.png](imgs%2F2.png)
3. JPA 사용 및 프로젝트 구성을 위한 application.yml 파일 작성
![3.png](imgs%2F3.png)
- 해당 jpa 관련 application.yml 파일의 설정은 Hibernate 원본 옵션을 사용한 것이고 이와 반대로 스프링 자체 옵션을 사용해서 작성할 수 도 있다.


- 스프링 자체 옵션은 파일을 작성하면 스프링이 직접 파싱해서 Hibernate Property로 변환해준다.
가독성이 좋지만, Hibernate에서 제공하는 모든 설정 중 없는 옵션도 존재한다.
    - 예시
        ```yaml
        #스프링 자체 옵션
        jpa:
        show-sql: true
        hibernate:
        ddl-auto: update
        
        #Hibernate 원본 옵션
        jpa:
        properties:
        hibernate:
        show_sql: true
        hbm2ddl.auto: update
        ```

```yaml
 properties:
      hibernate:
        format_sql: true                
        use_sql_comments: true          
        default_batch_fetch_size: 1000  
```

- 해당 옵션들은 스프링 간편 키가 존재하지 않는 옵션들이다.
    - `format_sql` : `show-sql`과 함께 쓰면 줄바꿈·들여쓰기가 들어가서 훨씬 읽기 편한 로그가 나온다
    - `use_sql_comments` : Hibernate가 변환한 SQL 앞에 **`/* entityName */`** 같은 주석을 달아 줘서 “어떤 JPQL이었는지” 추적하기 쉬워진다. 성능변화는 X
    - `default_batch_fetch_size` : 1:N 컬렉션이나 @ManyToOne 프록시를 N+1 대신 IN 절 한방으로 묶어 가져오는 전역 배치 크기. 값이 너무 크면 메모리를, 너무 작으면 왕복 횟수를 잡아먹으니 50 ~ 500 사이부터 점진 조정이 보통이다.


- hdm2ddl option
    - create : 재실행시 기존에 있던 테이블 모두 삭제하고 새로 테이블을 만듦
    - create-drop : 종료시점에 테이블 drop
    - update : 변경분만 반영(운영 DB에는 사용x)
    - vaildate : 엔티티와 테이블이 정상 매핑되었는지만 확인
    - none : 사용하지 않음.
        - 보통의 운영 서버에서는 none으로 둔다. 경험에 의하면, 실제 운영 서버에서 패치시, Entity에 추가적인 변경사항이 많을 때가 있다. update로 두면 서버를 실행할 때 나도 모르는 Hibernate 쿼리가 나가게된다.
        - 따라서 운영서버에서는 none으로 설정하고 직접 쿼리를 날려 테이블을 수정하는 것이 좋다.


4. 작성한 ERD를 바탕으로 Domain Model 설계

- ManyToOne의 Optional ⇒ 자식 클래스의 FK가 비어있어도 되는가?  ⇒ 안되면 false

- 변경사항
    - 추가적으로 기존의 설계보다 더 많은 ManyToMany 관계가 필요함.

      Store ↔ Category : 한 가게가 여러 카테고리를 달 수 있기 때문에 ManyToMany 관계

    - ManyToMany 관계에 primary key를 어떤 것으로 사용할지 많이 고민해봤음.

      굳이 AUTO_INCREAMENT ID를 PrimaryKey로 만들어도 인덱스는 사용불가

    - 복합 Index의 사용은 순서가 중요. 비즈니스 로직을 생각해볼 것.

      보통 음식의 카테고리 같은 경우, 카테고리를 통해 가게를 검색하는 상황이 많다.

      → 즉, 관계 테이블인 StoreCategory에 CategoryID → StoreId의 형태로 인덱스를 걸어두면 해당 상황에서 Index Range Scan이 되므로 DB 응답속도가 빨라진다.


5. 서버 시작 시 Hibernate 쿼리가 날아가는 모습
![5.png](imgs%2F5.png)

- ManyToMany
    - User ↔ Category
    - Store ↔ Category
    - Mission ↔ User

![ERD.png](imgs%2FERD.png)


### List VS Set

- `List`는 **중복 체크를 하지 않으므로 equals/hashCode에 덜 의존**한다. 아직 PK가 없는 신규 엔티티라도 곧바로 add/remove가 가능하다.
- `Set`은 add/remove/contains 할 때마다 **`equals`와 `hashCode`를 호출**해 “이미 있는지” 확인한다. PK가 null(미저장)인 상태에서 중복 여부를 판단하려면 business key + 복잡한 equals 로직이 필요하고, 실수하면 삭제가 안 되거나 성능이 급격히 떨어진
- **HashSet 내부는 hashCode 값으로 버킷을 나누는데**, ‘엔티티 아이디가 생기기 전까지는 hashCode가 거의 일정’해서 모든 요소가 하나의 버킷에 몰린다 → N이 커지면 add/remove 시간이 급등
- `Set`을 쓰면 **중복을 막아 주지만**, 그 **중복 체크 시점**에 컬렉션 전체를 로딩해야 해서 **Lazy-Loading의 장점을 잃어버리게 된다**—바로 이 순간을 “proxy가 강제로 초기화된다”고 표현한다. Situational 성능·도메인 요구를 보고 List(=bag)와 Set을 선택하자!