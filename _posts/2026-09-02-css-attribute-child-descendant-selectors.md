---
layout: post
title: "속성 선택자 6개를 썼는데 div가 전부 초록색 — 같은 명시도의 순서, 자손·후손 선택자, font 단축 속성"
date: 2026-09-02 15:00:00 +0900
categories: [Frontend]
tags: [css, selectors, cascade, specificity]
---

## 들어가며 (Situation)

CSS 두 번째 수업 파일 `02_css선택자2.html`은 스타일을 외부 파일 `selector.css`로 옮기면서 시작한다.

```html
<!-- 01 파일에서는 head 영역에 css 언어가 잔뜩 들어가 있었다.
     외부 스타일시트 방식으로 01 파일에서의 문제를 해결해보자. -->
<link rel="stylesheet" href="selector.css">
```

그리고 세 종류의 선택자를 배웠다.

| 종류 | 예 |
|------|----|
| 속성 선택자 | `div[name=name2]`, `div[name~=name1]`, `div[class|=class]` … |
| 자손(`>`) / 후손(공백) 선택자 | `#test1 > h4`, `#test1 ul` |
| 반응·상태 선택자 | `:hover`, `:active`, `:checked`, `:focus`, `:enabled`, `:disabled` |

[이전 글]({% post_url 2026-09-02-css-selectors-and-flexbox-guide %})에서는 선택자 게임으로 형제 선택자를 정리했다. 이번에는 수업 파일이 실제로 어떻게 그려지는지 확인했다.

## 문제 상황 (Task)

속성 선택자 부분은 div 네 개에 규칙 여섯 개를 줬다.

```html
<div name="name1 name5 name6" class="div-class">div1</div>
<div name="name2" class="div-class">div2</div>
<div name="name3" class="div-class">div3</div>
<div name="name4" class="class-div">div4</div>
```

```css
div[name=name2]   { border: 1px; background-color: orange; }
div[name~=name1]  { background-color: yellow; }
div[class|=class] { background: chartreuse; }
div[name^=name]   { background: blue; }
div[class$=class] { background-color: skyblue; }
div[class*=div]   { background-color: seagreen; }
```

규칙마다 다른 색을 줬으니 div마다 다른 색이 나와야 할 것 같았다. 그런데 화면에서는 **네 개가 모두 같은 초록색**이었다. 아래쪽의 `li`도 `aqua`를 기대했는데 색이 없었다.

1. 왜 모든 div가 같은 색일까?
2. `#test1 > ul > li`는 왜 아무것도 칠하지 못했을까?
3. 그 밖에 의도와 다르게 동작한 규칙은 없을까?

## 해결 과정 (Action)

### 1. 규칙마다 어느 div에 맞는지 표로 그리기

먼저 각 선택자가 어느 div에 맞는지 정리했다.

| 선택자 | 뜻 | div1 | div2 | div3 | div4 |
|--------|----|------|------|------|------|
| `[name=name2]` | 값이 정확히 `name2` | | O | | |
| `[name~=name1]` | 공백으로 나눈 단어 중 `name1` | O | | | |
| `[class|=class]` | `class`이거나 `class-`로 시작 | | | | O |
| `[name^=name]` | `name`으로 시작 | O | O | O | O |
| `[class$=class]` | `class`로 끝남 | O | O | O | |
| `[class*=div]` | `div`를 포함 | **O** | **O** | **O** | **O** |

마지막 규칙 `[class*=div]`가 **네 개 모두에 맞는다.** `div-class`와 `class-div` 둘 다 `div`라는 글자를 품고 있기 때문이다.

### 2. 명시도가 같으면 나중에 쓴 규칙이 이긴다

Chrome 헤드리스로 계산된 배경색을 찍었다.

```text
DIV[name1 name5 name6|div-class] bg=rgb(46, 139, 87)
DIV[name2|div-class]             bg=rgb(46, 139, 87)
DIV[name3|div-class]             bg=rgb(46, 139, 87)
DIV[name4|class-div]             bg=rgb(46, 139, 87)
```

