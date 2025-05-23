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
## 복합키 저장  
   복합키를 primary key로 하는 엔티티를 저장할때 JpaRepository.save()를 사용하면 에러 발생.

   - 왜?
     - jpaRepository.save()는 EntityManger.merge()로 구현되어있음
     - 복합 PK 엔티티는 테이블에 저장하기 이전에 미리 Entity에 PK를 주입시켜놓은 상태  
       즉, save()는 이미 아이디가 있는 것을 보고 테이블에 이미 존재하는 엔티티라 생각, merge()를 호출함
     - merge()는 새로운 ‘관리되는 복제본’을 만들어서 상태를 복사한 뒤 그 인스턴스를 반환한다.  
        이 경우 영속성 컨텍스트 안에  
        **detached 원본  ← 같은 PK →  managed 복제본**  
        다음과 같은 두 객체가 동시에 존재하게되고 A different object with the same identifier value was already associated…
       예외를 던져버림 
   

   - 해결방법
     - 단순하게 저장로직을 persist()로 만들면됨
        ![persist.png](imgs%2Fpersist.png)


## DB에 존재하는 카테고리인지 체크하는 부분
   먼저 실습에 나와있는 코드에서 문제가 될만한 부분을 가져와봤다.
    ![problem.png](imgs%2Fproblem.png)

   단순하게 유저의 카테고리를 받아서 존재하는 카테고리인지 확인한 후에 해당 유저를 저장하는 로직이다.  
   문제되는 부분은 다음과 같다.  
   - 굳이 annotation을 만들고 DTO에서 카테고리가 존재하는지 체크까지 했는데 여기에서 다시 한 번 쿼리들을 호출하면서 체크하고있다.
     만약 사용자가 선호 카테고리를 10개 추가해서 보내면 그대로 카테고리가 있는지 확인하는 것에 20개의 쿼리가 발생한다. 
   - 이유는 당연하게도 stream으로 쪼갠다음 각각에 대해서 findById를 하고있으니 ...

해당 부분을 다음과 같이 수정했다.
    ![solve.png](imgs%2Fsolve.png)
- 이렇게 수정할 경우 CategoryIdList를 받아서 In절을 통해 벌크로 카테고리들을 가져온다.
- 단점은, 만약 실제 존재하지 않는 카테고리 ID들이 들어와도 에러를 발생시키진 않는다. 만약 CategoryIdList에 12124 이라는 값이 있어도 그냥 무시하고 나머지만 가져오는방식
- 즉, 무조건 존재하는 카테고리 Id가 들어온다는 가정하에 해당 방식을 사용할 수 있다. 앞에서 @ExistCategories 를 이용해 체크했으니 가능하다.

<br>
<br> 

### 추가적인 문제
이러한 방식을 통해 해결을 했어도 어쨌든 존재하는 카테고리인지 확인하기 위해 @ExistCategories 에서 카테고리 개수만큼 쿼리가 발생하는 것은 마찬가지다.
위의 해결방법은 단순히 2N의 쿼리를 N개의 쿼리로 줄인 것.

성능개선을 위해 다음과 같이 생각해봤다. 

없는 카테고리 ID가 들어왔을 때 해당 ID가 무엇인지는 그렇게 궁금한 정보가 아니다. 
때문에, List<CategoryID> 를 받고, 위에서와 마찬가지로 List에 존재하는 카테고리들을 In절을 통해 벌크로 가져온다.
그 다음 두 리스트의 개수를 세어 같으면 true, 다르면 false를 return 하는 방식으로 구현해보자.
이 경우 카테고리 개수만큼이 아닌 단순하게 In 절 쿼리 한번만으로 올바른 카테고리들이 들어왔는지 아닌지를 판단할 수 있다.

In 절이 당연히 디비 응답속도는 안좋다. 쿼리 실행시 or 로 변경되어 직접 동등연산을 진행하기 때문이다. 하지만, 카테고리의 개수가 백개 천개가 될 가능성은 없다고 판단했다.

어떤 카테고리 ID가 들어와서 문제가 되었는지에 대한 추적은 불가능하지만 그에 상응하는 성능 개선이라는 이점이 존재하므로 프로젝트 성향에 맞춰서 구현하면 될 것 같다.