# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 프로젝트 개요

`ItemRepository` 인터페이스의 구현체를 메모리 → JDBC → JPA 등으로 **갈아끼우며 학습하는 구조**의 Spring MVC 상품 관리 예제다. 서비스/웹 계층은 고정한 채 저장소 구현만 교체하는 것이 이 코드베이스의 핵심 목적이다.

Spring Boot 4.1.1 / Java 25 (Gradle toolchain) / Thymeleaf / H2 2.4.240.

## 명령어

```bash
./gradlew bootRun                                        # 애플리케이션 실행 (http://localhost:8080)
./gradlew build                                          # 빌드
./gradlew test                                           # 전체 테스트
./gradlew test --tests 'ItemRepositoryTest'              # 클래스 단위 테스트
./gradlew test --tests 'ItemRepositoryTest.findItems'    # 메서드 단위 테스트

./h2/bin/h2.sh -tcp -tcpAllowOthers                      # H2 TCP 서버 기동 (앱 실행 전 필수)
```

린터/포매터는 설정돼 있지 않다.

## 아키텍처

### 컴포넌트 스캔이 web 패키지로 제한돼 있다 (가장 중요)

```java
@Import(MemoryConfig.class)
@SpringBootApplication(scanBasePackages = "hello.itemservice.web")
```

`ItemServiceApplication`의 이 선언 때문에 **`MemoryItemRepository`의 `@Repository`, `ItemServiceV1`의 `@Service` 애노테이션은 동작하지 않는다.** 서비스·저장소 빈은 오직 `@Import`로 지정된 Config 클래스의 `@Bean` 메서드를 통해서만 등록된다.

따라서 저장소 구현체를 추가할 때의 절차는:

1. `repository/` 하위에 `ItemRepository` 구현체 작성
2. `config/` 에 해당 구현체를 `@Bean`으로 등록하는 `@Configuration` 클래스 작성 (`MemoryConfig` 참고)
3. `ItemServiceApplication`의 `@Import(...)` 를 새 Config로 **교체**

애노테이션만 붙이고 스캔되길 기대하면 안 된다.

### 계층 구조

`ItemController` → `ItemService`(인터페이스) → `ItemRepository`(인터페이스) → 구현체

계층 간 파라미터 객체는 `repository` 패키지에 있다. 수정은 `ItemUpdateDto`, 검색 조건은 `ItemSearchCond`(`itemName` 부분일치 + `maxPrice` 이하, 둘 다 null/빈값이면 전체 조회)를 쓴다.

### 프로필

- `main/resources/application.properties` → `local`
- `test/resources/application.properties` → `test`

`TestDataInit`(itemA, itemB 시드 데이터 주입)은 `@Profile("local")` 이라 **테스트에서는 실행되지 않는다.** 시드 데이터를 전제로 테스트를 작성하면 안 된다.

### 테스트

`ItemRepositoryTest`는 `@SpringBootTest`라 현재 `@Import` 중인 Config의 구현체를 그대로 검증한다. 즉 구현체를 교체하면 **같은 테스트가 새 구현체를 검증하게 된다** — 이것이 의도된 설계다.

`@AfterEach`의 `instanceof MemoryItemRepository` 분기는 메모리 구현 전용 정리 로직이다. 실제 DB 구현체를 붙일 때는 이 부분 대신 트랜잭션 롤백 방식으로 전환해야 한다. `MemoryItemRepository`의 `store`/`sequence`가 `static` 이라 테스트 간 상태가 공유되는 점도 주의.

## H2 데이터베이스

H2 배포본이 `h2/`에, DB 파일이 `data/test.mv.db`에 있다. 둘 다 `.gitignore` 대상이다.

접속 URL (`application.properties`에 설정됨):
```
jdbc:h2:tcp://localhost//Users/jyr/IdeaProjects/spring/spring_db/itemservice-db/data/test
```

- **절대 경로가 하드코딩돼 있다.** 다른 환경에서는 수정이 필요하다.
- TCP 모드를 쓰는 이유는 애플리케이션과 웹 콘솔이 DB에 동시 접속하기 위해서다. 파일 모드(`jdbc:h2:~/test`)는 단독 점유라 동시 접속이 불가능하다.
- H2 서버 버전과 `com.h2database:h2` 드라이버 버전은 **일치해야 한다** (현재 둘 다 2.4.240). 버전을 올릴 때는 `h2/` 배포본도 함께 교체할 것.

`sql/schema.sql`은 **자동 실행되지 않는다** (classpath 밖인 `sql/` 에 있음). 스키마 변경 시 직접 적용해야 한다:

```bash
java -cp h2/bin/h2-2.4.240.jar org.h2.tools.RunScript \
  -url "jdbc:h2:<위 URL의 경로>" -user sa -script sql/schema.sql
```

이 스크립트는 `drop table if exists item CASCADE`로 시작하므로 **기존 데이터가 모두 삭제된다.**

## 커밋

전역 지침에 따라 Conventional Commits v1.0.0을 따른다.
