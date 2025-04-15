---
title: Javascript
date: 2025-04-14 +019:00
categories: [AIBE, FRONT-END]
tags: [frontend, javascript]
---

### DOM

- 요소를 추가하기 위한 준비

```javascript
$(document).ready(function() {

});
```

##### 객체 편집 메소드

- append() : 선택한 요소의 마지막 위치(내부)에 새 요소 추가
- prepend() : 선택한 요소의 맨 앞 위치(내부)에 새 요소 추가
- after() : 선택한 요소의 다음 위치에 새 요소 추가
- before() : 선택한 요소의 이전 위치에 새 요소 추가
- remove() : 선택한 요소 삭제

#### 이벤트

- 클릭 이벤트

```javascript
$(document).ready(function() {
    $('button').on('click', function() {

    });
});
```

> 예제 - 클릭 이벤트

```javascript
$(document).ready(function() {
    $('button').on('click', function() {
        var price = $('<p></p>');
        $('.vacation').append(price);
        $('button').remove();
    });
});
```

-> 클릭하지 않은 버튼이 삭제될 가능성이 있습니다.

```javascript
$(document).ready(function() {
    $('button').on('click', function() {
        var price = $('<p></p>');
        $('.vacation').append(price);
        $(this).remove();
    });
});
```

-> 클릭하지 않은 버튼 영역에도 p 엘리먼트가 추가될 가능성이 있습니다.

```javascript
$(document).ready(function() {
    $('button').on('click', function() {
        var price = $('<p></p>');
        $(this).append(price);
        $(this).remove();
    });
});
```

-> 만약 의도한 위치보다 button 위치가 안쪽에 있을 경우

```javascript
$(document).ready(function() {
    $('button').on('click', function() {
        var price = $('<p></p>');
        $(this).closest('.vacation').append(price);
        $(this).remove();
    });
});
```