`rgb(46, 139, 87)`은 `seagreen`이다. 여섯 규칙은 모두 "태그 1개 + 속성 1개" 형태라서 **명시도가 (0, 1, 1)로 같다.** 속성 선택자는 클래스 선택자와 같은 점수를 받는다. 명시도가 같으면 CSS는 **나중에 나온 규칙**을 적용한다. 그래서 모든 div에 맞는 마지막 규칙이 앞의 다섯 규칙을 전부 덮었다.

앞의 규칙들이 정말 맞았는지 보려면 마지막 규칙을 주석 처리하면 된다. 실습에서 각 선택자를 하나씩 확인하려면 규칙을 하나씩 켜 보거나, 서로 겹치지 않는 대상을 고르는 편이 좋았다. 명시도와 순서는 [다음 수업의 우선순위 실습]({% post_url 2026-09-03-css-specificity-important %})에서 따로 정리했다.

### 3. `border: 1px`은 테두리를 그리지 않는다

div2에 준 `border: 1px`도 확인했다.

```text
borderTop=0px none
```

굵기를 1px로 줬는데 계산된 굵기가 **0px**이고 스타일이 `none`이다. `border` 단축 속성은 굵기, 스타일, 색을 한꺼번에 정한다. 빠진 값은 초기값으로 채워진다. 스타일의 초기값이 `none`이라 테두리가 그려지지 않고, 그려지지 않는 테두리의 굵기는 0으로 계산된다. 테두리를 보려면 `border: 1px solid`처럼 스타일까지 써야 한다.

### 4. `>`는 바로 아래 자식만 고른다

```html
<div id="test1">
    <h4>자손 1번 입니다.</h4>
    <h4>자손 2번 입니다.</h4>
    <div>
        <ul id="testul">리스트
            <li>ul 태그의 자손이면서, div 태그의 후손</li>
            <li>ul 태그의 자손이면서, div 태그의 후손</li>
        </ul>
    </div>
</div>
```

```css
#test1 > h4      { background-color: hotpink; }
#test1 > ul > li { background-color: aqua; }
#test1 ul        { background-color: chocolate; }
```

```text
H4     bg=rgb(255, 105, 180)   (hotpink)
testul bg=rgb(210, 105, 30)    (chocolate)
LI     bg=rgba(0, 0, 0, 0)     (투명)
```

`#test1 > ul > li`는 **아무것도 고르지 못했다.** `ul`은 `#test1`의 바로 아래 자식이 아니다. 중간에 `div`가 하나 더 있다. `>`는 한 단계만 내려가므로 `#test1 > ul`이 성립하지 않는다. `li`가 갈색으로 보인 것은 투명한 `li` 뒤로 부모 `ul`의 `chocolate` 배경이 비쳤기 때문이다.

| 결합자 | 이름 | 범위 | `#test1`에서 `ul`까지 |
|--------|------|------|-----------------------|
| `A > B` | 자식(자손) 결합자 | 바로 아래 한 단계 | 실패 (`div`를 거침) |
| `A B` | 후손 결합자 | 몇 단계 아래든 | 성공 |

수업 메모는 바로 아래를 "자손", 모든 하위를 "후손"이라고 불렀다. 명세 용어로는 `>`가 child combinator, 공백이 descendant combinator다. 고치려면 `#test1 > div > ul > li`나 `#test1 li`라고 쓰면 된다.

### 5. `font: 60px bold`는 굵어지지 않는다

`:hover` 규칙에는 `font: 60px bold;`가 있다. 마우스를 올리는 동작은 헤드리스로 재현하기 어려워서, 같은 값을 준 요소의 계산된 스타일을 찍었다.

```text
size=60px weight=400 family=bold
```

