---
title: "NoModel 데이터 설계"
date: 2025-10-03 12:00:00 +0900
categories: [PROGRAMMERS, PROJECT]
tags: [기획, NoModel, 데이터설계, ERD]
description: "NoModel 서비스의 도메인 및 데이터베이스 설계를 도메인별 책임과 개선 포인트 관점에서 회고한 글"
---

## 01. 설계 개요

NoModel의 데이터 구조는 도메인별 책임을 명확히 나누고 감사 메타데이터를 공통 상속으로 묶어 무결성을 확보하는 데 초점을 맞추었습니다. Spring Data Auditing을 전 도메인에 도입해 운영·보안 감사 요건을 충족하고 동일한 정책을 강제했습니다. 모든 핵심 엔티티는 `BaseTimeEntity`와 `BaseEntity` 계층을 공유하며, 값 객체와 enum을 적극 활용하여 상태 전이를 제한하고 있습니다.

## 02. 회원 · 보안 도메인

- `member_tb`는 이메일과 비밀번호를 값 객체로 감싸 유효성과 암호화를 도메인 내부에서 처리하며, `ACTIVE/SUSPENDED` 전이를 enum으로 제한하여 의도치 않은 상태 변화를 예방합니다. 초기 설계보다 강한 검증을 엔티티에 내장해 도메인 오류를 선제적으로 차단하고자 했습니다. 다만 username에는 유니크 제약이 없어 검색 인덱스 추가가 향후 과제로 남았습니다.
- `login_history_tb`는 hashed IP와 로그인 결과, 시각을 기록하여 보안 감사를 지원합니다. 실패 이력도 저장하지만 `memberId`를 null로 허용하고 있어 실패 유형 보조 컬럼과 not-null 전략을 비교 검토하고 있습니다.

```sql
CREATE TABLE member_tb (
  member_id BIGINT PRIMARY KEY AUTO_INCREMENT,
  email VARCHAR(255) NOT NULL UNIQUE,
  password VARCHAR(255) NOT NULL,
  status VARCHAR(20) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  created_by VARCHAR(50),
  modified_by VARCHAR(50),
  KEY idx_member_created_at (created_at)
);

CREATE TABLE login_history_tb (
  login_history_id BIGINT PRIMARY KEY AUTO_INCREMENT,
  member_id BIGINT,
  hashed_ip VARCHAR(64) NOT NULL,
  login_status VARCHAR(20) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  KEY idx_login_history_member_id (member_id),
  KEY idx_login_history_created_at (created_at)
);
```


## 03. AI 모델 · 콘텐츠 도메인

- `ai_model_tb`는 프롬프트, 시드값, 해상도 등 생성 파라미터를 `@Embeddable` 값 객체로 묶어 재생성과 파라미터 히스토리를 쉽게 추적하도록 설계했습니다. 공개 여부는 `isPublic`과 `ModelType`으로 구분하여 마켓플레이스 필터링에 활용하고 있습니다.
- `model_statistics_tb`는 조회수와 사용량을 분리해 관리함으로써 AI 모델을 로드하지 않고도 주요 지표를 확인할 수 있도록 구성했습니다. `last_updated`를 감사 필드로 대체해 중복 컬럼을 제거했습니다. 주기적 초기화 기능은 갖추었으나, 장기 집계를 위한 시간 단위 파티셔닝은 향후 보완할 계획입니다.
- `ad_result_tb`, `file_tb`, `model_review`는 생성 산출물, 첨부파일, 리뷰를 각각 관리합니다. 파일 컨텍스트를 분리해 생성 파이프라인과 산출물 저장을 독립적으로 운영하려는 판단이었습니다. 특히 `file_tb`는 `relationType/relationId` 다형성 구조로 여러 도메인에서 재사용할 수 있도록 설계했지만, 조인 인덱스가 없어 대량 데이터 환경에서는 성능 튜닝이 필요합니다. 리뷰는 `createdAt`만 보유하고 있어 수정 이력 필드가 추가되면 더 완전한 감사를 수행할 수 있습니다.

