---
title: "NoModel 프로젝트 설정 가이드"
date: 2025-10-03 14:00:00 +0900
categories: [PROGRAMMERS, PROJECT]
tags: [기획, NoModel, 아키텍처]
description: "NoModel 서비스의 프로젝트 구조, 공통 모듈 설계, DevOps 자원을 정리한 설정 가이드"
---

## 01. 프로젝트 전반 구조

- **루트 구성**
  - Gradle(Spring Boot 3, Java 21) 단일 모듈 기준입니다.
  - 루트에는 `build.gradle`, `settings.gradle`, 각종 `docker-compose` 파일,
    REST Docs 산출물(`docs/`), 성능 스크립트(`k6/`), 모니터링 설정(`monitoring/`)이 위치합니다.
- **애플리케이션 레이어링**
  - `src/main/java/com/example/nomodel` 하위에 `member`, `subscription`, `model`, `report`,
    `point`, `file`, `generationjob`, `compose`, `coupon`, `review`, `search`, `statistics`,
    `removebg`, `generate` 모듈을 배치합니다.
  - 각 모듈은 `application`·`domain`·`infrastructure` 계층을 고정합니다.
- **리소스 자원**
  - `src/main/resources`에는 환경별 `application-*.yml`, Flyway 마이그레이션(`db/migration`),
    더미 데이터 SQL(`sql/dummy-data`), 정적 문서, Firebase 키 경로가 묶여 있습니다.
- **테스트 구조**
  - `src/test/java`는 메인 패키지 트리를 그대로 따라가며 `_core`가 공통 테스트 베이스와 fixtures를 제공합니다.
  - 통합 테스트 시드는 `src/test/resources/sql`에서 관리합니다.

## 02. 도메인 모듈 패턴

- **member**: 인증/회원 API와 이벤트 리스너(`application`), `Member`·`LoginHistory`·값 객체(`domain`), 보안 컴포넌트와 스케줄러(`infrastructure`)를 분리합니다.
- **subscription**: 플랜, 회원 구독, 할인 정책 엔티티를 유지하며 외부 결제 연동 스켈레톤을 `infrastructure`에 준비합니다.
- **model**: `model/command`와 `model/query`를 나눠 CQRS 스타일을 구현하고, `AIModel`, `ModelMetadata`, `ModelStatistics`를 명령/조회 컨텍스트에 배치합니다.
- **기타 모듈**: `generationjob`, `file`, `point`, `report`, `review`, `coupon` 등도 동일한 3계층 패턴을 따르며 Firebase, Redis, Elasticsearch, PortOne 등 외부 API 클라이언트나 배치 스케줄러는 `infrastructure`에 위치합니다.

## 03. _core 패키지 역할

### AOP

- `AopConfig`에서 AspectJ 자동 프록시를 활성화하고, `ControllerAspect`, `ServicePerformanceAspect`, `SlowQueryDetectorAspect`가 비즈니스 크리티컬 요청과 서비스 실행 시간, 슬로우 쿼리를 감시합니다.
  ```java
  @Configuration
  @EnableAspectJAutoProxy
  public class AopConfig { }
  ```
  ```java
  @Around("pointcut()")
  public Object logBusinessRequests(ProceedingJoinPoint joinPoint) throws Throwable {
      MDC.put("layer", "Controller");
      StopWatch watch = new StopWatch();
      watch.start();
      try {
          Object response = joinPoint.proceed();
          watch.stop();
          structuredLogger.logApiRequest(...);
          return response;
      } catch (Exception ex) {
          watch.stop();
          structuredLogger.logApiRequest(...);
          throw ex;
      } finally {
          MDC.clear();
      }
  }
  ```

### 공통 베이스

- `BaseTimeEntity`, `BaseEntity`가 모든 엔티티에서 생성·수정 시각과 작성자·수정자를 일관되게 관리합니다.

### 구성 설정

