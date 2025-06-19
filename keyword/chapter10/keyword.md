# CSRF
CSRF(Cross-Site Request Forgery, 사이트 간 요청 위조)는 사용자가 의도하지 않은 HTTP 요청을 공격자가 강제로 전송하도록 만드는 웹 공격 기법

<br>

**공격흐름**
1. 사용자가 사이트 A(은행)에 로그인 한다.
2. 사용자가 피싱사이트에 접근한다.
3. 피싱사이트에 접속시 사이트에 숨어있는 악의적인 스크립트가 실행된다.
  ```html
    <img src="https://siteA.com/transfer?amount=1000&to=attacker" />
  ```
4. 사용자는 A 사이트에 로그인 되어있기 때문에, session이 유지되고 있으므로 사이트는 해당 요청을 받아들인다.
5. 사이트 A는 요청을 처리하고, 사용자의 1000원이 공격자의 계좌에 이체된다.


<br>


**방어**  
1. JWT Token을 헤더를 통해 전송
    - http 헤더는 브라우저가 자동으로 붙이지 않고 스크립트(AJAX, fetch)가 명시적으로 설정해야만 전송.
    - 피싱 페이지는 다른 도메인에서 JS를 실행해도 이 헤더를 만들 수 없음 -> 자연스럽게 CSRF 차단
2. CAPTCHA
    - 사용자가 직접 입력해야 하는 CAPTCHA를 추가하여 자동화된 요청을 방지.
3. SameSite 쿠키 속성
    - 쿠키에 SameSite 속성을 설정하여, 다른 사이트에서의 요청에는 쿠키가 전송되지 않도록 한다.
    - `SameSite=Strict` 또는 `SameSite=Lax` 설정을 통해 CSRF 공격을 방지할 수 있다.
4. CSRF Token
    - 서버가 요청마다 난수(_csrf)를 세션에 저장 → 클라이언트가 hidden 필드·헤더로 그대로 다시 보내야 통과


<br>

**JWT Token Login 구현시, Spring Security에서 비활성화 하는 이유**
- JWT Token 로그인 시, Session을 사용하지 않고 Stateless 한 구조.
- 토큰은 클라이언트의 헤더에 저장해서 요청을 날림. 피싱사이트에서 자동으로 요청을 보내도 토큰 헤더 값을 추가할 수 없다.
- 따라 CSRF 공격이 발생할 수 있는 경로가 없으므로, CSRF 방어를 비활성화해도 된다.

**굳이 왜 끄는거지? 냅두면 안되나..**
- Session이 다시 생김 
  - `CsrfFilter`의 기본 저장소는 `HttpSessionCsrfTokenRepository` 필터가 요청을 처리하는 순간 HttpSession이 강제로 생성되어 버림. -> SessionCreationPolicy.STATELESS를 선언해도 “유령 세션”이 생겨 stateless가 아니게 됨.
- 토큰 없다고 403이 터진다
  - EST, 모바일 클라이언트, Postman에서 POST/PUT/DELETE 요청을 보내면 CSRF 토큰 헤더가 없어서 403 Forbidden 발생.
  - 매 호출마다 토큰을 실어 보내도록 모든 클라이언트 코드를 고칠 이유가 없음.
- 불필요한 필터 = 불필요한 비용
  - 필터 자체 비용은 작지만, TPS가 높은 API라면 매 요청마다 토큰 로직과 Origin 비교가 들어가는 것보다 깔끔히 빼는 편이 효율적.
- 예외 설정 지옥 방지
  - 필터를 켜 둔 채로 일부 URL만 토큰 검사를 건너뛰려면 `requestMatchers("/webhook/**").permitAll()·csrfTokenRequestHandler(...)` 등 예외 규칙을 계속 추가해야 한다.


# Spring Security
Spring Security는 스프링 기반 애플리케이션에 '인증','인가' 기능을 끼워 넣어 주는 보안 프레임워크다.

다른 라이브러리처럼 한 번 import-하면 끝인 도구는 아니다. 내부에 필터 체인, 전략 인터페이스, 다양한 헬퍼가 층층이 쌓여 있어서 “필요한 지점만 골라 끼워 맞추는” 방식으로 쓰게 된다.

