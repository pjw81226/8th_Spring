# 실습기록
- 6주차 워크북의 build.gradle
    ```groovy
    plugins {
        id 'java'
        id 'org.springframework.boot' version '3.0.0'
        id 'io.spring.dependency-management' version '1.0.15.RELEASE'
        id "com.ewerk.gradle.plugins.querydsl" version "1.0.10"
    }
    
    group = 'com.example'
    version = '1.0.0'
    sourceCompatibility = '17'
    
    repositories {
    mavenCentral()
    }
    
    dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.hibernate.orm:hibernate-core:6.0.2.Final'  // Hibernate 6.0.2 이상
    implementation 'mysql:mysql-connector-java:8.0.33'  // MySQL 드라이버 추가
    
        // queryDSL
        implementation 'com.querydsl:querydsl-jpa:5.0.0:jakarta'
        annotationProcessor "com.querydsl:querydsl-apt:5.0.0:jakarta"
        annotationProcessor "jakarta.annotation:jakarta.annotation-api"
        annotationProcessor "jakarta.persistence:jakarta.persistence-api"
    
        implementation 'com.fasterxml.jackson.core:jackson-databind:2.13.3'
        implementation 'com.fasterxml.jackson.core:jackson-annotations:2.13.3'
        implementation 'com.fasterxml.jackson.core:jackson-core:2.13.3'
    
    
        compileOnly 'org.projectlombok:lombok:1.18.22'
        annotationProcessor 'org.projectlombok:lombok:1.18.22'
    
        implementation 'org.hibernate:hibernate-core:5.6.9.Final'
    
        implementation 'org.springdoc:springdoc-openapi-ui:1.6.9'
        implementation 'org.springdoc:springdoc-openapi-data-rest:1.6.9'
    
        testImplementation 'org.springframework.boot:spring-boot-starter-test'
    }
    
    sourceSets {
    main {
    java {
    srcDirs = ['src/main/java', 'src/main/resources']
    }
    }
    }
    
    tasks.named('test') {
    useJUnitPlatform()
    }
    
    // Querydsl 설정부
    def generated = 'src/main/generated'
    
    querydsl {
    jpa = true
    querydslSourcesDir = generated
    }
    sourceSets {
    main.java.srcDir generated
    }
    
    compileQuerydsl{
    options.annotationProcessorPath = configurations.querydsl
    }
    
    configurations {
    compileOnly {
    extendsFrom annotationProcessor
    }
    querydsl.extendsFrom compileClasspath
    }
        
    ```

해당 build.gradle의 문제점
1. 하이버네이트 의존성 두개가 공존
    ```groovy
    implementation 'org.hibernate.orm:hibernate-core:6.0.2.Final' 
    implementation 'org.hibernate:hibernate-core:5.6.9.Final'
    ```
    spring boot 3.0.0 이상은 자체 BOM으로 Hibernate 6.1.x 를 관리해준다.  
    hibernate 5.x 계열은 javax.persistence 기반이라 spring boot 3.0.0 이상과 완벽히 호환되지 않는다.


2. jackson 세개의 버전 2.13.3 으로 고정
    ```groovy
    implementation 'com.fasterxml.jackson.core:jackson-databind:2.13.3'
    implementation 'com.fasterxml.jackson.core:jackson-annotations:2.13.3'
    implementation 'com.fasterxml.jackson.core:jackson-core:2.13.3'
    ```
    spring boot는 2.14.x 를 자동으로 끌고온다. 해당 의존성으로 인해서 서로 다른 버전이 충돌해 런타임 에러 발생 가능성이 농후 


3. com.ewerk.gradle.plugins.querydsl 플러그인은 Gradle6 시대의 플러그인  
    최근에는 플러그인 없이 annotationProcessor + javaCompile 옵션만으로 해결하는 방식이 더 안정적 


