---
title: "NoModel 모델 조회"
date: 2025-10-10 14:00:00 +0900
categories: [PROGRAMMERS, PROJECT]
tags: [NoModel, 검색, Elasticsearch, Redis]
description: "Elasticsearch 색인부터 API, 캐시 전략까지 NoModel 모델 검색 흐름을 코드 중심으로 정리한 문서"
---

## 검색 파이프라인 개요

- **데이터 수집**: 모델이 생성·수정·삭제되면 `ModelCreatedEvent`·`ModelUpdatedEvent`·`ModelDeletedEvent`가 발행되고, `AIModelIndexingListener`가 트랜잭션 커밋 이후 비동기로 응답합니다.
- **색인 구성**: `ElasticsearchIndexService`가 모델·통계·리뷰 데이터를 묶어 `AIModelDocument`로 변환하고, Nori 분석기와 completion 설정이 적용된 `ai-models` 인덱스에 저장합니다.
- **검색 API**: `AIModelSearchService`가 키워드·무료 여부·소유자 조건을 조합해 Spring Data Elasticsearch 쿼리를 실행하고, 자동완성은 completion suggester로 처리합니다.
- **캐시·후처리**: `CachedModelSearchService`가 키워드 없는 인기 조회만 Redis에 캐싱하고, `FileService`가 이미지 URL을 일괄 조합해 검색 응답 DTO를 완성합니다.
- **무효화**: `SmartCacheEvictionService`와 `LazyInvalidationService`가 모델·리뷰 이벤트에 따라 즉시 또는 지연 방식으로 캐시를 비웁니다.

## 도메인 모델 구조

도메인 엔티티는 검색 파이프라인의 데이터 흐름을 결정하므로 주요 클래스별 구조를 토글로 정리했습니다.

<details markdown="1">
<summary>`AIModel` 엔터티</summary>

- 모델 기본 정보와 소유자, 가격, 공개 여부, 메타데이터를 한 번에 관리하며 관리자·사용자 모델을 각각 생성할 정적 팩터리를 제공합니다.

```java
@Getter
@Setter
@Entity
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Table(name = "ai_model_tb")
public class AIModel extends BaseTimeEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "model_id")
    private Long id;

    @Column(name = "model_name", nullable = false, length = 100)
    private String modelName;

    @Embedded
    private ModelMetadata modelMetadata;

    @Enumerated(EnumType.STRING)
    @Column(name = "own_type", nullable = false)
    private OwnType ownType;

    @Column(name = "owner_id", nullable = false)
    private Long ownerId;

    @Column(name = "price", precision = 10, scale = 2)
    private BigDecimal price;

    @Column(name = "is_public", nullable = false)
    private boolean isPublic;

    @Builder
    private AIModel(String modelName, ModelMetadata modelMetadata, OwnType ownType,
                    Long ownerId, BigDecimal price, boolean isPublic) {
        this.modelName = modelName;
        this.modelMetadata = modelMetadata;
        this.ownType = ownType;
        this.ownerId = ownerId;
        this.price = price;
        this.isPublic = isPublic;
    }

    public static AIModel createUserModel(String modelName, ModelMetadata modelMetadata, Long ownerId) {
        return AIModel.builder()
                .modelName(modelName)
                .modelMetadata(modelMetadata)
                .ownType(OwnType.USER)
                .ownerId(ownerId)
                .price(BigDecimal.ZERO)
                .isPublic(false)
                .build();
    }

    public static AIModel createAdminModel(String modelName, ModelMetadata modelMetadata, BigDecimal price) {
        return AIModel.builder()
                .modelName(modelName)
                .modelMetadata(modelMetadata)
                .ownType(OwnType.ADMIN)
                .ownerId(0L)
                .price(price)
                .isPublic(true)
                .build();
    }

    public void updatePrice(BigDecimal newPrice) {
        this.price = newPrice;
    }

    public void updateVisibility(boolean isPublic) {
        this.isPublic = isPublic;
    }

    public void updateMetadata(ModelMetadata modelMetadata) {
        this.modelMetadata = modelMetadata;
    }

    public boolean isPubliclyAvailable() {
        return this.isPublic;
    }

    public boolean isOwnedBy(Long userId) {
        return this.ownerId.equals(userId);
    }

    public boolean isAdminModel() {
        return this.ownType.isAdminOwned();
    }

    public boolean isPaidModel() {
        return this.price != null && this.price.compareTo(BigDecimal.ZERO) > 0;
    }

    public boolean isAccessibleBy(Long userId) {
        return this.isPublic || this.isOwnedBy(userId);
    }

    public boolean isHighResolutionModel() {
        return this.modelMetadata != null && this.modelMetadata.isHighResolution();
    }
}
```

</details>

<details markdown="1">
<summary>`ModelMetadata` 값 타입</summary>

- 시드, 프롬프트, 해상도, 샘플러와 같은 생성형 이미지 파라미터를 캡슐화해 업데이트 시 새로운 인스턴스를 반환하도록 설계했습니다.

