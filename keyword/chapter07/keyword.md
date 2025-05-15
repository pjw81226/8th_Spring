

# RestControllerAdvice
스프링에서 제공하는 어노테이션임

주로 REST API 개발에서 사용

컨트롤러에서 발생하는 예외를 한 곳에서 전역 처리 가능

@ControllerAdvice + @ResponseBody 기능 합친 것.

예외 처리 메서드는 @ExceptionHandler 붙여서 특정 예외 종류별로 만들 수 있음

예를 들어, DB에서 데이터 못 찾으면 404 에러, 서버 오류는 500 에러 등 상태코드랑 메시지를 일괄적으로 관리 가능

에러 처리 로직을 여러 컨트롤러마다 중복 작성 할 필요 없음

응답은 JSON이나 XML 형태로 바로 내려줘서 클라이언트가 쉽게 파싱 가능함

필요하면 예외 처리할 때 로그 남기거나, 커스텀 에러 객체 만들어서 상세 정보 전달 가능
```java 
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(NullPointerException.class)
    public String handleNull(NullPointerException e) {
        return "널포인터 예외 발생";
    }
}
```


# Lombok
자바에서 가장 많이 쓰이는 라이브러리 중 하나임

자바는 기본적으로 getter/setter, 생성자, toString, equals, hashCode 같은 메서드 작성 -> 보일러 플레이트 코드

Lombok의 다양한 어노테이션 붙이면 컴파일 시점에 자동으로 이런 메서드들을 만듬

그래서 코드가 훨씬 간결해지고 가독성이 상승

대표 어노테이션

@Getter / @Setter : 필드별 getter/setter 자동 생성

@NoArgsConstructor : 기본 생성자 자동 생성

@AllArgsConstructor : 모든 필드 매개변수 받는 생성자 생성

@Data : 위 기본 어노테이션들 (getter, setter, toString, equals, hashCode) 한번에 생성

@Builder : 빌더 패턴 지원 (객체 생성 시 가독성 증가)

프로젝트에 Lombok 라이브러리 추가하고 IDE 설정(플러그인)만 해주면 바로 사용 가능


```java
@Data
public class User {
    private String name;
    private int age;
}

```