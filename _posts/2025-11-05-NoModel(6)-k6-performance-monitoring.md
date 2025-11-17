---
title: "k6 기반 모델 검색 테스트·모니터링 파이프라인"
date: 2025-11-05 10:00:00 +0900
categories:
  - 운영
  - 테스트
tags:
  - k6
  - 성능테스트
  - 모니터링
  - Grafana
  - InfluxDB
---

모델 검색 품질을 안정적으로 유지하려면 기능 검증과 성능 관찰을 한 번에 수행할 수 있는 파이프라인이 필요합니다. 이 글에서는 k6 시나리오부터 InfluxDB·Grafana 기반 모니터링까지, AI 모델 검색 도메인을 검증하는 전체 흐름을 정리합니다.

## 실행 파이프라인
- 최상위 스크립트 `k6/run-tests.sh`가 공용 시나리오를 실행하며 `smoke|load|stress|spike` 옵션과 `--influxdb`, `--prometheus` 플래그를 지원합니다.
- 검색 전용은 `k6/aimodel/run-search-tests.sh`로 분리되어 있고, 선택한 시나리오에 맞춰 `grafana/k6:latest` 컨테이너를 기동한 뒤 결과를 `k6/results/aimodel/*.json`에 저장합니다.
- 두 스크립트 모두 실행 전에 `/api/actuator/health`를 호출해 Spring 애플리케이션이 준비됐는지 확인하고, 종료 시 `k6/utils/influxdb-utils.sh`의 `trap` 처리로 k6 컨테이너와 InfluxDB를 정리합니다.
- InfluxDB 사용 시 스크립트가 `docker-compose-k6.yml`을 통해 컨테이너를 올리고, 테스트가 끝나면 자동으로 내립니다.

## 시나리오 구성
- `k6/scenarios/smoke.js`는 `vus: 1`, `duration: 1m` 고정 구성과 `http_req_duration{p(95)} < 300ms` 임계값을 사용합니다. 루프마다 `/actuator/health` → `/models/search` → `/models/search/admin` → `/models/search/suggestions` 엔드포인트만 순차 호출해 기본 검색 흐름과 관리자 API의 응답성을 초단위로 점검합니다.
- `k6/scenarios/load.js`는 하나의 스크립트 안에 load/stress/spike 프리셋을 담아두고 `TEST_TYPE` 환경 변수로 원하는 스테이지·threshold 세트를 선택해 실행합니다. load 모드는 목표 VU를 `10 → 25 → 60 → 25 → 0`으로 계단식 조정하고, stress는 `80 → 160 → 240 → 120 → 0`, spike는 `0 → 200 → 0`으로 즉시 상승·하강합니다. 모든 모드는 루프에서 건강 상태 → 공개 검색 → 가격 필터 검색 → 관리자 검색 → 자동완성 순으로 호출하며, `http_req_duration`과 `error_rate` 임계값을 모드에 따라 별도로 적용합니다(예: load p95<500ms, error_rate<1%, stress p95<800ms, spike 에러율<3%).
- 검색 특화 메인 스크립트 `k6/aimodel/search-test.js`는 smoke, load, load-short, stress, spike 다섯 가지 프리셋을 개별 Stage·Threshold 묶음으로 정의합니다. 프리셋마다 VU/Duration 사다리와 `http_req_duration`, `search_error_rate`, `search_response_time` 한계가 다르고, 한 루프에서 통합 검색, 관리자 검색, 사용자 전용 검색(로그인 인증), 복합 필터, 페이지네이션, 정렬을 모두 실행하며 `price_filter_accuracy`, `search_response_time`, `search_vus_max` 같은 커스텀 메트릭을 수집합니다.
- `k6/aimodel/filter-test.js`는 필터 정확도만 떼어서 `30s@3 → 1m@10 → 2m@15 → 30s@5 → 30s@0` 스테이지를 적용하고, 가격/카테고리/태그/복합 필터를 번갈아 호출합니다. `price_filter_accuracy`, `filter_error_rate`, `filter_response_time{p(95)}` 임계값을 통해 필터별 정확도와 응답시간을 개별적으로 감시합니다.

## 공통 유틸리티와 검증
- `k6/utils/test-data.js`가 키워드, 카테고리, 태그, 페이지 크기 샘플을 무작위로 만들어 시나리오 다양성을 높이고, SLO 비교용 임계값을 제공합니다.
- `k6/utils/common-checks.js`는 JSON 응답 여부, 페이지네이션 필드, 가격 필터 정확도 등을 공통 `check` 로직으로 묶어 실패 시 Rate 메트릭에 누적합니다.
- `k6/utils/auth.js`는 로그인 요청으로 JWT 쿠키를 받아 인증이 필요한 `/models/search/my-models` 호출에 재사용합니다.

## 모니터링 스택
- 성능 테스트는 필요한 순간에만 InfluxDB 1.8을 기동하고, `--out influxdb=http://influxdb:8086/k6` 형태로 세부 메트릭을 적재합니다.
- Grafana(기본 포트 3000, 계정 `admin/admin123`)에서 InfluxDB 데이터 소스를 통해 응답 시간, 에러율, 커스텀 메트릭을 시각화합니다.
- Spring Boot Actuator, Node Exporter, Loki 등은 Prometheus가 15초 간격으로 스크랩하며, `--prometheus` 옵션을 사용하면 k6 메트릭도 Prometheus 라우터로 보낼 수 있습니다.
- `k6/utils/influxdb-utils.sh`는 InfluxDB 상태를 점검하고 필요 시 `docker compose -f docker-compose-k6.yml up -d influxdb` 명령을 실행합니다.

## 성능 지표 활용
- 모든 시나리오는 `--summary-export`로 JSON 요약을 남기며, `k6/aimodel/run-search-tests.sh --results` 명령으로 최신 결과의 p95 응답 시간과 실패율을 빠르게 확인할 수 있습니다.
- JSON 결과는 `jq`나 Python으로 후처리해 회귀 여부를 판별하고, 장기 추세는 Grafana 대시보드에서 관측합니다.
- 커스텀 메트릭(`search_error_rate`, `price_filter_accuracy`, `search_response_time` 등)을 SLA 임계값과 함께 정의했기 때문에, 기능적 실패도 성능 지표와 함께 모니터링할 수 있습니다.

## 운영 팁
- 테스트 실행 전에 애플리케이션과 모니터링 스택의 헬스 체크를 통과했는지 반드시 확인합니다.
- 장시간 부하 테스트 후에는 `--results` 옵션으로 요약 데이터를 검토하고, 필요 시 InfluxDB 컨테이너를 내려 리소스를 회수합니다.
- 자주 사용하는 시나리오는 Cron이나 CI 파이프라인에 연결해 자동으로 실행하고, Grafana 알람으로 SLA 초과 시 빠르게 대응할 수 있도록 구성합니다.

위 구조를 활용하면 기능 검증과 성능 관측을 한 번에 수행하면서도 모니터링 인프라는 필요할 때만 가동하는 유연한 운영이 가능합니다. k6 스크립트의 커스텀 메트릭과 InfluxDB·Grafana 연계를 적극 활용해 검색 품질을 지속적으로 확인해 보세요.