```java
@Embeddable
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@AllArgsConstructor(access = AccessLevel.PRIVATE)
@Builder
public class ModelMetadata {
    @Column(name = "seed")
    private Long seed;

    @Column(name = "prompt", length = 2000)
    private String prompt;

    @Column(name = "negative_prompt", length = 1000)
    private String negativePrompt;

    @Column(name = "width", nullable = false)
    private Integer width;

    @Column(name = "height", nullable = false)
    private Integer height;

    @Column(name = "steps", nullable = false)
    private Integer steps;

    @Enumerated(EnumType.STRING)
    @Column(name = "sampler_index", nullable = false)
    private SamplerType samplerIndex;

    @Column(name = "n_iter", nullable = false)
    private Integer nIter;

    @Column(name = "batch_size", nullable = false)
    private Integer batchSize;

    public static ModelMetadata of(Long seed, String prompt, String negativePrompt,
                                   Integer width, Integer height, Integer steps,
                                   SamplerType samplerIndex, Integer nIter, Integer batchSize) {
        return ModelMetadata.builder()
                .seed(seed)
                .prompt(prompt)
                .negativePrompt(negativePrompt)
                .width(width)
                .height(height)
                .steps(steps)
                .samplerIndex(samplerIndex)
                .nIter(nIter)
                .batchSize(batchSize)
                .build();
    }

    public ModelMetadata updatePrompt(String prompt, String negativePrompt) {
        return ModelMetadata.builder()
                .seed(this.seed)
                .prompt(prompt)
                .negativePrompt(negativePrompt)
                .width(this.width)
                .height(this.height)
                .steps(this.steps)
                .samplerIndex(this.samplerIndex)
                .nIter(this.nIter)
                .batchSize(this.batchSize)
                .build();
    }

    public ModelMetadata updateDimensions(Integer width, Integer height) {
        return ModelMetadata.builder()
                .seed(this.seed)
                .prompt(this.prompt)
                .negativePrompt(this.negativePrompt)
                .width(width)
                .height(height)
                .steps(this.steps)
                .samplerIndex(this.samplerIndex)
                .nIter(this.nIter)
                .batchSize(this.batchSize)
                .build();
    }

    public ModelMetadata updateSamplingSettings(Integer steps, SamplerType samplerIndex, Integer nIter, Integer batchSize) {
        return ModelMetadata.builder()
                .seed(this.seed)
                .prompt(this.prompt)
                .negativePrompt(this.negativePrompt)
                .width(this.width)
                .height(this.height)
                .steps(steps)
                .samplerIndex(samplerIndex)
                .nIter(nIter)
                .batchSize(batchSize)
                .build();
    }

    public boolean isHighResolution() {
        return this.width != null && this.height != null &&
               (this.width * this.height) >= (1920 * 1080);
    }

    public Integer getTotalImages() {
        return this.nIter * this.batchSize;
    }

    public boolean isRandomSeed() {
        return this.seed == null || this.seed == -1L;
    }

    public boolean isFixedSeed() {
        return !isRandomSeed();
    }

    public ModelMetadata updateSeed(Long seed) {
        return ModelMetadata.builder()
                .seed(seed)
                .prompt(this.prompt)
                .negativePrompt(this.negativePrompt)
                .width(this.width)
                .height(this.height)
                .steps(this.steps)
                .samplerIndex(this.samplerIndex)
                .nIter(this.nIter)
                .batchSize(this.batchSize)
                .build();
    }
}
```

</details>

<details markdown="1">
<summary>`ModelStatistics` 엔터티</summary>

- 모델과 1:N으로 연결돼 조회수·사용량을 누적하며, 필요 시 집계 값을 직접 조정하거나 초기화하도록 제공합니다.

```java
@Getter
@Entity
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Table(name = "model_statistics_tb")
public class ModelStatistics extends BaseTimeEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "statistics_id")
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "model_id", nullable = false)
    private AIModel model;

    @Column(name = "usage_count", nullable = false)
    private Long usageCount = 0L;

    @Column(name = "view_count", nullable = false)
    private Long viewCount = 0L;

    @Builder
    private ModelStatistics(AIModel model) {
        this.model = model;
    }

    public static ModelStatistics createInitialStatistics(AIModel model) {
        return ModelStatistics.builder()
                .model(model)
                .build();
    }

    public void incrementUsageCount() {
        this.usageCount++;
    }

    public void incrementViewCount() {
        this.viewCount++;
    }

    public void updateStatistics(long usageCount, long viewCount) {
        if (usageCount >= 0) {
            this.usageCount = usageCount;
        }
        if (viewCount >= 0) {
            this.viewCount = viewCount;
        }
    }

    public Long getTotalInteractions() {
        return this.usageCount + this.viewCount;
    }
}
```

</details>

<details markdown="1">
<summary>`AIModelDocument` Elasticsearch 문서</summary>

- 모델 기본 필드와 통계·자동완성 정보를 함께 담아 추가 DB 조회 없이 검색 응답을 구성합니다.

```java
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Document(indexName = "ai-models")
@Setting(settingPath = "/elasticsearch/ai-models-settings.json")
@Mapping(mappingPath = "/elasticsearch/ai-models-mappings.json")
public class AIModelDocument {
    @Id
    private String id;

    private Long modelId;
    private String modelName;
    private java.util.List<String> suggest;
    private String prompt;
    private String[] tags;
    private String ownType;
    private Long ownerId;
    private String ownerName;
    private BigDecimal price;
    private Boolean isPublic;
    private Long usageCount;
    private Long viewCount;
    private Double rating;
    private Long reviewCount;
    @Field(type = FieldType.Date, format = {}, pattern = "uuuu-MM-dd'T'HH:mm:ss.SSSSSS")
    private LocalDateTime createdAt;
    @Field(type = FieldType.Date, format = {}, pattern = "uuuu-MM-dd'T'HH:mm:ss.SSSSSS")
    private LocalDateTime updatedAt;

    @Builder
    private AIModelDocument(Long modelId, String modelName, java.util.List<String> suggest, String prompt,
                           String[] tags, String ownType, Long ownerId, String ownerName,
                           BigDecimal price, Boolean isPublic,
                           Long usageCount, Long viewCount, Double rating, Long reviewCount,
                           LocalDateTime createdAt, LocalDateTime updatedAt) {
        this.modelId = modelId;
        this.modelName = modelName;
        this.suggest = suggest != null ? suggest : buildSuggestions(modelName);
        this.prompt = prompt;
        this.tags = tags;
        this.ownType = ownType;
        this.ownerId = ownerId;
        this.ownerName = ownerName;
        this.price = price;
        this.isPublic = isPublic;
        this.usageCount = usageCount != null ? usageCount : 0L;
        this.viewCount = viewCount != null ? viewCount : 0L;
        this.rating = rating != null ? rating : 0.0;
        this.reviewCount = reviewCount != null ? reviewCount : 0L;
        this.createdAt = createdAt;
        this.updatedAt = updatedAt;
    }

    public static AIModelDocument from(AIModel aiModel, String ownerName,
                                      Long usageCount, Long viewCount,
                                      Double rating, Long reviewCount) {

        AIModelDocument aiModelDocument = AIModelDocument.builder()
                .modelId(aiModel.getId())
                .modelName(aiModel.getModelName())
                .suggest(buildSuggestions(aiModel.getModelName()))
                .prompt(extractPrompt(aiModel))
                .tags(extractTags(aiModel))
                .ownType(aiModel.getOwnType().name())
                .ownerId(aiModel.getOwnerId())
                .ownerName(ownerName)
                .price(aiModel.getPrice())
                .isPublic(aiModel.isPublic())
                .usageCount(usageCount)
                .viewCount(viewCount)
                .rating(rating)
                .reviewCount(reviewCount)
                .createdAt(aiModel.getCreatedAt())
                .updatedAt(aiModel.getUpdatedAt())
                .build();

        aiModelDocument.id = String.valueOf(aiModel.getId());
        return aiModelDocument;
    }

    private static String extractPrompt(AIModel aiModel) {
        if (aiModel.getModelMetadata() != null && aiModel.getModelMetadata().getPrompt() != null) {
            return aiModel.getModelMetadata().getPrompt();
        }
        return "";
    }

    private static String[] extractTags(AIModel aiModel) {
        if (aiModel.getModelMetadata() != null && aiModel.getModelMetadata().getSamplerIndex() != null) {
            return new String[]{"AI", "IMAGE_GENERATION", aiModel.getModelMetadata().getSamplerIndex().name()};
        }
        return new String[]{"AI", "IMAGE_GENERATION"};
    }

    private static java.util.List<String> buildSuggestions(String modelName) {
        if (modelName == null || modelName.trim().isEmpty()) {
            return new java.util.ArrayList<>();
        }

        String[] words = modelName.trim()
                .toLowerCase()
                .split("[\\s\\-\\_\\.]");

        java.util.List<String> validWords = new java.util.ArrayList<>();
        for (String word : words) {
            String trimmedWord = word.trim();
            if (trimmedWord.length() >= 2 && !trimmedWord.matches("^[0-9.]+$")) {
                validWords.add(trimmedWord);
            }
        }

        java.util.List<String> allSuggestions = new java.util.ArrayList<>();
        allSuggestions.add(modelName.trim());
        allSuggestions.addAll(validWords);

        return allSuggestions;
    }
}
```