<br>


1. 요청이 지나가는 큰 흐름

   - Filter Chain 시작
        - SecurityFilterChain이 앞단에 붙어 있다.
        - 요청이 오면 OncePerRequestFilter 계열 필터들이 차례로 호출된다.

   - Authentication Filter
        - 전통 세션 방식이면 UsernamePasswordAuthenticationFilter, 토큰 방식이면 커스텀 JwtAuthenticationFilter 등이 먼저 사용자를 식별한다.

   - AuthenticationManager → Provider
        - 필터가 추출한 자격 증명(Authentication)을 AuthenticationManager에 던진다.
        - ProviderManager가 등록된 AuthenticationProvider들(예: DaoAuthenticationProvider, JwtProvider)에 순서대로 위임해 검증한다.
   - SecurityContextHolder 저장
        - 검증 결과가 성공이면, 해당 Authentication을 SecurityContextHolder에 넣고 체인을 계속 돌린다.
        - 실패 시 ExceptionTranslationFilter가 잡아 401/403 처리.
   - Authorization (인가)
        - FilterSecurityInterceptor 단계에서 @PreAuthorize, hasRole, URL 매칭 규칙 등을 확인해 “접근 가능 여부”를 마지막으로 판정한다.
        - 이 흐름이 끝나야 @Controller 로직이 실행된다.

<br> <br>

2. 흔히 커스터마이징하는 지점
   - JWT 로그인
     - 세션 대신 JwtAuthenticationFilter를 UsernamePasswordAuthenticationFilter 앞에 붙인다. 인증 성공 시 SecurityContextHolder에 Authentication을 넣고, 실패 시 바로 401 응답.
    
   <br>

   - CSRF 비활성화
     - REST API만 제공할 때 http.csrf().disable()로 끈다. “그냥 꺼도 되나?” 싶겠지만, 스테이트리스 API라면 토큰 자체가 CSRF 대책 역할을 하므로 실무에서 자주 끄는 편.
   
    <br>
   
   - 메소드 보안
     - @EnableGlobalMethodSecurity(→ 6.3부터는 @EnableMethodSecurity)로 켜서 @PreAuthorize("hasRole('ADMIN')") 붙인다. URL 기준 접근 제어보다 세밀하게 걸 수 있어 도메인 계층에서 자주 쓴다.



# Authentication, Authorization

- 인증(Authentication)
  - 사용자가 누구인지 확인하는 과정. 
  - 예를 들어, 로그인 시 사용자 이름과 비밀번호를 확인하는 것. 
  - 실패시 401 Unauthorized 응답을 반환한다.


- 인가(Authorization)
  - 인증된 사용자가 특정 리소스나 기능에 접근할 수 있는 권한이 있는지 확인하는 과정. 
  - 예를 들어, 관리자가 아닌 사용자가 관리자 페이지에 접근하려고 할 때 거부하는 것. 
  - 실패시 403 Forbidden 응답을 반환한다.


- 팁
  - 토큰 만료
    - 재인증 필요(Refresh Token·Re-login). 일정시간이 지나면 토큰을 Expire. 토큰이 살아 있어야 인가도 의미가 있다. 
  - 캐시
    - 인증 정보는 캐시해서 성능 향상 가능. 예를 들어, Redis 같은 인메모리 데이터베이스에 저장해 빠르게 조회할 수 있다.
  - 오버-퍼미션
    - “모든 권한 포함” 토큰 발급 금지. 최소 권한을 발급하자.
  - 혼동
    - 401 vs 403을 뒤섞지 말기. → 신원 미확인이면 401, 권한 없음이면 403.

# Session, Token

