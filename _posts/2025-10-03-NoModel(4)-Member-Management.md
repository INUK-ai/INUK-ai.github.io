---
title: "NoModel 회원·보안 아키텍처"
date: 2025-10-03 15:30:00 +0900
categories: [PROGRAMMERS, PROJECT]
tags: [NoModel, 회원관리, 보안, Redis]
description: "NoModel 서비스의 회원 도메인, 보안 정책, 이벤트 기반 감사 흐름을 코드 중심으로 정리한 문서"
---

## 01. Member 관리 개요

- `member` 모듈은 `application`·`domain`·`infrastructure` 3계층으로 구성되어 가입, 로그인, 사용자 정보 조회 여정을 담당합니다.
- 영속 계층은 JPA(`member_tb`, `login_history_tb`)와 Redis(`refresh:*`, `firstLogin:*`)를 함께 사용해 토큰과 최초 로그인 캐시를 분리 관리합니다.
- Spring Security 필터 체인과 감사 설정은 `_core/config`에서 통합되어 로그인 성공 시 `createdBy`·`modifiedBy`에 `memberId` 문자열이 자동 기록됩니다.
- 스케줄러는 90일이 지난 로그인 이력을 삭제하고, 도메인 서비스는 IP 차단 여부를 Redis에서 곧바로 확인합니다.
- `JWT` 기반 토큰을 선택해 API 게이트웨이 없이도 무상태 인증을 유지하며, Refresh Token으로 장기 세션을 안전하게 연장합니다.
- 토큰은 `ResponseCookie`에 담아 `httpOnly`·`secure`·`sameSite` 옵션을 강제하므로 XSS·CSRF 공격면을 최소화하고 브라우저 자동 전송을 이용합니다.

## 02. 도메인 모델 & 검증

- `Member` 엔티티는 값 객체 `Email`, `Password`와 역할/상태 enum으로 구성되며 팩토리 메서드가 기본 권한과 상태를 고정합니다.
  ```java
    @Builder
    private Member(String username, Email email, Password password, Role role, Status status) {
        this.username = username;
        this.email = email;
        this.password = password;
        this.role = role;
        this.status = status;
    }

    public static Member createMember(String username, Email email, Password password) {
        return Member.builder()
                .username(username)
                .email(email)
                .password(password)
                .role(Role.USER)
                .status(Status.ACTIVE)
                .build();
    }
  ```
- 빌더는 프라이빗 생성자에 연결되어 있고, 팩토리는 `Role.USER`·`Status.ACTIVE`를 강제해 신규 가입자가 항상 일반 사용자/활성 상태로 시작하게 합니다. `MemberAuthService`는 DTO를 값 객체로 변환한 뒤 이 팩토리를 호출해 생성 로직을 단순화합니다.
- 추가적으로 `validatePassword`와 `isActive` 헬퍼로 인증 로직에 필요한 검증을 도메인 내부에서 수행합니다.
  ```java
  public boolean validatePassword(String rawPassword, PasswordEncoder passwordEncoder) {
      return this.password.matches(rawPassword, passwordEncoder);
  }

  public boolean isActive() {
      return this.status == Status.ACTIVE;
  }
  ```
- `Email`, `Password`는 생성 시 자체 검증을 수행해 잘못된 값이 엔티티에 주입되지 않도록 합니다. 정규식·길이 검증을 통과한 값만 팩토리(`Email.of`, `Password.encode`)에서 반환되며, 암호는 `PasswordEncoder`로 즉시 해싱됩니다.
  ```java
  @Embeddable
  public class Email {
      public static Email of(String value) {
          return new Email(value);
      }

      private static void validateEmail(String value) {
          if (value == null || !value.matches("^[\\w.-]+@[\\w.-]+\\.\\w{2,}$")) {
              throw new ApplicationException(ErrorCode.INVALID_REQUEST);
          }
      }
  }

  @Embeddable
  public class Password {
      public static Password encode(String value, PasswordEncoder encoder) {
          validatePassword(value);
          return new Password(encoder.encode(value));
      }

      private static void validatePassword(String password) {
          if (password == null || password.length() < 4) {
              throw new ApplicationException(ErrorCode.INVALID_REQUEST);
          }
      }
  }
  ```

- `LoginHistory`는 성공/실패 여부와 해시된 IP를 저장하며, 실패 이력에도 `member_id`를 선택적으로 연결해 공격 패턴을 추적합니다.
- `RefreshToken`은 TTL 3일짜리 `RedisHash`로 저장되어 JWT 재발급과 권한을 묶어 유지합니다.
- `MemberDomainService`는 이메일 중복 검사와 향후 활성/탈퇴 규칙 확장 지점을 확보하고, DTO 레이어는 Bean Validation으로 1차 입력 검증을 수행합니다.

## 03. 보안 & 감사

- `CustomUserDetailsService`는 이메일로 회원을 조회해 `CustomUserDetails`를 생성하고, 계정 상태를 확인합니다.
  ```java
  public UserDetails loadUserByUsername(String email) {
      Member member = memberJpaRepository.findByEmail(Email.of(email))
              .orElseThrow(() -> new ApplicationException(ErrorCode.MEMBER_NOT_FOUND));
      if (!member.getStatus().equals(Status.ACTIVE)) throw new ApplicationException(ErrorCode.MEMBER_NOT_ACTIVE);
      return new CustomUserDetails(member.getId(), member.getEmail().getValue(),
                                   member.getPassword().getValue(),
                                   List.of(new SimpleGrantedAuthority(member.getRole().getKey())));
  }
  ```