</details>

## 01. 검색 인덱스 개요

- `model` 모듈은 명령(`command`)과 조회(`query`) 레이어를 분리해 색인과 검색 책임을 나눕니다.
- Elasticsearch 도큐먼트는 `AIModelDocument`로 정의되어 모델 기본 정보, 가격, 통계(사용량·조회수·평점)를 한 번에 담습니다.
- 색인 설정(`ai-models-settings.json`)은 Nori 형태소 분석기와 completion suggester를 적용해 한글 검색과 자동완성을 동시에 지원합니다.
- 매핑(`ai-models-mappings.json`)은 `modelName`·`prompt`에 커스텀 분석기를 붙이고, `tags`·`ownType` 등을 `keyword` 타입으로 선언해 정확한 필터링을 수행합니다.
- 전체 문서는 `usageCount`, `rating`, `reviewCount` 필드를 통해 랭킹 가중치 조정과 하이라이트 전략을 뒷받침합니다.

## 02. 인덱싱 파이프라인

### 데이터 수집

- `ModelEvent.java`는 모든 모델 이벤트의 공통 부모로 `modelId`, 이벤트 발생 시각(`timestamp`), 이벤트 타입명을 기본 필드로 제공합니다. 이를 통해 이후 단계에서 추가 조회 없이 이벤트 메타데이터를 활용할 수 있습니다.

```java
@Getter
@RequiredArgsConstructor
public abstract class ModelEvent {
    private final Long modelId;
    private final LocalDateTime timestamp = LocalDateTime.now();
    private final String eventType = this.getClass().getSimpleName();
}
```

- `ModelCreatedEvent`는 생성 시점의 공개 여부, 판매가, 소유 타입을 `AIModel` 엔터티에서 즉시 추출해 보존하므로 캐시 정책이나 색인 로직이 동일 이벤트 페이로드로 의사결정을 수행할 수 있습니다.

```java
@Getter
public class ModelCreatedEvent extends ModelEvent {
    private final boolean isPublic;
    private final int price;
    private final String ownType;

    public ModelCreatedEvent(AIModel model) {
        super(model.getId());
        this.isPublic = model.isPublic();
        this.price = model.getPrice().intValue();
        this.ownType = model.getOwnType().name();
    }
}
```

- `ModelDeletedEvent`는 삭제 대상 모델 ID만 캡처해 불필요한 영속성 로딩 없이 캐시 무효화와 색인 제거가 가능하도록 단순화되어 있습니다.

```java
public class ModelDeletedEvent extends ModelEvent {

    public ModelDeletedEvent(Long modelId) {
        super(modelId);
    }
}
```

- `ModelUpdateEvent`는 `updateType`과 이전/이후 값을 함께 담고, `priceChange`·`visibilityChange` 등 정적 팩터리 메서드를 제공해 서비스 레이어가 의도를 명확히 표현할 수 있습니다.

```java
@Getter
public class ModelUpdateEvent extends ModelEvent {
    private final String updateType;
    private final Object oldValue;
    private final Object newValue;

    public ModelUpdateEvent(Long modelId, String updateType, Object oldValue, Object newValue) {
        super(modelId);
        this.updateType = updateType;
        this.oldValue = oldValue;
        this.newValue = newValue;
    }

    public static ModelUpdateEvent priceChange(Long modelId, Object oldPrice, Object newPrice) {
        return new ModelUpdateEvent(modelId, "PRICE", oldPrice, newPrice);
    }
    public static ModelUpdateEvent visibilityChange(Long modelId, Boolean oldVisibility, Boolean newVisibility) {
        return new ModelUpdateEvent(modelId, "VISIBILITY", oldVisibility, newVisibility);
    }
    public static ModelUpdateEvent basicInfoChange(Long modelId) {
        return new ModelUpdateEvent(modelId, "BASIC_INFO", null, null);
    }
    public static ModelUpdateEvent filesChange(Long modelId) {
        return new ModelUpdateEvent(modelId, "FILES", null, null);
    }
}
```

- `ModelUpdateEventPublisher`는 `publishAfterCommit` 메서드로 트랜잭션 동기화가 활성화된 경우 `TransactionSynchronization`을 등록해 커밋 이후에만 이벤트를 발행하고, 그렇지 않은 경우 즉시 발행해 테스트나 비동기 시나리오에서도 일관성을 유지합니다.

```java
@Component
public class ModelUpdateEventPublisher {

    private final ApplicationEventPublisher eventPublisher;

    public ModelUpdateEventPublisher(ApplicationEventPublisher eventPublisher) {
        this.eventPublisher = eventPublisher;
    }

    public void publishAfterCommit(ModelUpdateEvent event) {
        if (TransactionSynchronizationManager.isSynchronizationActive()) {
            TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronization() {
                @Override
                public void afterCommit() {
                    eventPublisher.publishEvent(event);
                }
            });
        } else {
            eventPublisher.publishEvent(event);
        }
    }
}
```

- 상위 이벤트들은 `AIModelIndexingListener.java`가 `@TransactionalEventListener(phase = AFTER_COMMIT)` 형태로 구독하며, 생성·수정 이벤트는 `AIModelJpaRepository`에서 모델을 재조회한 뒤 `ElasticsearchIndexService#indexModel`로 색인을 갱신하고 삭제 이벤트는 `deleteModelIndex`를 호출해 문서를 제거합니다.