4. compileQuerydsl { … }처럼 직접 태스크를 선언하면 Gradle 8+에서 deprecation 경고가 뜸  
    tasks.named("compileQuerydsl") { … } 형태로 람다를 써서 안전하게 접근해야 함

<br>
<br>

- 변경한 build.gradle
    ```groovy
    plugins {
        id 'java'
        id 'org.springframework.boot' version '3.4.5'
        id 'io.spring.dependency-management' version '1.1.7'
    }
    
    group = 'umc'
    version = '0.0.1-SNAPSHOT'
    
    java {
        toolchain {
            languageVersion = JavaLanguageVersion.of(17)
        }
    }
    
    configurations {
        compileOnly {
            extendsFrom annotationProcessor
        }
    }
    
    repositories {
        mavenCentral()
    }
    
    dependencies {
        implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
        implementation 'org.springframework.boot:spring-boot-starter-web'
        compileOnly 'org.projectlombok:lombok'
        runtimeOnly 'com.mysql:mysql-connector-j'
        annotationProcessor 'org.projectlombok:lombok'
        testImplementation 'org.springframework.boot:spring-boot-starter-test'
        testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
    
        implementation 'com.querydsl:querydsl-jpa:5.1.0:jakarta'
        annotationProcessor 'com.querydsl:querydsl-apt:5.1.0:jakarta'
        annotationProcessor 'jakarta.persistence:jakarta.persistence-api:3.1.0'
    }
    
    tasks.named('test') {
        useJUnitPlatform()
    }
    
    def querydslDir = "$buildDir/generated/querydsl"
    
    sourceSets {
        main {
            java {
                srcDir querydslDir
            }
        }
    }
    
    tasks.withType(JavaCompile).configureEach {
        options.generatedSourceOutputDirectory = file(querydslDir)
    }
    ```
  - Q파일은 빌드 폴더에 들어가도록 수정함

<br>
<br>

- 아키텍쳐
  - 워크북에 나와있는 것처럼 JpaRepository를 Application Service Layer까지 끌고 올라가는 방식은 
    유지보수, 아키텍쳐 적 측면에서 좋지않은 형태, 이후에 JPA가 아닌 R2DBC, GRPC 등으로 전환시 
    상위 레이어의 코드를 전부 수정해야하는 상황이 생김
    (워크북 상에는 JpaRepository를 상위 스코프인 서비스 레이어에서 직접 의존)
  - 따라서, 상위 레이어에서 사용할 메소드를 StoreRepository에서 선언, 해당 메소드들의 정의를 StoreRepositoryImpl 에서 진행
    JpaStoreRepository는 StoreRepositoryImpl 에서 의존하여 필요한 메소드를 사용하는 방식으로 구현했다.

<br>
<br>
- 실습 코드

- store repository
    ![storerepository.png](imgs%2Fstorerepository.png)

- store repository impl
    ![storerepositoryimpl.png](imgs%2Fstorerepositoryimpl.png)

- jpa store repository
    ![jpastorerepository.png](imgs%2Fjpastorerepository.png)

- store query service
    ![storequeryservice.png](imgs%2Fstorequeryservice.png)

- store query service impl
    ![storeserviceimpl.png](imgs%2Fstoreserviceimpl.png)

# 미션기록

1. 내가 진행중이던 미션, 진행 완료된 미션
    ```java 
    private final JPAQueryFactory jpaQueryFactory;
    private final QMission qMission = QMission.mission;
    private final QUserMission qUserMission = QUserMission.userMission;
    private final QStore qStore = QStore.store;
   
    @Override
    public List<Mission> findMissionByUserAndState(Long userId, String missionState, Integer pagingOffset) {

        return jpaQueryFactory
                .selectFrom(qMission)
                .leftJoin(qUserMission)
                .on(qUserMission.mission.eq(qMission))
                .leftJoin(qStore)
                .on(qMission.store.eq(qStore))
                .where(
                        qUserMission.user.id.eq(userId),
                        qStore.state.eq(missionState)
                )
                .orderBy(qUserMission.createdAt.desc())
                .limit(10)
                .offset((long) (pagingOffset - 1) * 10)
                .fetch();
    }
    ```
   

