# 미션 인증
1번 특정 지역에 가게 추가하기 API  
![request1.png](imgs%2Frequest1.png)
![response1.png](imgs%2Fresponse1.png)

<br>
<br>

2번 가게 리뷰추가 API
![response1.png](imgs%2Fresponse1.png)
![response2.png](imgs%2Fresponse2.png)

<br>
<br>

3번 가게에 미션 추가하기
![request3.png](imgs%2Frequest3.png)
![response3.png](imgs%2Fresponse3.png)

<br>
<br>

4번 가게의 미션을 도전 중인 미션에 추가(미션 도전하기) API
![request4.png](imgs%2Frequest4.png)
![lastresponse.png](imgs%2Flastresponse.png)



# 생각해볼점
1. 복합키 저장  
   복합키를 primary key로 하는 엔티티를 저장할때 JpaRepository.save()를 사용하면 에러 발생.

   - 왜?
     - jpaRepository.save()는 EntityManger.merge()로 구현되어있음
     - 복합 PK 엔티티는 테이블에 저장하기 이전에 미리 Entity에 PK를 주입시켜놓은 상태  
       즉, save()는 이미 아이디가 있는 것을 보고 테이블에 이미 존재하는 엔티티라 생각하고 merge()를 호출함
     - merge()는 새로운 ‘관리되는 복제본’을 만들어서 상태를 복사한 뒤 그 인스턴스를 반환한다.  
        이 경우 영속성 컨텍스트 안에  
        **detached 원본  ← 같은 PK →  managed 복제본**  
        다음과 같은 두 객체가 동시에 존재하게되고 A different object with the same identifier value was already associated…
       예외를 던져버림

    - 해결방법
        - 단순하게 저장로직을 persiste()로 만들면됨
        ![persist.png](imgs%2Fpersist.png)

          