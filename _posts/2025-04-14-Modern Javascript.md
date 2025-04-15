---
title: Javascript
date: 2025-04-15 +019:00
categories: [AIBE, FRONT-END]
tags: [frontend, javascript]
---

## Modern Javascript

#### 특징

- 가상 DOM 을 이용하는 라이브러리 혹은 프레임워크 사용
- npm, yarn 등의 패키지 관리자 사용
- 주로 ES6 이후 표기법 사용
- 모듈 핸들러 사용
- 트랜스파일러 사용
- SPA 로 작성

##### 패키지 관리자

> 패키지 설치 시 패키지 관리, 설치, 업그레이드 등을 전담

- 의존 관계를 의식하지 않아도 자동으로 해결
- 팀 안에서 패키지를 공유하고 버전을 통일하기 쉬움
- 전 세계에 공개된 패키지를 하나의 명령어로 이용 가능

```bash
npm install [패키지명] // package-lock.json
yarn add [패키지명] // yarn.lock
```

`package.json` 이 변경되고 패키지 정보가 추가 됩니다.

```bash
npm install 
yarn add 
```

`package.json`, `package-lock.json` 을 참조해 버전이나 의존 관계가 해결된 상태로 `node_modules` 라는 폴더를 만들고 그 안에 실제 패키지를 전개합니다.

##### 모듈 핸들러

> 개발 시 파일을 나누고 프로덕션 용으로 빌드할 때 파일 하나에 모으기 위해 js 파일, css 파일 등을 한 곳에 합친 것

##### 트랜스파일러

> 자바스크립트 표기법을 브라우저에서 실행할 수 있는 형태로 변환

##### SPA

> HTML 파일은 하나만 사용하고 자바스크립트를 이용해 DOM 을 바꿔 써서 화면 이동을 구현하는 것

- 사용자 경험 향상
    - 서버 측에 요청을 보내지 않고 페이지 이동 가능 -> 화면 표시 속도 향상
- 컴포넌트 분리가 쉬워짐에 따른 개발 효율 향상
    - 화면의 각 요소를 컴포넌트로 정의해서 재사용

### 기초 문법

##### const & let

- var 를 이요한 변수 선언의 문제

- 덮어쓰기
- 재선언 가능
    - 프로그램 실행 순서에 따라 어느 변수가 사용되는지 해석하기 어렵기 때문에 지양

**let**

```javascript
let strLet = "let";
console.log(strLet); // 출력 : let

strLet = "newLet"; // 재할당
console.log(strLet); // 출력 : newLet

let strLet = "reLet"; // 중복 선언 -> 오류 발생
console.log(strLet); // Uncaught SyntaxError
```

**const**

```javascript
const strConst = "const";
console.log(strConst); // 출력 : const

strConst = "newConst"; // 재할당 불가 -> 오류 발생

const strConst = "reConst"; // 동일한 이름으로 재선언 불가 -> 오류 발생
```

*** const 로 정의한 변수를 변경할 수 있는 경우

- Primitive Type 의 데이터는 const 를 이용해 덮어쓸 수 없습니다.
- 객체나 배열 등 Object Type 의 데이터들은 const 로 정의해도 도중에 값 변경이 가능합니다.