```sql
CREATE TABLE file_tb (
  file_id BIGINT PRIMARY KEY AUTO_INCREMENT,
  relation_type ENUM('MODEL','REVIEW','PROFILE','AD','REMOVE_BG') NOT NULL,
  relation_id BIGINT NOT NULL,
  is_primary TINYINT(1) NOT NULL DEFAULT 0,
  KEY idx_file_relation (relation_type, relation_id)
);

CREATE TABLE model_statistics_tb (
  statistics_id BIGINT PRIMARY KEY AUTO_INCREMENT,
  model_id BIGINT NOT NULL,
  usage_count BIGINT NOT NULL DEFAULT 0,
  view_count BIGINT NOT NULL DEFAULT 0,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```


## 04. 생성 파이프라인

- `generation_job`은 UUID 기반 식별자와 상태 전이를 집중 관리하여 비동기 잡 큐와 연동하기 쉽게 구성했습니다. Stable Diffusion 등 외부 생성 API 연동을 비동기 잡으로 분리해 운영 부담을 줄였습니다. 현재 자체 `createdAt`과 `updatedAt` 필드를 사용하고 있어 JPA 감사와 중복되지 않도록 `BaseTimeEntity` 상속으로 통일하는 방안을 검토하고 있습니다.

```sql
CREATE TABLE generation_job (
  id CHAR(36) PRIMARY KEY,
  status VARCHAR(20) NOT NULL,
  parameters TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  KEY ix_job_status_createdAt (status, created_at)
);
```


## 05. 구독 · 과금 설계

- `subscription_plan → subscription → member_subscription → discount_policy`로 이어지는 계층 구조를 통해 플랜 정의, 판매 SKU, 회원 연결, 할인 정책을 분리했습니다. 그러나 `Subscription` 생성자가 `dailyLimit`과 `selfMadeModelNum`을 전달받지 않아 런타임 오류 위험이 존재하므로 생성자 시그니처 보완을 최우선 과제로 삼고 있습니다. 또한 플랜과 SKU의 역할을 포트폴리오 설명에서 명확히 구분해 혼선을 줄이고자 합니다.
- `coupon`은 사용/미사용 상태와 사용 시각만 기록하므로 `couponCode` 유니크 제약과 타임존 일관성 전략(UTC 저장)을 추가할 계획입니다.

```sql
CREATE TABLE subscription (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  plan_type VARCHAR(20) NOT NULL,
  daily_limit INT,
  self_made_model_num INT
);
```


## 06. 포인트 시스템

- `member_point_balance`는 낙관적 락과 3-tier 잔액(합계, 가용, 보류, 예약)을 한 행으로 묶어 동시성 충돌 없이 적립·사용·보류 흐름을 분리 관리합니다. `point_transaction`으로 흐름을 기록하지만 회원·거래유형 인덱스가 부족해 대규모 조회 성능을 개선해야 합니다.
- `review_reward_transaction`은 모델당 1회 보상 지급을 보장하는 유니크 키가 장점입니다. 보상 보류 상태를 추가하면 운영 유연성을 높일 수 있다고 판단합니다.
- `point_policy`는 문자열 정책명만 사용해 오탈자 위험이 있으므로 enum 또는 코드 테이블 도입을 검토하고 있습니다.

```sql
CREATE TABLE member_point_balance (
  member_id BIGINT PRIMARY KEY,
  total_points DECIMAL(18,2) NOT NULL,
  available_points DECIMAL(18,2) NOT NULL,
  pending_points DECIMAL(18,2) NOT NULL,
  reserved_points DECIMAL(18,2) NOT NULL,
  version BIGINT NOT NULL
);

CREATE TABLE point_transaction (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  member_id BIGINT NOT NULL,
  transaction_type VARCHAR(30) NOT NULL,
  direction VARCHAR(10) NOT NULL,
  created_at DATETIME NOT NULL,
  KEY idx_point_tx_created_at (created_at)
  -- 추천: KEY idx_point_tx_member_type (member_id, transaction_type)
);
```

## 07. 신고 · 운영 도메인

