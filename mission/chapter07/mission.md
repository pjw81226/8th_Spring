# week7 Mission

이번 주차 미션은 단순히 실습 코드를 작성하는 것.

# 생각해보기

### ResponseEntity? ApiResponse?
이번주차의 실습은 ApiResponse와 예외를 전역처리하는 ExceptionHandler를 구현하는 것.

실습을 진행하면서 생각해볼 점을 정리해봄.

1. ResponseEntity  
   HTTP 응답(Response)을 좀 더 세밀하게 제어할 수 있게 해주는 스프링의 클래스.  
   HTTP 상태 코드(status code), 헤더(headers), 바디(body)를 직접 설정해서 클라이언트에 반환가능

2. ApiResponse
   API 응답 바디에 들어가는 표준화된 데이터 구조(일명 DTO, Data Transfer Object).
   클라이언트에게 항상 일정한 형태의 JSON 응답을 보내기 위해 사용.


즉, ResponseEntity는 HTTP 응답을 구성하는 클래스이고, ApiResponse는 그 응답 바디에 들어가는 데이터 구조라고 할 수 있음.


### 응답의 일관성
실습 코드를 보자.   
예외가 발생한 경우, 당연하게도 HttpStatus를 올바른 값으로 설정해야하므로 ResponseEntity를 반환한다.

![advice.png](imgs%2Fadvice.png)

다만, Controller를 보면 ApiResponse를 그대로 반환하는 것을 볼 수 있다.
![prevcontroller.png](imgs%2Fprevcontroller.png)


물론 굳이 성공한 응답에 대해 HttpStatus를 따로 설정할 필요는 없을 수 있다.  

다만, 요청이 성공적으로 처리 되었을 때도, 201(Created), 202(Accepted)와 같이 다양한 상태코드를 반환할 수 있고 이를 통해 서버가 클라이언트에게 좀 더 정확하고 의미 있는 정보를 줄 수 있다.

추가적으로, 오청이 성공했을 때 반환하는 데이터 타입과 에러가 발생했을 때 반환하는 데이터 타입이 다르면 큰 기능적 문제는 없을 수 있지만, 다음과 같은 문제가 발생할 수 있다.
1. 응답 일관성 저해  
   클라이언트는 매번 응답 구조가 달라서 이를 처리하는 로직을 복잡하게 작성해야함
2. 프론트엔드와의 소통 문제  
   프론트 개발자가 API 응답을 예상하기 어렵고, 문서화가 까다로움
3. 유지보수 어려움  
   API 확장이나 수정 시, 다양한 응답 타입을 모두 고려해야 해서 개발과 테스트가 복잡해짐

### 수정한 부분
Controller에서 ApiResponse 위에 ResponseEntity를 감싸는 형식으로 반환.
![aftercontroller.png](imgs%2Faftercontroller.png)

이때 좀 더 일관성 있는 응답을 위해 ResponseEntity를 만드는 Util 클래스 작성  
(Util 클래스는 일반적으로 static 메소드로 작성하여 매번 객체를 생성하지 않고도 사용 가능하도록 함.)
![util.png](imgs%2Futil.png)

Util 클래스의 메소드를 이용해서 Controller 다시 작성
![lastcontroller.png](imgs%2Flastcontroller.png)

# 미션 인증

![temptest.png](imgs%2Ftemptest.png)
![flag2.png](imgs%2Fflag2.png)
![flag1.png](imgs%2Fflag1.png)


