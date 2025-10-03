# Repository Guidelines

## 프로젝트 구조 및 모듈 구성
이 저장소는 Chirpy Jekyll 테마를 확장합니다. 블로그 글과 페이지는 `_posts/`(형식: `YYYY-MM-DD-title.md`)에, 탭 콘텐츠는 `_tabs/`에 둡니다. Liquid 레이아웃과 조각은 `_layouts/`와 `_includes/`, 메뉴와 메타데이터는 `_data/`에 정리합니다. 스타일은 `_sass/`, 자바스크립트 소스는 `_javascript/` 내에 두고 Rollup 번들은 `assets/js/dist/`로 출력됩니다. 정적 자산은 `assets/`, 문서와 운영 가이드는 `docs/`, 자동화 스크립트는 `tools/`에 위치합니다.

## 빌드 · 테스트 · 개발 명령어
작업 전 `bundle install`과 `npm install`로 의존성을 설치합니다. 주요 명령은 다음과 같습니다.
- `bundle exec jekyll serve --livereload`: 로컬 미리보기(127.0.0.1:4000) 제공
- `bundle exec jekyll build`: `_site/`에 프로덕션 빌드 생성
- `npm run watch`: `_javascript/`를 감시하며 Rollup 개발 번들 수행
- `npm run build`: `assets/js/dist`를 정리 후 프로덕션 번들 생성

## 코딩 스타일 및 네이밍 규칙
`.editorconfig`에 따라 공백 들여쓰기, 2칸 인덴트, LF 줄바꿈, 파일 끝 개행을 유지합니다. JS·SCSS는 작은따옴표, YAML은 큰따옴표를 사용합니다. 모든 신규 문서와 UI 텍스트는 한국어로 작성하고, 프런트매터에는 `title`, `date`, `categories`, `tags`를 포함합니다. 본문 링크는 즉시 URL을 노출하지 말고 `[문구](URL)` 또는 각주형 매핑으로 표현합니다. 스타일 변경 시 `npm test`를 통과해야 하며 필요하면 `npm run fixlint`로 자동 수정합니다. 파일명은 기능을 드러내도록(`analytics-tracking.js`, `landing.scss`) 작성하고 `_data/` 키는 snake_case를 권장합니다.

## 테스트 가이드라인
커밋 전 `npm test`로 `_sass/`와 공유 스타일을 린트합니다. 전면 검증이 필요하면 `bash ./tools/test -c _config.yml`을 실행해 정적 사이트를 재빌드하고 `htmlproofer` 링크 검증을 수행합니다. 스크립트나 빌드 파이프라인을 변경했다면 자동 테스트가 없다면 수동 확인 절차를 PR 설명에 기록합니다.

## 커밋 및 PR 가이드라인
`git log`에 맞춰 제목은 Conventional 양식을 한국어로 유지합니다(예: `Docs: 운영체제 #4 - 프로세스 동기화`, `Refactor: 프로젝트 설정 파일 추가`). 타입은 대문자로 시작하고, 관련 이슈는 `#번호`로 연결합니다. 커밋은 논리 단위로 쪼개고 빌드·린트를 선행합니다. PR은 변경 요약, 설정 변경 사항, 실행한 테스트 명령, UI 변경 시 스크린샷을 포함하며 관련 이슈나 토론 링크를 첨부해 리뷰어가 맥락을 빠르게 파악하도록 돕습니다.