```java
// model/command/domain/service/AIModelIndexingListener.java
@Async
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void onModelCreated(ModelCreatedEvent event) {
    aiModelRepository.findById(event.getModelId()).ifPresent(indexService::indexModel);
}

@Async
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void onModelUpdated(ModelUpdateEvent event) {
    aiModelRepository.findById(event.getModelId()).ifPresent(indexService::indexModel);
}

@Async
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void onModelDeleted(ModelDeletedEvent event) {
    indexService.deleteModelIndex(event.getModelId());
}
```

- 이벤트 스트림은 캐시 서비스에도 그대로 전달되어 모델 검색 파이프라인 전반의 캐시·색인 일관성을 유지합니다.

### 인덱싱 흐름과 배치 구조

#### 실시간 인덱싱 리스너

- `AIModelIndexingListener`는 모델 생성·수정·삭제 이벤트를 `@TransactionalEventListener(AFTER_COMMIT)`로 비동기 수신해 커밋 직후에만 색인을 수행합니다.
- 생성·수정 이벤트는 JPA에서 모델을 재조회해 최신 스냅샷을 확보한 뒤 `ElasticsearchIndexService#indexModel`로 위임하고, 삭제 이벤트는 문서 ID(모델 ID 문자열)를 기반으로 즉시 제거해 일관성을 유지합니다.

<details markdown="1">
<summary>`AIModelIndexingListener.java`</summary>

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class AIModelIndexingListener {

    private final ElasticsearchIndexService indexService;
    private final AIModelJpaRepository aiModelRepository;

    @Async
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onModelCreated(ModelCreatedEvent event) {
        log.info("ModelCreatedEvent 수신 - modelId={}", event.getModelId());
        aiModelRepository.findById(event.getModelId()).ifPresent(indexService::indexModel);
    }

    @Async
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onModelUpdated(ModelUpdateEvent event) {
        log.info("ModelUpdateEvent 수신 - modelId={}", event.getModelId());
        aiModelRepository.findById(event.getModelId()).ifPresent(indexService::indexModel);
    }

    @Async
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onModelDeleted(ModelDeletedEvent event) {
        log.info("ModelDeletedEvent 수신 - modelId={}", event.getModelId());
        indexService.deleteModelIndex(event.getModelId());
    }
}
```

</details>

#### Elasticsearch 인덱스 서비스

- `ElasticsearchIndexService#indexModel`은 소유자 이름, 사용량·조회수, 평점·리뷰 수까지 조회해 `AIModelDocument`를 upsert합니다. 조회수/사용량은 `ModelStatistics` 테이블에서 관리되고, 증가는 `ModelViewCountService`가 JPA 트랜잭션으로 처리합니다.
- `deleteModelIndex`는 잘못된 문서가 남지 않도록 즉시 삭제하고, `syncAllModelsToElasticsearch`는 JOIN + 리뷰 일괄 조회(총 2회의 쿼리)로 전체 재색인을 수행합니다.
- 인덱스 재생성, 상태 조회, 통계 조회 유틸리티를 함께 제공해 운영 점검과 복구를 지원합니다.

<details markdown="1">
<summary>`ElasticsearchIndexService.java` 주요 메서드</summary>

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class ElasticsearchIndexService {

    private final ElasticsearchTemplate elasticsearchTemplate;
    private final AIModelJpaRepository aiModelJpaRepository;
    private final AIModelSearchRepository aiModelSearchRepository;
    private final ModelStatisticsJpaRepository modelStatisticsRepository;
    private final MemberJpaRepository memberRepository;
    private final ReviewRepository reviewRepository;

    public void indexModel(AIModel aiModel) {
        try {
            String ownerName = getOwnerName(aiModel);
            Long usageCount = getUsageCount(aiModel);
            Long viewCount = getViewCount(aiModel);
            Double rating = getAverageRating(aiModel);
            Long reviewCount = getReviewCount(aiModel);

            AIModelDocument document = AIModelDocument.from(
                aiModel, ownerName, usageCount, viewCount, rating, reviewCount);
            aiModelSearchRepository.save(document);
            log.info("Elasticsearch에 모델 색인 완료: modelId={}", aiModel.getId());
        } catch (Exception e) {
            log.error("모델 색인 실패: modelId={}, error={}", aiModel.getId(), e.getMessage());
        }
    }

    public void deleteModelIndex(Long modelId) {
        try {
            aiModelSearchRepository.deleteById(modelId.toString());
            log.info("Elasticsearch에서 모델 인덱스 삭제 완료: modelId={}", modelId);
        } catch (Exception e) {
            log.error("모델 인덱스 삭제 실패: modelId={}, error={}", modelId, e.getMessage());
        }
    }

    public long syncAllModelsToElasticsearch() {
        try {
            List<ModelWithStatisticsProjection> projections =
                    aiModelJpaRepository.findAllModelsWithStatisticsAndOwner();
            if (projections.isEmpty()) {
                return 0;
            }

            List<Long> modelIds = projections.stream()
                    .map(p -> p.getModel().getId())
                    .toList();

            Map<Long, Double> ratingMap = buildRatingMap(modelIds);
            Map<Long, Long> reviewCountMap = buildReviewCountMap(modelIds);

            aiModelSearchRepository.deleteAll();

            long indexedCount = 0;
            for (ModelWithStatisticsProjection projection : projections) {
                AIModel model = projection.getModel();
                String ownerName = projection.getOwnerName() != null ?
                        projection.getOwnerName() :
                        (model.getOwnType() != null ? model.getOwnType().name() : "ADMIN");

                ModelStatistics stats = projection.getStatistics();
                Long usageCount = stats != null ? stats.getUsageCount() : 0L;
                Long viewCount = stats != null ? stats.getViewCount() : 0L;

                Double rating = ratingMap.getOrDefault(model.getId(), 0.0);
                Long reviewCount = reviewCountMap.getOrDefault(model.getId(), 0L);

                AIModelDocument document = AIModelDocument.from(
                    model, ownerName, usageCount, viewCount, rating, reviewCount);
                aiModelSearchRepository.save(document);
                indexedCount++;
            }

            return indexedCount;
        } catch (Exception e) {
            log.error("AI 모델 동기화 중 오류 발생", e);
            throw new RuntimeException("동기화 실패", e);
        }
    }
}
```

</details>

#### 증분 배치 & 전체 동기화

- `AIModelIndexScheduler`는 `app.batch.aimodel-index.enabled=true`일 때 2분 주기로 최근 5분 이내 수정된 모델을 대상으로 증분 배치를 실행하며, 조회수·사용량 같은 변동 데이터도 `ModelStatistics`에 축적된 값을 반영합니다.
- 배치 잡은 `RepositoryItemReader` → `Processor` → `Writer` 순서로 모델·통계·리뷰 집계를 읽어 `AIModelDocument`로 변환하고, `TaskExecutor`로 제한된 병렬도(4)에서 벌크 저장해 처리량과 안정성을 확보합니다.
- 전체 재색인용 `syncAllModelsToElasticsearch`는 하이브리드 쿼리 전략(모델/통계/소유자 JOIN + 리뷰 일괄 조회)으로 정합성 복구와 대량 업데이트를 지원합니다.

<details markdown="1">
<summary>`AIModelIndexScheduler.java`</summary>

```java
@Slf4j
@Component
@ConditionalOnProperty(name = "app.batch.aimodel-index.enabled", havingValue = "true", matchIfMissing = false)
public class AIModelIndexScheduler {

