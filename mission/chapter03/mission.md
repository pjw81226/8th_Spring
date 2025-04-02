# Chapter03

로그인한 유저의 Id는 spring security context holder에서 가져온다고 가정.
프론트에서 보내는 요청이라고 가정

## 홈 화면
| API EndPoint            | Request Body | Request Header                      | Query String                 | Path Variable |
|-------------------------|--------------|-------------------------------------|------------------------------|---------------|
| GET /api/missions/count | x            | Authorization : accessToken(string) | state=complete&location=~~   | x             |
| GET /api/missions       | x            | Authorization : accessToken(string) | state=not-started&location=~~ | x             |


## 마이 페이지 리뷰 작성
| API EndPoint     | Request Body                                                                   | Request Header                      | Query String | Path Variable  |
|------------------|--------------------------------------------------------------------------------|-------------------------------------|--------------|----------------|
| POST /api/review | storeId : ,<br> score : ,<br> content: ,<br>  image -> multipartfile/form-data | Authorization : accessToken(string) | x            | x              |

## 미션 목록 조회
| API EndPoint      | Request Body | Request Header                      | Query String | Path Variable |
|-------------------|--------------|-------------------------------------|--------------|---------------|
| GET /api/missions | x            | Authorization : accessToken(string) | state=~~     | x             |

## 미션 성공 누르기
| API EndPoint         | Request Body    | Request Header                      | Query String | Path Variable |
|----------------------|-----------------|-------------------------------------|--------------|---------------|
| PATCH /api/missions/ | state: complete | Authorization : accessToken(string) | x            | {missionId}   |

## 회원 가입 하기
| API EndPoint                | Request Body                                       | Request Header                      | Query String | Path Variable |
|-----------------------------|----------------------------------------------------|-------------------------------------|--------------|---------------|
| POST /api/users/signup      | name: ,<br>gender: ,<br> birth: ,<br> address:     | x                                   | x            | x             |
| POST /api/users/prefer-food | categoryId1: ,<br>categoryId2: ,<br> categoryId3:  | Authorization : accessToken(string) | x            | x             |
