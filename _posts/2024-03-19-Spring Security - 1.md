---
title: Spring Security - 1 
date: 2024-03-19 +09:00
categories: [Backend, Spring]
tags: [backend, spring, security, config]
---

## Servlet Authentication Architecture
---

### SecurityContextHolder

![](https://docs.spring.io/spring-security/reference/_images/servlet/authentication/architecture/securitycontextholder.png)
_출처 : [Spring Security Docs][Spring Security]_

> _Spring Security_ 가 인증된 사용자의 세부 정보를 저장하는 곳

```java
SecurityContext context = SecurityContextHolder.createEmptyContext();
Authentication authentication =
    new TestingAuthenticationToken("username", "password", "ROLE_USER");
context.setAuthentication(authentication);

SecurityContextHolder.setContext(context);
```

1. 빈 _SecurityContext_ 를 생성합니다.
    - _Multi Threads_ 에서의 _Race conditions_ 를 피하기 위해 `SecurityContextHolder.getContext().setAuthentication(authentication)` 대신 새 _SecurityContext_ 인스턴스를 만들어야 합니다.
2. 새 _Authentication_ 객체를 생성합니다.
    - _Spring Security_ 는 _SecurityContext_ 에 어떤 유형의 인증 구현이 설정되어 있는지는 신경 쓰지 않습니다.
    - 일반적인 _Production Scenario_ 는 `UsernamePasswordAuthenticationToken(userDetails, password, authorities)` 입니다.
3. 마지막으로 _SecurityContextHolder_ 에서 _SecurityContext_ 를 설정합니다.
    - _Spring Security_ 는 _Authorization_ 을 위해 해당 정보를 사용합니다.

##### 특징

- *ThreadLocal* : 기본 _Strategy_
    - 동일 _Thread_ 내의 메소드들이 명시적으로 _SecurityContext_ 를 전달받지 않아도 이에 접근할 수 있게 해줍니다.
    - 현재 _pricipal_ 의 요청이 처리된 후 _Thread_ 를 정리하는 것에 주의한다면 매우 안전합니다.
- _Spring Security_ 의 _FilterChainProxy_ 는 요청 처리 후 _SecurityContext_ 를 항상 정리해 보안상의 문제를 예방합니다.
- _Strategy_
    - *SecurityContextHolder.MODE_GLOBAL* : JVM 내의 모든 _Thread_ 가 동일한 _SecurityContext_ 사용
    - *SecurityContextHolder.MODE_INHERITABLETHREADLOCAL* : _Security Thread_ 에 의해 생성된 자식 _Thread_ 가 부모의 _Security Identity_ 상속

### SecurityContext

> `SecurityContextHolder`로부터 얻어지며 인증된 사용자의 정보를 담고 있는 `Authentication` 객체를 포함합니다.

### Authentication

##### 1. 사용자가 제공한 자격 증명을 _AuthenticationManager_ 에 입력하여 인증을 시도할 때

- `isAuthenticated()` = false 

##### 2. 현재 인증된 사용자를 나타낼 때

- _SecurityContext_ 에서 현재 _Authentication_ 을 얻을 수 있습니다.

##### 구성

- ***Principal***
    - _UserDetails_ 의 인스턴스
    - 식별된 사용자 정보를 보관
    - 시스템에 따라 _UserDetails_ 클래스를 상속하여 커스텀한 형태로 유지할 수 있습니다.
- ***Credentials***
    - 사용자(주체)가 올바르다는 것을 증명하는 자격 증명
    - 보통 비밀번호를 의미합니다.
- ***Authorities***
    - _AuthenticationManager_ 가 설정한 권한
    - _Authentication_ 상태에 영향을 주지 않거나 수정할 수 없는 인스턴스를 사용해야 합니다.

#### GrantedAuthority

> 사용자에게 부여된 _high-level permission_, 주로 `역할(role)`이나 `범위(scope)`와 같은 형태

- `Authentication.getAuthority()` 메소드를 통해 얻을 수 있는 _GrantedAuthority_ 객체의 _Collection_ 으로 제공됩니다.
- 일반적으로 애플리케이션 전체의 권한을 나타내며 특정 도메인 객체에 국한되지 않습니다.
    - `ROLE_ADMIN`,  `ROLE_USER`와 같은 역할을 통해 사용자가 수행할 수 있는 작업의 범위를 정의합니다.

### AuthenticationManager

> _SpringSecurityFilter_ 가 인증을 수행하는 방법을 정의하는 **API**

- _Filter_ 를 통해 인증이 수행된 후, 반환된 _Authentication_ 객체는 _SecurityContextHolder_ 에 설정됩니다.

### ProviderManager

> _AuthenticationManager_ 의 가장 일반적인 구현체

- _AuthenticationProvider_ 인스턴스로 구성된 목록에 인증 처리를 위임합니다.
- _AuthenticationProvider_ 다음과 같은 판단 기회를 가집니다.
    - 인증이 성공적이었는지, 실패했는지
    - 결정을 내릴 수 없어 다음 _AuthenticationProvider_ 에게 결정을 넘길지
- _ProviderNotFoundException_
    - _AuthenticationProvider_ 가 어느 인증도 수행할 수 없을 시 발생
    - _ProviderManager_ 가 전달된 인증 유형을 지원하도록 구성되지 않았음을 나타냅니다.

![](https://docs.spring.io/spring-security/reference/_images/servlet/authentication/architecture/providermanager.png)
_출처 : [Spring Security Docs][Spring Security]_

각 _AuthenticationProvider_ 는 특정 유형의 인증을 수행하는 방법을 가집니다.
- username / password
- saml assertion 
- etc..

이를 통해 다양한 인증 유형을 지원하면서 단일 _AuthenticationManager Bean_ 을 노출할 수 있습니다.
<br>
만약 모든 _AuthenticationProvider_ 가 인증을 처리할 수 없는 경우, 선택적으로 구성된 부모 _AuthenticationManager_ 에게 인증을 위임합니다.

![](https://docs.spring.io/spring-security/reference/_images/servlet/authentication/architecture/providermanagers-parent.png)
_출처 : [Spring Security Docs][Spring Security]_

여러 _ProviderManager_ 인스턴스가 같은 부모 _AuthenticationManager_ 를 공유할 수 있습니다.
- 특정 인증을 공통으로 사용하면서도 다른 인증 메커니즘을 가진 여러 _SecurityFilterChain_ 인스턴스가 존재하는 시나리오에서 흔히 볼 수 있습니다.

##### 특징

_ProviderManager_ 는 성공적인 인증 후 반환된 _Authentication_ 객체에서 민감한 자격 증명 정보를 제거 시도
- 비밀번호 같은 정보가 _HttpSession_ 에 필요 이상으로 유지되는 것을 방지
- _stateless_ 애플리케이션의 성능 향상을 위해 사용자 객체 캐시를 사용할 때 문제가 발생할 수 있습니다.
    - 자격 증명 정보가 제거된 _UserDetails_ 인스턴스 같은 캐시된 객체에 대해 다시 인증을 시도할 수 없게 되기 때문

    - 해결 방법
        - 객체의 복사본을 생성
        - _ProviderManager_ 의 `eraseCredentialsAfterAuthentication` 속성 비활성화

#### AuthenticationProvider

_ProviderManager_ 는 여러 _AuthenticationProvider_ 인스턴스를 수용할 수 있으며 각각 특정 유형의 인증을 담당합니다.
- _DaoAuthenticationProvider_
- _JwtAuthenticationProvider_
- etc..

#### AuthenticationEntryPoint

> 클라이언트에게 자격 증명을 요청하는 _HTTP_ 응답을 보내는 데 사용

- 인증되지 않은 요청을 할 때 (권한이 없는 리소스에 접근하려 할 때) 클라이언트로부터 자격 증명을 요청하기 위해 사용됩니다.
- 로그인 페이지로의 리다이렉트나 _WWW-Authenticate_ 헤더 응답 등의 동작을 수행할 때 구현이 필요합니다.

### AbstractAuthenticationProcessingFilter

![](https://docs.spring.io/spring-security/reference/_images/servlet/authentication/architecture/abstractauthenticationprocessingfilter.png)
_출처 : [Spring Security Docs][Spring Security]_

> 사용자 _credentials_ 를 인증하기위한 기본 필터

### Reference

✔ Spring Security Docs | [Spring Security][Spring Security]

[Spring Security]: https://docs.spring.io/spring-security/reference/servlet/authentication/architecture.html#servlet-authentication-securitycontextholder