2. 리뷰 작성  
    QueryDSL으로 단건의 리뷰를 Insert할 필요 없음 단순히 Jpa 메소드 이용
    ```java
    Review review = Review.builder()
            .user(user)
            .store(store)
            .title(title)
            .content(content)
            .score(score)
            .build();

    jpaReviewRepository.save(review)
    ```


3. 홈화면
    - 현재 달성 미션 개수 / 10
        ```java
        private final JPAQueryFactory jpaQueryFactory;
        private final QMission qMission = QMission.mission;
        private final QUserMission qUserMission = QUserMission.userMission;
        private final QStore qStore = QStore.store;
       
        @Override
        public Long countCompletedMissionRemainder(Long userId, String location) {
            NumberExpression<Long> remainderExpr = qUserMission.id.count().mod(10L);
    
            return jpaQueryFactory
                    .select(remainderExpr)
                    .from(qUserMission)
                    .join(qUserMission.mission, qMission)
                    .join(qMission.store, qStore)
                    .where(qUserMission.user.id.eq(userId),
                            qStore.location.eq(location),
                            qUserMission.state.eq(MissionState.COMPLETE)
                    )
                    .fetchOne();
        }
        ```

      - 진행하지 않은 미션
          ```java
          private final JPAQueryFactory jpaQueryFactory;
          private final QMission qMission = QMission.mission;
          private final QUserMission qUserMission = QUserMission.userMission;
          private final QStore qStore = QStore.store;
       
         @Override
         public List<MissionCardDto> findNotAcceptedMissions (Long userId, Integer pagingOffset){
         QStoreCategory qStoreCategory = QStoreCategory.storeCategory;

          return jpaQueryFactory
                  .select(Projections.constructor(MissionDto.class,
                          qStore.storeName,
                          qStoreCategory.category,
                          qMission.content,
                          qMission.point,
                          Expressions.numberTemplate(Long.class, "DATEDIFF({0}, current_timestamp)", qMission.dueDate)
                  ))
                  .from(qMission)
                  .leftJoin(qUserMission).on(qUserMission.mission.eq(qMission).and(qUserMission.user.id.eq(userId)))
                  .leftJoin(qStore).on(qMission.store.eq(qStore))
                  .leftJoin(qStoreCategory).on(qStore.storeCategories.contains(qStoreCategory))
                  .where(qUserMission.user.id.isNull())
                  .orderBy(qMission.createdAt.desc())
                  .limit(10)
                  .offset((long) (pagingOffset - 1) * 10)
                  .fetch();
          }
          ```
          ```java
          /*MissionDto*/
            public record MissionDto(String storeName,
                                   String storeCategoryName,
                                   String content,
                                   int    point,
                                   long   dDay) { }
          ```
        !. Projection.constructor는 생성하는 DTO의 생성자에 맞춰서 필드를 매핑하므로 DTO에 생성자가 필요.
    
    
4. 마이페이지
    ```java
    private final JPAQueryFactory jpaQueryFactory;
    private final QUser qUser = QUser.user;
   
    @Override
    public UserProfileDto getUserProfile(Long userId){
        return jpaQueryFactory
                .select(Projections.constructor(UserProfileDto.class,
                        qUser.email,
                        qUser.name,
                        qUser.point,
                        new CaseBuilder()
                                .when(qUser.phoneAuth.eq(false))
                                .then("미인증")
                                .otherwise(qUser.phoneNum)
                ))
                .from(qUser)
                .where(qUser.id.eq(userId))
                .fetchOne();
    }
    ```
   ```java
   /*UserProfileDto*/
    public record UserProfileDto(String email,
                             String name,
                             Integer point,
                             String phoneVerified) { }
    ```
    