- session
  - 핵심아이디어
    - 로그인 성공 -> 서버가 내부 저장소(Map, DB, Redis 등)에 세션 레코드 생성 -> 브라우저에 Set-Cookie: JSESSIONID=abc123 전송
  - 요청흐름
    - 쿠키( JSESSIONID=abc123 )가 자동 전송 -> 서버가 세션 테이블에서 abc123 조회 -> 사용자 정보 삽입 후 컨트롤러 실행
  - 장점
    - 쿠키만 관리하면 되니 브라우저 기본 플로우랑 궁합이 좋다.
    - 서버단에서 즉시 세션 만료·로그아웃 처리 가능.
    - 서버가 세션을 관리하므로, 클라이언트가 토큰을 직접 관리할 필요가 없다.
  - 단점
    - 서버가 상태를 보관하므로 scale-out 시 부하 분산이 까다롭다.
    - 쿠키 기반이라 CSRF 방어를 별도 적용해야 한다.
    - 세션 만료 시 클라이언트가 자동으로 재로그인하지 않는다. (토큰 기반은 자동 재로그인 가능)
  - 보통 쓰는 곳
    - 전통 웹(서버 사이드 렌더링), 관리 콘솔, 로그인 빈도 높은 사내 시스템
  
<br>

- token
  - 핵심아이디어
    - 로그인 성공 -> 서버가 자체 서명한 JWT발급 -> 클라이언트가 Authorization: Bearer <token>로 전송
  - 요청흐름
    - 서버는 토큰 서명, 만료시간만 확인 -> 사용자 정보는 토큰 안의 claim으로 바로 복원 -> 컨트롤러 실행
  - 장점
    - 서버 stateless -> 수평 확장, CDN 캐시 모두 편하다.
    - 모바일·SPA 같은 API 중심 아키텍처와 잘맞음
    - 토큰 하나에 역할, 권한 등 claim을 담아 불필요한 조회 줄임.
  - 단점
    - 토큰 자체를 revoke하기 어렵다 -> 블랙리스트나 짧은 TTL + Refresh Token 패턴 필요.
    - 로컬 스토리지에 두면 XSS, 쿠키에 두면 세션이랑 비슷하게 CSRF 위험.
  - 보통 쓰는 곳
    - SPA(React/Vue/Next.js), 모바일 앱, 마이크로서비스 내부 통신, B2B API Gateway

# Access Token, Refresh Token

- Access Token
  - 역할 : API 요청을 할 때 “나는 인증된 사용자”임을 증명
  - TTL : 짧다 (몇 분 ~ 수 시간)
  - 서버 상태 : 대개 stateless, 서명 검증만으로 충분
  - 전송 : 보통 Authorization: Bearer <token> 헤더 또는 httpOnly 쿠키
  - 폐기 : 만료 시 자동 폐기, 또는 블랙리스트 테이블
  - 보안위협 : 탈취 시 짧은 시간 동안만 악용 가능
  - 내용물(claims) : sub(사용자 ID), exp(만료 시각), roles, 기타 필요한 권한 정보.
  - 검증 방식 : 서버는 서명만 확인하고 DB 조회 없이 바로 권한을 판단할 수 있다.
  - 짧게 두는 이유 : 탈취 리스크를 줄이고, 사용자의 최신 권한 변경이 빠르게 반영되도록 하기 위함.


- Refresh Token
  - 역할 : Access Token이 만료됐을 때 새 Access Token을 발급받도록 허가
  - TTL : 길다 (며칠 ~ 수 주)
  - 서버 상태 : 보안상 서버측 저장소(화이트리스트)를 둬서 revoke 가능하도록 설계하는 편
  - 전송 : Access Token처럼 보내지 않고, 전용 POST /auth/refresh 같은 엔드포인트에서만 사용
  - 폐기 : 서버의 저장소에서 제거하거나 회전(rotation)
  - 보안위협 : 탈취 시 장기간 재발급 가능 -> 엄격하게 보호 필요
  - 발급, 보관
    1. 로그인 성공 시 Access Token과 함께 발급.
    2. 서버(또는 Redis 등)에 토큰 ID -> 사용자 매핑을 저장해 둔다.
    3. 클라이언트는 Refresh Token을 안전한 저장소(보통 httpOnly 쿠키)로 보관.
  - 재발급 흐름
    1. Access Token 만료 → 401
    2. 프론트엔드 → POST /auth/refresh { refreshToken: … }
    3. 서버: 토큰 검증 + 저장소 조회 → 새 Access Token(+옵션: 새 Refresh Token) 반환
  - 회전(rotation) : 매번 새 Refresh Token을 돌려주고, 이전 토큰 ID는 즉시 무효화. 탈취 시도 탐지에도 활용 가능.