    private final JobLauncher jobLauncher;
    private final Job aiModelIndexJob;

    @Scheduled(fixedRate = 120000)
    public void runIncrementalSync() {
        try {
            LocalDateTime fromDateTime = LocalDateTime.now().minusMinutes(5);

            JobParameters jobParameters = new JobParametersBuilder()
                    .addLocalDateTime("timestamp", LocalDateTime.now())
                    .addLocalDateTime("fromDateTime", fromDateTime)
                    .addString("syncType", "incremental")
                    .toJobParameters();

            log.info("AIModel 5분 증분 인덱싱 배치 시작 - fromDateTime: {} (updatedAt 기준)", fromDateTime);
            jobLauncher.run(aiModelIndexJob, jobParameters);

        } catch (Exception e) {
            log.error("AIModel 증분 인덱싱 배치 실행 실패", e);
        }
    }
}
```

</details>

## 03. 검색 서비스 & API

### REST 엔드포인트 구성

- `AIModelSearchController`는 `/models/search`, `/models/search/admin`, `/models/search/my-models`, `/models/search/owner/{ownerId}`, `/models/search/suggestions`, `/models/search/recommended` 등 엔드포인트를 `PageResponse` 포맷으로 노출합니다.
- 통합 검색·관리자 모델·내 모델 API는 `CachedModelSearchService`를 통해 캐시와 파일 조합 로직을 공유하고, 소유자 검색과 자동완성은 `AIModelSearchService`를 직접 호출합니다.

<details markdown="1">
<summary>`AIModelSearchController.java`</summary>

```java
@RestController
@RequestMapping("/models/search")
@RequiredArgsConstructor
public class AIModelSearchController {

    private final AIModelSearchService searchService;
    private final CachedModelSearchService cachedSearchService;

    @GetMapping
    public ResponseEntity<?> searchModels(String keyword, Boolean isFree, int page, int size) {
        PageResponse<AIModelSearchResponse> result = cachedSearchService.search(keyword, isFree, page, size);
        return ResponseEntity.ok(ApiUtils.success(result));
    }

    @GetMapping("/admin")
    public ResponseEntity<?> getAdminModels(String keyword, Boolean isFree, int page, int size) {
        PageResponse<AIModelSearchResponse> result = cachedSearchService.getAdminModels(keyword, isFree, page, size);
        return ResponseEntity.ok(ApiUtils.success(result));
    }

    @GetMapping("/my-models")
    public ResponseEntity<?> getMyModels(String keyword, Boolean isFree, int page, int size,
                                         @AuthenticationPrincipal CustomUserDetails userDetails) {
        Long userId = userDetails.getMemberId();
        Page<AIModelDocument> result = searchService.getUserModels(keyword, isFree, userId, page, size);
        return ResponseEntity.ok(ApiUtils.success(PageResponse.from(result)));
    }

    @GetMapping("/owner/{ownerId}")
    public ResponseEntity<?> searchByOwner(@PathVariable Long ownerId, int page, int size) {
        Page<AIModelDocument> result = searchService.searchByOwner(ownerId, page, size);
        return ResponseEntity.ok(ApiUtils.success(PageResponse.from(result)));
    }

    @GetMapping("/suggestions")
    public ResponseEntity<?> getModelNameSuggestions(String prefix) {
        List<String> suggestions = cachedSearchService.getModelNameSuggestions(prefix);
        return ResponseEntity.ok(ApiUtils.success(suggestions));
    }
}
```

</details>

### 검색 서비스 로직

- `AIModelSearchService#search`는 키워드·무료 여부 조합에 따라 `_score` 또는 `createdAt` 정렬을 전환하고, 무료/유료 전용 메서드와 키워드 조합 메서드를 구분해 호출합니다.
- 관리자 전용 `getAdminModels`는 항상 공개된 `ADMIN` 모델만 반환하고, 사용자 전용 `getUserModels`는 소유자 ID 조건으로 본인 모델 전체(공개·비공개)를 조회합니다.
- `getModelNameSuggestions`는 Elasticsearch Completion Suggester를 직접 호출해 최대 10건의 중복 없는 추천을 반환합니다.

<details markdown="1">
<summary>`AIModelSearchService.java` 핵심 메서드</summary>

```java
@Slf4j
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class AIModelSearchService {

    private final AIModelSearchRepository searchRepository;
    private final ElasticsearchClient elasticsearchClient;

    public Page<AIModelDocument> search(String keyword, Boolean isFree, int page, int size) {
        Pageable pageable = PageRequest.of(page, size, Sort.by("_score").descending());
        if ((keyword == null || keyword.trim().isEmpty()) && isFree != null) {
            pageable = PageRequest.of(page, size, Sort.by("createdAt").descending());
            return isFree ? searchRepository.findFreeModels(pageable)
                          : searchRepository.findPaidModels(pageable);
        }
        if (keyword != null && !keyword.trim().isEmpty() && isFree != null) {
            pageable = PageRequest.of(page, size, Sort.by("_score").descending());
            return isFree ? searchRepository.searchFreeModelsWithKeyword(keyword, pageable)
                          : searchRepository.searchPaidModelsWithKeyword(keyword, pageable);
        }
        if (keyword != null && !keyword.trim().isEmpty()) {
            return searchRepository.searchByModelNameAndPrompt(keyword, pageable);
        }
        pageable = PageRequest.of(page, size, Sort.by("createdAt").descending());
        return searchRepository.findByIsPublic(true, pageable);
    }

    public List<String> getModelNameSuggestions(String prefix) {
        Suggester suggester = Suggester.of(s -> s
            .suggesters("model-name-suggest", st -> st
                .prefix(prefix)
                .completion(c -> c.field("suggest").size(10).skipDuplicates(true))
            )
        );
        SearchRequest req = SearchRequest.of(b -> b
            .index("ai-models")
            .suggest(suggester)
            .source(src -> src.fetch(false))
            .size(0)
        );
        SearchResponse<Void> resp = elasticsearchClient.search(req, Void.class);
        return resp.suggest().get("model-name-suggest").stream()
                .flatMap(s -> s.completion().options().stream())
                .map(CompletionSuggestOption::text)
                .distinct()
                .toList();
    }
}
```