- `AuditConfig` + `AuditorAwareImpl`이 Spring Data Auditing을 통해 `CustomUserDetails.memberId`를 감사 컬럼에 주입합니다.
- `SecurityConfig`, `AuthBeansConfig`, `AsyncConfig`, `AopConfig`, `SwaggerConfig`, `LoggingConfig`, `MetricsConfig`, `RedisCacheConfig`, `WebClientConfig`, `ElasticsearchConfig`, `FireBaseConfig` 등 인프라 설정이 `_core/config`에 모여 있으며 테스트/통합 테스트 프로필에서는 Firebase·Elasticsearch 초기화를 끕니다.
  ```java
  @Bean
  public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
      http.csrf(AbstractHttpConfigurer::disable)
          .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
          .authorizeHttpRequests(auth -> auth
              .requestMatchers(WHITE_LIST).permitAll()
              .requestMatchers(ADMIN_LIST).hasRole("ADMIN")
              .anyRequest().authenticated())
          .addFilterBefore(new JWTTokenFilter(jwtTokenProvider),
                           UsernamePasswordAuthenticationFilter.class)
          .oauth2Login(oauth -> oauth
              .userInfoEndpoint(u -> u.userService(customOAuth2UserService))
              .successHandler(oAuth2SuccessHandler));
      return http.build();
  }
  ```

### 컨트롤러 및 데모

- `ApiDemoController`, `TestApiController`, `AopDemoController`가 공통 응답(`ApiUtils.success`)과 AOP 동작을 예시로 제공합니다.

### 로깅·보안·유틸

- `StructuredLogger`, `LoggingFilter`가 MDC 기반 구조화 로그와 `traceId` 주입, 토큰 마스킹을 처리합니다.
- `CustomUserDetails`, `CustomUserDetailsService`, `JWTTokenProvider`, `CookieProperties`가 JWT 인증과 사용자 식별을 담당합니다.
- `ApiUtils`, `SecurityUtils`가 응답 표준화와 현재 사용자 조회를 돕습니다.
  ```java
  public static Long getCurrentMemberId() {
      CustomUserDetails user = getCurrentUserDetails();
      if (user.getMemberId() == null) throw new ApplicationException(ErrorCode.AUTHENTICATION_FAILED);
      return user.getMemberId();
  }
  public static boolean hasAuthority(String authority) {
      return getCurrentAuthentication().getAuthorities().stream()
              .anyMatch(granted -> granted.getAuthority().equals(authority));
  }
  ```

#### ApiUtils 구조

```java
package com.example.nomodel._core.utils;

import org.springframework.http.HttpStatus;

public class ApiUtils {

    public static <T> ApiResult<T> success(T response) {
        return new ApiResult<>(true, response, null);
    }

    public static ApiResult<?> error(String message, HttpStatus status) {
        return new ApiResult<>(false, null, new ApiError(message, status.value()));
    }

    public static <T> ApiResult<T> error(T data) {
        return new ApiResult<>(false, null, data);
    }

    public record ApiResult<T>(
            boolean success,
            T response,
            T error
    ) {}

    public record ApiError(
            String message,
            int status
    ) {}
}
```

### 예외 처리

- `ApplicationException`, `ErrorCode`, `GlobalControllerAdvice`가 도메인별 오류를 표준 응답으로 변환합니다.
  ```java
  public class ApplicationException extends RuntimeException {
      private final ErrorCode errorCode;
      private final LocalDateTime timestamp = LocalDateTime.now();
      public ApplicationException(ErrorCode errorCode) {
          super(errorCode.getMessage());
          this.errorCode = errorCode;
      }
  }
  ```
  ```java
  @ExceptionHandler(ApplicationException.class)
  public ResponseEntity<?> handle(ApplicationException e) {
      ErrorResponse body = new ErrorResponse(
          e.getErrorCode().getStatus().value(),
          e.getErrorCode().getErrorCode(),
          e.getErrorCode().getMessage(),
          e.getTimestamp()
      );
      return ResponseEntity.status(e.getErrorCode().getStatus())
                           .body(ApiUtils.error(body));
  }
  ```
- `ErrorCode` 열거형에는 도메인별 코드와 `HttpStatus`가 정의돼 있으며, 아래처럼 일부 대표 항목을 사용합니다.
  ```java
  @Getter
  public enum ErrorCode {
      INVALID_REQUEST("IRE001", HttpStatus.BAD_REQUEST, "Invalid request"),
      MEMBER_NOT_FOUND("MNF001", HttpStatus.NOT_FOUND, "Member not found"),
      MODEL_NOT_FOUND("MONF001", HttpStatus.NOT_FOUND, "Model not found"),
      REVIEW_NOT_ALLOWED("RV003", HttpStatus.FORBIDDEN, "Not allowed to modify or delete this review"),
      PAYMENT_VERIFICATION_FAILED("PVF001", HttpStatus.BAD_REQUEST, "Payment verification failed.");
      // ... (도메인별 항목 추가 정의)
  }
  ```