글자 크기는 60px이 됐지만 굵기는 **400(보통)**이고, 글꼴 이름이 **`bold`**가 됐다. `font` 단축 속성의 문법은 `[style] [weight] size family` 순서다. 굵기는 크기 **앞에** 와야 하고, 크기 뒤의 값은 글꼴 이름으로 읽힌다. 그래서 `bold`가 "bold라는 이름의 글꼴"로 해석됐다. 의도대로 하려면 `font: bold 60px sans-serif`처럼 쓰거나, `font-size`와 `font-weight`를 따로 쓴다.

### 6. 상태 선택자

| 선택자 | 수업 파일의 대상 | 결과 |
|--------|------------------|------|
| `input[type=checkbox]:checked` | 체크한 체크박스 | 100×100으로 커짐 |
| `#userId:focus` | 포커스된 입력 칸 | 분홍 배경 |
| `option:enabled` / `:disabled` | 붕어빵·모닝빵(disabled)과 나머지 | 계산값: 빨강 / 진한 빨강 |

`option`의 배경은 계산값으로는 잘 들어갔다(`붕어빵(dis)=rgb(255, 0, 0)`, `피자빵=rgb(139, 0, 0)`). 하지만 드롭다운 목록은 운영체제가 그리는 부분이라, 브라우저와 OS에 따라 화면에 반영되지 않을 수 있다.

## 결과 (Result)

| 확인 항목 | 결과 |
|-----------|------|
| 속성 선택자 6개 적용 후 div 4개 | 모두 `seagreen` (마지막 `[class*=div]`가 전부 덮음) |
| 6개 규칙의 명시도 | 모두 (0, 1, 1) → 순서로 결정 |
| `border: 1px` | `0px none` (테두리 없음) |
| `#test1 > ul > li` | 일치 0개 (`ul`이 `#test1`의 직계 자식이 아님) |
| `font: 60px bold` | 크기 60px, 굵기 400, 글꼴 이름 `bold` |
| 환경 | Chrome 155 헤드리스, `getComputedStyle`로 측정 |

배운 점은 다음과 같다.

- 선택자가 맞는지와 그 스타일이 최종 적용되는지는 다르다. 명시도가 같으면 **나중 규칙**이 이긴다.
- 단축 속성(`border`, `font`)은 빠진 값을 초기값으로 채운다. 그래서 일부만 쓰면 의도와 다르게 동작할 수 있다.

## 더 학습하면 좋은 개념

- **명시도(Specificity) 계산** — (id, class·속성·가상 클래스, 태그)의 세 자리 점수. 다음 우선순위 수업의 핵심이다.
- **CSS 단축 속성과 초기값** — `border`, `font`, `background`가 빠진 값을 어떻게 채우는지.
- **`:has()` 선택자** — "특정 자식을 가진 부모"를 고르는 새 선택자. 자손 선택자의 방향을 뒤집는다.
- **개발자 도구의 Styles 패널** — 취소선이 그어진 규칙을 보면 어느 규칙이 이겼는지 바로 알 수 있다.

## 참고 자료

- [Selectors Level 4 - Attribute selectors](https://www.w3.org/TR/selectors-4/#attribute-selectors)
- [Selectors Level 4 - Child combinator](https://www.w3.org/TR/selectors-4/#child-combinators)
- [Selectors Level 4 - Calculating specificity](https://www.w3.org/TR/selectors-4/#specificity-rules)
- [MDN - font (shorthand)](https://developer.mozilla.org/en-US/docs/Web/CSS/font)
- [MDN - border (shorthand)](https://developer.mozilla.org/en-US/docs/Web/CSS/border)

---

**요약**
1. 속성 선택자 6개는 명시도가 모두 (0, 1, 1)이라 순서로 결정됐고, 네 div 모두에 맞는 마지막 `[class*=div]`가 전부 `seagreen`으로 덮었다.
2. `#test1 > ul > li`는 중간에 `div`가 있어 하나도 맞지 않았다. `>`는 바로 아래 한 단계만 고른다.
3. `border: 1px`은 스타일이 없어 0px로 계산됐고, `font: 60px bold`는 `bold`를 글꼴 이름으로 읽었다. 단축 속성은 빠진 값을 초기값으로 채운다.