- `LoginSecurityDomainService`는 실패 횟수를 Redis에 누적하고 지수형 차단 시간을 계산하며, 성공 시에는 즉시 초기화합니다.
- `AuditConfig`와 `SecurityConfig`가 결합되어 감사 컬럼과 JWT 기반 무상태 세션 정책을 동시에 적용합니다.

### 03-1. 로그인 이벤트 & 관찰 가능성

- `LoginEventListener`는 인증 성공/실패 이벤트를 비동기로 처리해 감사 로그, 실패 누적, 이상 로그인 감지를 동시에 수행합니다.
  ```java
  @Async
  @EventListener
  public void handleAuthenticationSuccess(AuthenticationSuccessEvent event) {
      CustomUserDetails user = (CustomUserDetails) event.getAuthentication().getPrincipal();
      Member member = memberRepository.findById(user.getMemberId()).orElseThrow(...);
      String ip = extractIpFromAuthentication(event.getAuthentication());
      String hashedIp = loginSecurityDomainService.hashIpAddress(ip);
      loginSecurityDomainService.clearLoginFailures(ip);
      loginSecurityDomainService.detectAnomalousLogin(member.getId(), hashedIp);
      loginHistoryRepository.save(LoginHistory.createSuccessHistory(member, hashedIp));
  }
  ```
- 실패 이벤트도 동일한 리스너에서 처리하여 실패 횟수 증가와 로그인 이력 저장을 자동화합니다.
- 감사 로그는 `_core/logging/StructuredLogger`와 `ControllerAspect`가 구조화하여 추적성을 확보합니다.

### 03-2. IP 차단 정책 (Redis)

- `LoginSecurityDomainService.recordLoginFailure`는 Redis에 실패 횟수를 누적하고, 차단 이력에 따라 지수형 차단 시간을 계산합니다.
  ```java
  public void recordLoginFailure(String ipAddress) {
      String hashedIp = hashIpAddress(ipAddress);
      String failureKey = FAILURE_KEY_PREFIX + hashedIp;
      long failures = Optional.ofNullable(redisTemplate.opsForValue().increment(failureKey)).orElse(1L);
      if (failures == 1) redisTemplate.expire(failureKey, Duration.ofMinutes(CHECK_MINUTES));
      if (failures >= MAX_FAILED_ATTEMPTS) {
          int blocks = ...;
          long blockMinutes = calculateBlockDuration(blocks);
          redisTemplate.opsForValue().set(historyKey, String.valueOf(blocks), Duration.ofHours(24));
          redisTemplate.expire(failureKey, Duration.ofMinutes(blockMinutes));
      }
  }
  ```
- `clearLoginFailures`는 성공 시 카운트와 차단 이력을 즉시 제거합니다.
- IP 해시화, 프록시 헤더 기반 IP 추출, 현재 IP가 차단되었는지 검사하는 메서드로 운영 중 보안 정책을 강화합니다.

## 04. 애플리케이션 서비스 & 컨트롤러

- `MemberAuthService`는 회원가입 시 값 객체 생성 → 중복 검사 → 암호화 → 저장 → 첫 로그인 캐시 기록을 한 플로우로 묶습니다.
- 로그인 플로우는 IP 차단 검증 → `AuthenticationManager` 인증 → JWT 발급 → 리프레시 토큰 Redis 저장까지 연결됩니다.
  ```java
  public AuthTokenDTO login(LoginRequestDto request) {
      loginSecurityDomainService.validateCurrentIpNotBlocked();
      Member member = memberJpaRepository.findByEmail(Email.of(request.email()))
              .orElseThrow(() -> new ApplicationException(ErrorCode.MEMBER_NOT_FOUND));
      return generateAuthTokens(member, request.email(), request.password());
  }
  ```
- `MemberAuthController`는 `ResponseCookie`로 httpOnly, secure, sameSite 옵션을 적용해 토큰을 반환합니다.
- 토큰 재발급/로그아웃도 동일 서비스에서 쿠키 추출 → 검증 → Redis 확인 → JWT 재발급/삭제 순으로 처리합니다.

## 05. 스케줄링 & 개인정보 보호

- `LoginHistoryCleanupScheduler`가 90일 이전 로그인 기록을 주기적으로 삭제해 개인정보 보존 정책과 보안 감사 요구를 동시에 충족합니다.
- 정리 작업은 `StructuredLogger`를 통해 삭제된 건수를 남겨 데이터 거버넌스에 필요한 추적성을 확보합니다.

## 06. 한계 및 향후 개선

- 이메일 중복 검증이 `MemberDomainService` 선행 조회에 의존하므로, 트래픽이 몰릴 때 동시 가입 시나리오에서 경합이 발생할 여지가 있습니다. 데이터베이스 유니크 제약과 애플리케이션 락 전략을 조합해 중복 허용 가능성을 차단해야 합니다.
- 동시 가입 경쟁으로 DB 유니크 제약(`DataIntegrityViolationException`)이 발생할 때 `EMAIL_ALREADY_EXISTS`로 매핑하는 전역 예외 처리기를 마련해 사용자 메시지를 일관되게 유지해야 합니다.
- Redis가 로그인 보안 로직의 단일 장애 지점으로 남아 있어 장애 시 IP 차단·리프레시 토큰 검증이 무력화됩니다. 멀티 AZ 배포와 복제본 승격 자동화, 장애 시 폴백 정책을 마련해 가용성을 높여야 합니다.

## 관련 레포지토리

- [NoModel 백엔드](https://github.com/prgrms-aibe-devcourse/AIBE2_FinalProject_NoModel_BE)
- [NoModel 프론트엔드](https://github.com/prgrms-aibe-devcourse/AIBE2_FinalProject_NoModel_FE)