</details>

### 캐시 & 파일 조합 서비스

- `CachedModelSearchService`는 키워드가 없는 인기 조회(페이지 ≤2, 사이즈 ≤20)만 Redis에 캐싱하고, 캐시 미스 시 로그를 남깁니다.
- 검색 결과에 필요한 이미지 URL은 `FileService#getImageUrlsMap`으로 일괄 조회해 N+1 문제를 방지합니다.
- 관리자 전용 검색도 동일 조건으로 캐싱하며, 자동완성 제안은 캐시 없이 직접 서비스 메서드를 호출합니다.

<details markdown="1">
<summary>`CachedModelSearchService.java` 주요 로직</summary>

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
@Slf4j
public class CachedModelSearchService {

    private final AIModelSearchService searchService;
    private final FileService fileService;

    @Cacheable(
            value = "modelSearch",
            key = "T(...ModelSearchCacheKey).generate(#keyword, #isFree, #page, #size)",
            condition = "#keyword == null && #page <= 2 && #size <= 20",
            unless = "#result == null || #result.empty()"
    )
    public PageResponse<AIModelSearchResponse> search(String keyword, Boolean isFree, int page, int size) {
        log.debug("[CACHE MISS] modelSearch -> keyword:{}, isFree:{}, page:{}, size:{}", keyword, isFree, page, size);
        Page<AIModelDocument> models = searchService.search(keyword, isFree, page, size);
        return toPageResponse(models);
    }

    private PageResponse<AIModelSearchResponse> toPageResponse(Page<AIModelDocument> models) {
        if (models.isEmpty()) {
            return PageResponse.empty(models.getNumber(), models.getSize());
        }
        List<Long> modelIds = models.getContent().stream()
                .map(AIModelDocument::getModelId)
                .toList();
        Map<Long, List<String>> imageUrlsMap = fileService.getImageUrlsMap(modelIds);

        List<AIModelSearchResponse> responses = models.getContent().stream()
                .map(document -> {
                    List<String> imageUrls = imageUrlsMap.getOrDefault(document.getModelId(), List.of());
                    return AIModelSearchResponse.from(document, imageUrls);
                })
                .toList();

        Page<AIModelSearchResponse> responsePage = new PageImpl<>(responses, models.getPageable(), models.getTotalElements());
        return PageResponse.from(responsePage);
    }
}
```

</details>

## 04. 캐시 전략 & 파일 조합

앞서 살펴본 `CachedModelSearchService` 코드처럼 키워드가 없는 무료/유료 목록(사용자·관리자 모델 포함)을 조건별로 캐싱하고 파일 URL을 일괄 조합하는 구조를 기반으로, 아래 캐시 무효화 체계가 연동됩니다.

### 캐시 무효화 & 운영

- `SmartCacheEvictionService`는 모델/리뷰 이벤트를 AFTER_COMMIT 으로 구독해 변경 유형에 따라 검색·상세 캐시를 선택적으로 무효화하고, 지연 무효화를 위해 `LazyInvalidationService`에 더티 마킹을 남깁니다.
- `ModelCacheEvictionService`는 검색 캐시 전체 삭제, 특정 키 삭제, 비동기 삭제 등 저수준 캐시 조작을 담당하며 `SmartCacheEvictionService`와 배치 로직이 이를 호출합니다.
- `LazyInvalidationService`는 Redis에 더티 키를 기록하고 주기적으로 순회해 캐시를 비운 뒤 통계를 남기며, 긴급 시 즉시 처리와 상태 조회 기능을 제공합니다.

<details markdown="1">
<summary>`SmartCacheEvictionService.java`</summary>

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class SmartCacheEvictionService {

    private final ModelCacheEvictionService cacheEvictionService;
    private final ModelCacheService modelCacheService;
    private final LazyInvalidationService lazyInvalidationService;

    @Async
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onModelCreated(ModelCreatedEvent event) {
        if (event.isPublic()) {
            cacheEvictionService.evictCache("modelSearch");
            if ("ADMIN".equalsIgnoreCase(event.getOwnType())) {
                lazyInvalidationService.markSearchCacheDirty("adminModels");
            }
        }
    }

    @Async
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onModelUpdated(ModelUpdateEvent event) {
        switch (event.getUpdateType()) {
            case "PRICE" -> handlePriceUpdate(event);
            case "VISIBILITY" -> handleVisibilityUpdate(event);
            case "BASIC_INFO" -> handleBasicInfoUpdate(event);
            case "FILES" -> handleFilesUpdate(event);
            default -> modelCacheService.updateModelDetailCache(event.getModelId());
        }
    }

    @Async
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onModelDeleted(ModelDeletedEvent event) {
        cacheEvictionService.evictOnModelDelete(event.getModelId());
    }

    @Async
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onReviewChanged(ReviewEvent event) {
        modelCacheService.updateModelDetailCache(event.getModelId());
        lazyInvalidationService.markSearchCacheDirty("modelSearch");
        lazyInvalidationService.markSearchCacheDirty("adminModels");
    }

    public void emergencyEviction(Long modelId, String reason) {
        cacheEvictionService.evictSpecificCacheKey("modelDetail", modelId);
        cacheEvictionService.evictAllSearchCaches();
        lazyInvalidationService.clearAllMarks();
        logEmergencyAction(modelId, reason);
    }
}
```

</details>

