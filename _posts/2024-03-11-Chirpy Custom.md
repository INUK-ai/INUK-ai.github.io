---
title: Chirpy Custom
date: 2024-03-11 +09:00
categories: [Blog, Config]
tags: [blog, config, custom]
---

## Chirpy Custom

GitHub Blog 작성을 위해 `jekyll-theme-chirpy`을 가져와 제 마음대로 커스텀한 부분들을 기록한 것입니다.

### Site 분석

***Web*** 에서는 `F12` - 개발자 도구 를 통해 해당 사이트를 분석 해볼 수 있습니다.
<br>
개발자 도구 페이지의 **\</>요소** 에서 각 code들에 마우스 커서를 올리면 해당하는 부분들이 하이라이트 됩니다.
<br>
그렇게 제 마음에 안 드는 부분을 구체적으로 나타내주는 부분을 찾을 때까지 파고들어 가다보면 해당 태그의 속성들을 하단 스타일 부분에서 확인할 수 있습니다.
<br>

### Custom


#### 간단 Custom

> *Chirpy* 에서 간단하게 Custom 할 부분을 찾는 방법

- 기본 골자 : **[_layouts]** 내부 *html* 파일
- 속성 : **[_sass]** 내부 *scss* 파일
- 색상 : *html* 파일, *scss* 파일 내에서 변수명으로 설정되어 있을 경우 *[_sass/colors]* 에서 메인 백그라운드 색상에 해당하는 *scss* 파일

모든 파일들의 이름은 직관적으로 이해할 수 있으니 왠만한 부분들은 해당 순서로 찾아 수정하면 됩니다.

#### 번역

```yml
# The language of the webpage › http://www.lingoes.net/en/translator/langcode.htm
# If it has the same name as one of the files in folder `_data/locales`, the layout language will also be changed,
# otherwise, the layout language will use the default value of 'en'.
lang: ko-KR
```

- ***Chirpy***에서는 __config.yml_ 의 'lang' 부분에서 쉽게 적용된 언어를 바꿀 수 있습니다.
    - 물론 만들어져있는 부분에만 해당합니다.
- **Template** 으로 만들어져있는 언어 모음은 **[_data/locales]** 에 있으며 해당하는 약어로 __conofig.yml_ 파일을 수정해주시면 됩니다.

```yml
# ----- Commons label -----

layout:
  post: 포스트
  category: 카테고리
  tag: 태그
```

- 각각의 파일에서 각 _Label_ 에 해당하는 단어들을 직접 수정 가능합니다.

#### 프로필 이미지

```yml
# the avatar on sidebar, support local or CORS resources
avatar: assets/img/avatar.jpg
```

- __config.yml_ 파일의 _avatar_ 부분을 건드리면 간단하게 바꿀 수 있습니다.
- 원하는 프로필 이미지는 **[assets/img/]** 에 넣어주시면 됩니다.

#### Sidebar 배경 이미지

```scss
  @include pl-pr(0);

  position: fixed;
  top: 0;
  left: 0;
  height: 100%;
  overflow-y: auto;
  width: $sidebar-width;
  z-index: 99;
  background: url('/assets/img/sidebarCover.png');
  background-size: 100% 100%;
  background-position: center;
  border-right: 1px solid var(--sidebar-border-color);
```

- [_sass/addon/commons.scss] 의 *#sidebar* 부분에 `background` 관련 코드를 수정하시면 됩니다.

#### 각종 SNS 관련

##### Sidebar_

```liquid
{% raw %}
{% case entry.type %}
{% when 'github', 'twitter' %}
    {%- capture url -%}
    https://{{ entry.type }}.com/{{ site[entry.type].username }}
    {%- endcapture -%}
{% when 'email' %}
    {% assign email = site.social.email | split: '@' %}
    {%- capture url -%}
    javascript:location.href = 'mailto:' + ['{{ email[0] }}','{{ email[1] }}'].join('@')
    {%- endcapture -%}
{% when 'rss' %}
    {% assign url = '/feed.xml' | relative_url %}
{% else %}
    {% assign url = entry.url %}
{% endcase %}
{% endraw %}
```

- 해당 부분을 찾아 원하시는 대로 바꾸면 됩니다.

##### Post_

```yml
platforms:
  # - type: Twitter
  #   icon: "fa-brands fa-square-x-twitter"
  #   link: "https://twitter.com/intent/tweet?text=TITLE&url=URL"

  # - type: Facebook
  #   icon: "fab fa-facebook-square"
  #   link: "https://www.facebook.com/sharer/sharer.php?title=TITLE&u=URL"

  # - type: Telegram
  #   icon: "fab fa-telegram"
  #   link: "https://t.me/share/url?url=URL&text=TITLE"
```

- 해당 부분을 찾아 원하시는 대로 바꾸면 됩니다.