- `report_tb`는 `createdBy` 감사 정보를 활용해 신고자를 추적하며, 상태 전이 검증을 엔티티에 포함시켜 무결성을 보장합니다. 모델, 리뷰 등 여러 타깃을 하나의 워크플로로 관리하기 위해 신고 도메인을 통합했습니다. 다만 `targetType/targetId` 조합 인덱스와 중복 신고 제약이 없어 향후 개선이 필요합니다.

```sql
CREATE TABLE report_tb (
  report_id BIGINT PRIMARY KEY AUTO_INCREMENT,
  target_type VARCHAR(30) NOT NULL,
  target_id BIGINT NOT NULL,
  report_status VARCHAR(20) NOT NULL,
  admin_note VARCHAR(500),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  created_by VARCHAR(50),
  modified_by VARCHAR(50),
  KEY idx_report_target (target_type, target_id)
);
```


## 08. 공통 감사 설계

- `BaseTimeEntity`는 `@MappedSuperclass`로 `createdAt`과 `updatedAt`을 자동 관리하고, `BaseEntity`는 `createdBy`와 `modifiedBy`를 추가하여 누가 레코드를 생성·수정했는지 추적합니다. AuditorAware를 통해 인증 주체를 자동 주입할 수 있어 감사 일관성을 확보했습니다.

```java
@MappedSuperclass
public abstract class BaseTimeEntity {
    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;
}

@MappedSuperclass
public abstract class BaseEntity extends BaseTimeEntity {
    @CreatedBy
    @Column(updatable = false)
    private String createdBy;

    @LastModifiedBy
    private String modifiedBy;
}
```

- 일부 엔티티가 자체 타임스탬프를 유지하는 예외가 있어 사유를 주석으로 명시하거나 상속 구조로 통일할 예정입니다. 또한 `createdBy`와 `modifiedBy`가 문자열이므로 ID 기반 추적 전략을 도입하면 사용자 변경에도 더욱 안전합니다.

## 10. 향후 개선 방향

1. 회원·보안 도메인
   - username 유니크 제약과 로그인 실패 유형을 세분화해 계정 탈취 시도를 조기 탐지합니다.
   - 실패 로그에 입력 ID·실패 유형을 추가해 공격 패턴 분석과 계정 보호에 대비합니다.
2. 구독·과금 도메인
   - 구독 생성자 시그니처를 보완하고 종료·일시정지 분기를 정교화해 사용 제한 정책을 안정화합니다.
   - dailyLimit/selfMadeModelNum 초기화 누락을 점검해 정량 제한이 정상 동작하도록 합니다.
   - MemberSubscription의 결제 연관 ID 구조를 재검토해 향후 결제 이력 테이블 연동 기반을 마련합니다.
   - DiscountPolicy/Coupon enum 중복을 정리해 정책 변경 시 유지보수 비용을 줄입니다.
3. 모델 도메인
   - ModelMetadata 임베디드 구조와 공개/가격 제어 메서드를 유지·보강해 운영 정책을 일관되게 적용합니다.
4. 파일·리뷰 도메인
   - 파일 relation 인덱스, contentType/isPrimary, 리뷰 Rating 임베디드를 다듬어 품질 관리 기능을 강화합니다.
5. 포인트 도메인
   - member_point_balance와 point_transaction에 복합 인덱스를 추가하고 감사 로그를 확장해 대량 조회·감사 대응을 높입니다.
6. 신고·운영 도메인
   - 신고 보류 상태 정의와 중복 신고 제약을 강화해 운영자 업무 부하와 악용 사례를 줄입니다.
7. 공통 감사·정책
   - 타임존, 법적 책임, 저작권 정책을 정비하고 값 객체 검증을 유지해 글로벌 규제와 도메인 일관성 요구에 대응합니다.

## 관련 레포지토리

- [NoModel 백엔드](https://github.com/prgrms-aibe-devcourse/AIBE2_FinalProject_NoModel_BE)
- [NoModel 프론트엔드](https://github.com/prgrms-aibe-devcourse/AIBE2_FinalProject_NoModel_FE)