<details markdown="1">
<summary>`ModelCacheEvictionService.java`</summary>

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class ModelCacheEvictionService {

    private final CacheManager cacheManager;

    @Caching(evict = {
            @CacheEvict(value = "modelDetail", key = "#modelId"),
            @CacheEvict(value = "modelSearch", allEntries = true)
    })
    public void evictOnModelDelete(Long modelId) {
        log.info("모델 삭제로 인한 캐시 무효화: modelId={}", modelId);
    }

    public void evictAllSearchCaches() {
        evictCaches(Arrays.asList("modelSearch", "adminModels"));
    }

    public void evictSpecificCacheKey(String cacheName, Object key) {
        var cache = cacheManager.getCache(cacheName);
        if (cache != null) {
            cache.evict(key);
        }
    }

    @Async
    public void evictCachesAsync(List<String> cacheNames) {
        cacheNames.forEach(cacheName -> {
            var cache = cacheManager.getCache(cacheName);
            if (cache != null) {
                cache.clear();
            }
        });
    }

    public void evictCache(String cacheName) {
        var cache = cacheManager.getCache(cacheName);
        if (cache != null) {
            cache.clear();
        }
    }

    public void evictCaches(List<String> cacheNames) {
        cacheNames.forEach(this::evictCache);
    }
}
```

</details>

<details markdown="1">
<summary>`LazyInvalidationService.java`</summary>

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class LazyInvalidationService {

    private final RedisTemplate<String, String> redisTemplate;
    private final ModelCacheEvictionService cacheEvictionService;

    private static final String DIRTY_SEARCH_PREFIX = "cache:dirty:search:";
    private static final String BATCH_STATS_KEY = "cache:batch_stats";

    public void markSearchCacheDirty(String cacheName) {
        String key = DIRTY_SEARCH_PREFIX + cacheName;
        String timestamp = LocalDateTime.now().format(DateTimeFormatter.ISO_LOCAL_DATE_TIME);
        redisTemplate.opsForHash().put(key, "ALL", timestamp);
        redisTemplate.expire(key, 1, TimeUnit.HOURS);
    }

    public void processDirtySearchCaches() {
        Set<String> dirtyKeys = redisTemplate.keys(DIRTY_SEARCH_PREFIX + "*");
        if (dirtyKeys == null || dirtyKeys.isEmpty()) {
            return;
        }
        for (String dirtyKey : dirtyKeys) {
            String cacheName = dirtyKey.substring(DIRTY_SEARCH_PREFIX.length());
            Set<Object> dirtyItems = redisTemplate.opsForHash().keys(dirtyKey);
            if (dirtyItems.isEmpty()) {
                redisTemplate.delete(dirtyKey);
                continue;
            }
            cacheEvictionService.evictCache(cacheName);
            redisTemplate.delete(dirtyKey);
        }
        recordBatchStats("search_cache", dirtyKeys.size());
    }

    public void processAllDirtyImmediately() {
        processDirtySearchCaches();
    }

    public LazyInvalidationStatusResponse getStatus() {
        long searchCount = getDirtyCacheNames().size();
        BatchStatisticsResponse batchStats = getBatchStatistics();
        return new LazyInvalidationStatusResponse(
                "LazyInvalidationService",
                searchCount,
                0L,
                batchStats,
                LocalDateTime.now()
        );
    }
}
```

</details>

## 05. 모델 상세·리뷰·UI 개선

### 모델 상세 조회 구조 개편

모델 상세 화면은 정적 정보(모델 소개, 태그)와 변동 정보(통계, 최신 리뷰)를 한 번에 제공해야 했습니다. 
초기에는 단일 API에서 모델·리뷰·파일 데이터를 순차적으로 조회해 응답마다 다중 DB 호출이 발생했고, 요청이 몰리면 지연이 크게 늘었습니다. 
정적 블록(모델 기본 정보, 이미지 등)은 매번 값이 크게 바뀌지 않는다는 점에 주목해 Redis 캐시 어사이드 도입을 검토했고, 그 결과 `CachedModelDetailService`가 정적 블록을 캐시에서 제공하면서 모델·파일 정보를 매번 JPA로 불러오지 않도록 개선했습니다. 
변동 정보는 요청 시점마다 `ModelStatisticsService`와 리뷰 리포지토리를 통해 즉시 조립해 최신성을 확보하며, 최종 응답은 `AIModelDetailResponse.of`가 정적·동적·리뷰 데이터를 합성하도록 분리되어 있습니다. Elasticsearch 증분 동기화는 검색 인덱스 유지를 위한 별도 경로로 활용되고, 상세 화면 통계와 조회수는 RDB + Redis 조합으로 직접 관리됩니다.

<details markdown="1">
<summary>`ModelStatisticsService.java`</summary>

```java
@Service
@RequiredArgsConstructor
public class ModelStatisticsService {

    private final ModelStatisticsJpaRepository modelStatisticsRepository;
    private final ReviewRepository modelReviewRepository;

    @Transactional(readOnly = true)
    public AIModelDynamicStats getDynamicStats(Long modelId, Long memberId) {

        ModelStatistics stats = modelStatisticsRepository.findByModelId(modelId).orElse(null);

        Double avgRating = modelReviewRepository.calculateAverageRatingByModelId(
                modelId, ReviewStatus.ACTIVE
        );
        Long reviewCount = modelReviewRepository.countByModelIdAndStatus(
                modelId, ReviewStatus.ACTIVE
        );

        return AIModelDynamicStats.builder()
                .avgRating(avgRating != null ? avgRating : 0.0)
                .reviewCount(reviewCount)
                .usageCount(stats != null ? stats.getUsageCount() : 0L)
                .viewCount(stats != null ? stats.getViewCount() : 0L)
                .build();
    }

    @Transactional
    public void createInitialStatistics(AIModel model) {
        ModelStatistics statistics = ModelStatistics.createInitialStatistics(model);
        modelStatisticsRepository.save(statistics);
    }
}
```

</details>

<details markdown="1">
<summary>`AIModelDetailFacadeService.java`</summary>

```java
@Service
@RequiredArgsConstructor
public class AIModelDetailFacadeService {

    private final CachedModelDetailService cachedModelDetailService;
    private final ModelStatisticsService statisticsService;
    private final ReviewRepository reviewRepository;

    public AIModelDetailResponse getModelDetail(Long modelId, Long memberId) {
        AIModelStaticDetail staticDetail = cachedModelDetailService.getModelStaticDetailWithView(modelId);
        AIModelDynamicStats dynamicStats = statisticsService.getDynamicStats(modelId, memberId);
        List<ReviewResponse> reviews = reviewRepository
                .findByModelIdAndStatus(modelId, ReviewStatus.ACTIVE)
                .stream()
                .map(ReviewResponse::from)
                .toList();

        return AIModelDetailResponse.of(staticDetail, dynamicStats, reviews);
    }
}
```

</details>