## 04. 리소스 · DevOps 세팅

- **로컬 컴포즈**: `compose.yml`로 MySQL·Redis를 포함한 로컬 개발 스택을 기동합니다.
- **추가 스택**: `docker-compose-*.yml`은 Elasticsearch, k6 부하 테스트, 모니터링, 애플리케이션 단독 실행 등 상황별 구성을 제공합니다.
- **모니터링 문서**: `monitoring/`, `ELK_PROMETHEUS_ROLE_SEPARATION.md`에서 ELK, Prometheus/Grafana 역할 분리를 정의합니다.
- **팀 가이드**: `TESTING_GUIDE.md`, `TEAM_SETUP_GUIDE.md`, `MONITORING_SETUP.md`가 온보딩과 품질 지침을 지원합니다.

## 05. 프론트엔드 프로젝트 세팅

- **기본 스택**
  - Vite 기반 React 18 + TypeScript 프로젝트입니다.
  - Tailwind CSS v4, Radix UI 컴포넌트, lucide 아이콘, react-hook-form, axios, recharts 등을 사용합니다.
  - 스크립트는 `npm run dev`(Vite 개발 서버), `npm run build`(배포 번들) 두 가지로 단순화했습니다.
- **구조 개요**
  - `src/main.tsx`, `src/App.tsx`에서 루트 렌더링 및 App stage 전환을 담당합니다.
  - `src/components`는 `ui`(Radix 래퍼), `common`, `figma`, `workflow` 하위로 나뉘어 화면 단위를 구성합니다.
  - `src/services`는 `AxiosInstance`, `ApiService`, `auth`, `modelApi`, `pointApi`, `reportApi` 등 REST 호출 모듈을 정리해 백엔드 API와 연결합니다.
  - `src/types`와 `src/utils`에서 API 응답, 도메인 모델, 쿠키/아바타 유틸을 정의합니다.
  - `src/styles/globals.css`, `src/index.css`는 Tailwind 프리셋과 글로벌 스타일을 적용합니다.
  - `src/config/env.ts`는 `VITE_OAUTH_CALLBACK` 등 런타임 환경 변수 접근을 통일합니다.
- **상태 관리 접근**
  - `App.tsx`는 `AppStage`를 기반으로 랜딩→로그인→생성 워크플로우→관리자 등 화면을 스위칭합니다.
  - `services/auth.ts`가 로그인 상태/토큰 저장, `services/AxiosInstance.ts`가 공통 인터셉터를 제공합니다.
- **정적 자료 및 문서화**
  - `src/guidelines/Guidelines.md`, `src/Attributions.md`, `COMPONENT_GUIDE.md`가 UI 사용법과 저작권 정보를 문서화합니다.
  - `build/`는 배포 산출물, `vercel.json`은 Vercel 배포 구성을 담고 있습니다.
- **디자인 소스**
  - Linear 제품 페이지를 벤치마킹해 공통 UI 패턴을 선행 분석하고, Figma Make 파일에서 프로토타입을 제작한 뒤 해당 컴포넌트 구조를 코드로 이식했습니다.
  - `src/components/ui`와 `COMPONENT_GUIDE.md`에서 Linear/Figma 기반 디자인 토큰과 패턴을 재사용할 수 있도록 정리했습니다.

## 06. 정리

- `_core`가 인프라·보안·감사·로깅·AOP를 캡슐화해 도메인 모듈은 비즈니스 로직에 집중할 수 있습니다.
- 모든 엔티티는 `BaseTimeEntity`/`BaseEntity`를 상속해 감사 컬럼을 공유하고, `AuditorAware`가 `SecurityContext.memberId`를 자동 반영합니다.
- 도메인 디렉터리뿐 아니라 테스트·문서·더미 데이터 폴더도 `application`·`domain`·`infrastructure` 규칙을 따라 협업자 탐색이 쉽습니다.

## 관련 레포지토리

- [NoModel 백엔드](https://github.com/prgrms-aibe-devcourse/AIBE2_FinalProject_NoModel_BE)
- [NoModel 프론트엔드](https://github.com/prgrms-aibe-devcourse/AIBE2_FinalProject_NoModel_FE)
