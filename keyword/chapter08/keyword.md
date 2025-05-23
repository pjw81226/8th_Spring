# Java 예외 종류
체크 예외(Checked Exception)  
- 컴파일 단계에서 반드시 try-catch나 throws로 처리해야 한다.

- 주로 외부 자원 접근 같은 상황에서 발생한다.

  - IOException : 파일·네트워크 I/O 실패

  - SQLException : DB 연결·쿼리 오류

  - ParseException : 날짜·숫자 파싱 실패

  - ClassNotFoundException : 동적 로딩 실패

- 장점

  - 오류 처리 의무가 있어 안전성이 높다.

- 단점

  - 코드가 장황해지고, 지나친 예외 전파가 발생할 수 있다.

<br>
<br>

언체크 예외(Unchecked Exception, RuntimeException 계열)  
- 컴파일러가 강제하지 않는다. 런타임에 발생하여 프로그램을 중단시킬 수 있다.

  - NullPointerException

  - IllegalArgumentException
    
  - IndexOutOfBoundsException

  - ArithmeticException

- 장점

  - 비즈니스 로직이 간결해진다.

- 단점

  - 예외 처리를 놓치기 쉬워 런타임 오류로 직결될 수 있다.

# @Valid (Bean Validation)
- 위치

  - Spring MVC·Spring Boot에서 컨트롤러 매개변수, 서비스 메서드 인자, DTO 필드 등에 붙인다.

- 동작

  - 객체에 선언된 제약 조건 애너테이션(@NotNull, @Size, @Email 등)을 검사한다.

  - 유효성 실패 시 MethodArgumentNotValidException 또는 ConstraintViolationException을 던진다.

  - BindingResult를 추가하면 예외를 던지지 않고 오류 정보를 담아준다.

- 장점

  - 검증 규칙을 DTO 클래스 내부에 기술하여 응집도 향상.

  - 컨트롤러·서비스 로직이 검증 코드로 어지럽혀지지 않는다.

  - @Valid + @Validated(groups = …)로 그룹별 검증이 가능하다.

- 주의점

  - 중첩 객체는 @Valid를 필드에도 붙여야 검증이 전파된다.

  - 컬렉션·배열 요소 검증은 @Valid + @Size 등을 함께 사용해야 한다.

  - 검증 실패 메시지는 ValidationMessages.properties 또는 messages.properties로 국제화할 수 있다.