<details markdown="1">
<summary>`ModelViewCountService.java`</summary>

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class ModelViewCountService {

    private final ViewCountThrottleService throttleService;
    private final ModelStatisticsJpaRepository statisticsRepository;

    @Async("viewCountExecutor")
    @Transactional
    public void processViewCountAsync(Long modelId, Long memberId) {
        if (!throttleService.canIncrementViewCount(modelId, memberId)) {
            log.debug("조회수 증가 스킵 (중복 방지): modelId={}, memberId={}", modelId, memberId);
            return;
        }
        incrementViewCount(modelId);
    }

    private void incrementViewCount(Long modelId) {
        ModelStatistics statistics = getModelStatistics(modelId);
        statistics.incrementViewCount();
        statisticsRepository.save(statistics);
    }

    private ModelStatistics getModelStatistics(Long modelId) {
        return statisticsRepository.findByModelId(modelId)
                .orElseThrow(() -> new ApplicationException(ErrorCode.MODEL_NOT_FOUND));
    }
}
```

</details>

<details markdown="1">
<summary>`CachedModelDetailService.java`</summary>

```java
@Slf4j
@Service
public class CachedModelDetailService {

    private final AIModelDetailService modelDetailService;
    private final ModelViewCountService viewCountService;
    private final CachedModelDetailService self;

    public CachedModelDetailService(AIModelDetailService modelDetailService,
                                   ModelViewCountService viewCountService,
                                   @Lazy CachedModelDetailService self) {
        this.modelDetailService = modelDetailService;
        this.viewCountService = viewCountService;
        this.self = self;
    }

    public AIModelStaticDetail getModelStaticDetailWithView(Long modelId, Long memberId) {
        viewCountService.processViewCountAsync(modelId, memberId);
        return self.getModelStaticDetail(modelId);
    }

    @Cacheable(value = "modelDetail", key = "#modelId", unless = "#result == null")
    @Transactional(readOnly = true)
    public AIModelStaticDetail getModelStaticDetail(Long modelId) {
        log.debug("캐시 미스 - 모델 정적 상세 조회 실행: modelId={}", modelId);
        return modelDetailService.getModelStaticDetail(modelId);
    }
}
```

</details>

### ModelCacheService 리팩터링

과거에는 모델 편집 직후에도 캐시가 늦게 비워지면서 오래된 데이터가 노출되는 일이 잦았습니다. 이를 해결하려고 `ModelCacheService`를 재정비해 `updateModelDetailCache` 중심의 즉시 갱신 패턴을 도입하고, 캐시 갱신은 `@Transactional(readOnly = true)` 경계에서 수행해 본 트랜잭션 롤백과 분리했습니다. 변경 유형별로 “즉시 무효화 vs 지연 배치”를 나눠 캐시 안정성을 유지하면서 재빌드 비용을 줄이는 구조입니다.

- **가격/공개 변경**: `handlePriceUpdate`, `handleVisibilityUpdate`가 상세 캐시를 즉시 갱신하고 검색 캐시를 바로 비우거나 덮어쓴 뒤 더티 마킹을 남깁니다.
- **리뷰 변경**: 상세 캐시는 즉시 갱신하지만 검색 캐시는 `LazyInvalidationService`가 처리하도록 지연 무효화합니다.
- **기본 정보·파일 변경**: 상세 캐시만 갱신하고 검색 캐시는 더티 마킹 후 배치가 비우도록 합니다.
- **모델 삭제**: `evictOnModelDelete` 호출로 상세·검색 캐시를 모두 즉시 제거합니다.

### 리뷰 API 분리 및 응답 통합

리뷰 데이터는 건수가 많아질수록 모델 상세 응답을 지연시키는 주된 원인이었습니다. 현재 상세 API(`/models/{id}`)는 최신 리뷰 목록을 함께 반환하고, 필요할 때 `/models/{id}/reviews` 엔드포인트를 통해 모델 정보와 리뷰 정보를 컴포넌트 단위로 따로 호출할 수 있도록 API가 마련되어 있습니다. 두 데이터를 완전히 분리해 컴포넌트별로 불러오는 개선은 추후 과제로 남겨두었습니다.

## 06. 테스트 & 운영 관찰

- `AIModelSearchControllerIntegrationTest`는 통합 검색, 소유자별 필터, 자동완성 API를 `MockMvc`로 검증해 JSON 구조와 페이징 정보를 보장합니다.
- `AIModelSearchService#getModelNameSuggestions`는 prefix·결과 건수를 디버그 로그로 남겨 추천 품질을 추적합니다.
- `LazyInvalidationService`는 배치 수행 횟수와 마지막 실행 시각을 Redis에 기록해 7일간 모니터링 데이터를 유지합니다.
- `SmartCacheEvictionService#emergencyEviction`은 구조화된 로그로 모델 ID·사유·타임스탬프를 남기고 향후 알림 연동 지점을 열어 둡니다.
- 인덱스 상태는 `ElasticsearchIndexService#getIndexStats`로 문서 수와 존재 여부를 확인해 운영 대시보드에 연계할 수 있습니다.

## 07. 한계 및 향후 개선

- 현재 검색 쿼리와 캐시 조건이 분기마다 제각각이어서 유지보수가 어렵습니다. 공통 파라미터 객체와 전략 패턴을 도입해 필터 조합을 선언형으로 다루는 방안을 검토 중입니다.
- 색인/검색 실패를 포착할 전용 알림이 없어 장애 시 로그에 의존하고 있습니다. Elasticsearch 응답 시간을 계량하고 APM·알림 시스템과 연동하는 방향으로 보강할 계획입니다.
- 키워드 없는 요청만 캐싱하고 있어 인기 키워드나 개인화 검색에는 대응하지 못합니다. 통계 수집과 적응형 TTL을 기반으로 확장 캐시 전략을 실험할 예정입니다.
- 모델 상세/검색 캐시 무효화가 Redis 단일 노드에 묶여 있어, 장애 시 검색까지 영향을 받을 수 있습니다. 클러스터 구성과 서킷 브레이커·폴백을 도입하는 시나리오를 준비하고 있습니다.
- 자동완성과 본문 검색이 동일 인덱스를 사용하므로 인덱스 장애가 전면 중단으로 이어집니다. 멀티 인덱스 전략이나 RDB 기반 폴백 검색을 마련해 리스크를 분산하려고 합니다.
- 도메인 이벤트는 `TransactionSynchronization` 기반으로 애플리케이션 내부에만 전파되고 있습니다. 외부 시스템 연동과 전달 보장을 위해 트랜잭셔널 아웃박스 패턴을 검토해 메시지 지속성과 재처리 가능성을 확보하려고 합니다.

## 관련 레포지토리

- [NoModel 백엔드](https://github.com/prgrms-aibe-devcourse/AIBE2_FinalProject_NoModel_BE)
- [NoModel 프론트엔드](https://github.com/prgrms-aibe-devcourse/AIBE2_FinalProject_NoModel_FE)
