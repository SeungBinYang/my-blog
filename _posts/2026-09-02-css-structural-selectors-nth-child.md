---
layout: post
title: "first-child에 준 aqua가 사라졌다 — nth-child 규칙끼리의 순서 싸움과 '#test :first-child'의 공백"
date: 2026-09-02 16:00:00 +0900
categories: [Frontend]
tags: [css, selectors, nth-child, pseudo-class]
---

## 들어가며 (Situation)

`03_css선택자3.html`은 **위치**로 요소를 고르는 선택자를 다룬다. 실습 대상은 `p` 다섯 개와 `pre` 하나다.

```html
<div id="test">
    <p>테스트1</p>
    <p>테스트2</p>
    <p>테스트3</p>
    <p>테스트4</p>
    <p>테스트5</p>
    <pre>테스트6</pre>
</div>
```

```css
#test :first-child       { background-color: aqua; }
#test :last-child        { background-color: thistle; }
#test :nth-child(2)      { background-color: cadetblue; }
#test :nth-child(2n-1)   { background-color: rgb(73, 148, 246); }
#test :nth-last-child(4) { background-color: coral; }
```

그 밖에 형제 선택자(`+`, `~`), 부정 선택자(`:not`), 문자 선택자(`::selection`)도 있었다.

## 문제 상황 (Task)

첫 번째 줄 "테스트1"은 `:first-child`로 `aqua`가 되어야 했다. 그런데 화면에서는 파란색이었다. 규칙 다섯 개가 서로 겹치는데, 어떤 규칙이 이겼는지 한눈에 알 수 없었다.

1. 여섯 줄 각각에 최종적으로 어떤 색이 칠해졌고, 그 이유는 무엇일까?
2. `#test :first-child`처럼 `#test`와 `:first-child` 사이에 **공백**이 있는데, 이것은 `#test:first-child`와 다를까?

## 해결 과정 (Action)

### 1. 규칙마다 맞는 줄 표시하기

`nth-child`는 부모 안의 **모든 형제 요소** 중 몇 번째인지 센다(1부터). `p`와 `pre`를 구분하지 않는다.

| 줄 | 순서 | `:first-child` | `:last-child` | `:nth-child(2)` | `:nth-child(2n-1)` | `:nth-last-child(4)` |
|----|------|----|----|----|----|----|
| 테스트1 | 1 | O | | | O (n=1) | |
| 테스트2 | 2 | | | O | | |
| 테스트3 | 3 | | | | O (n=2) | O (뒤에서 4번째) |
| 테스트4 | 4 | | | | | |
| 테스트5 | 5 | | | | O (n=3) | |
| 테스트6 (`pre`) | 6 | | O | | | |

`2n-1`에 n=1, 2, 3…을 넣으면 1, 3, 5다. 홀수 번째 줄이다. `nth-last-child(4)`는 끝에서부터 센다. 6개 중 뒤에서 4번째는 앞에서 3번째다.

### 2. 다섯 규칙은 명시도가 같다

다섯 선택자는 모두 "id 1개 + 가상 클래스 1개"다. 명시도가 **(1, 1, 0)으로 같으므로 나중에 쓴 규칙이 이긴다.** 규칙을 쓴 순서대로 위 표를 덮어 보면 최종 색을 예측할 수 있다.

| 줄 | 맞는 규칙 (쓴 순서) | 마지막 규칙 | 예상 색 |
|----|---------------------|-------------|---------|
| 테스트1 | first-child → **2n-1** | 2n-1 | 파랑 `rgb(73,148,246)` |
| 테스트2 | **nth-child(2)** | nth-child(2) | `cadetblue` |
| 테스트3 | 2n-1 → **nth-last-child(4)** | nth-last-child(4) | `coral` |
| 테스트4 | 없음 | - | 투명 |
| 테스트5 | **2n-1** | 2n-1 | 파랑 |
| 테스트6 | **last-child** | last-child | `thistle` |

Chrome 헤드리스로 계산된 배경색을 찍었다.

```text
P:테스트1   bg=rgb(73, 148, 246)
P:테스트2   bg=rgb(95, 158, 160)    (cadetblue)
P:테스트3   bg=rgb(255, 127, 80)    (coral)
P:테스트4   bg=rgba(0, 0, 0, 0)
P:테스트5   bg=rgb(73, 148, 246)
PRE:테스트6 bg=rgb(216, 191, 216)   (thistle)
```

예상과 정확히 같았다. 테스트1의 `aqua`는 **아래쪽 `2n-1` 규칙에 덮였다.** `:first-child` 선택자가 틀린 것이 아니다. 선택은 됐지만 마지막에 이기지 못했다. 테스트3도 `2n-1`로 파랑이 될 뻔했지만, 더 아래 `nth-last-child(4)`가 다시 덮었다.

### 3. 공백 하나가 뜻을 바꾼다

`#test :first-child`의 공백은 [후손 결합자]({% post_url 2026-09-02-css-attribute-child-descendant-selectors %})다. "`#test` 안에 있는 요소 중, 자기 부모의 첫 번째 자식인 것"이라는 뜻이다. 공백 없는 `#test:first-child`는 "`#test` 자신이 첫 번째 자식일 때"를 뜻한다. 완전히 다르다.

수업 파일에서는 `p` 안에 다른 태그가 없어서 차이가 드러나지 않았다. `p` 안에 `b`와 `i`를 넣고 일치하는 요소를 셌다. (확인용 **예시 코드**)

```html
<div id="test"><p><b>굵게</b> 테스트1</p><p>테스트2 <i>기울임</i></p></div>
```

```text
#test :first-child matches: P(굵게 테스트1), B(굵게), I(기울임)
```

첫 번째 `p`뿐 아니라 **그 안의 `b`, 두 번째 `p` 안의 `i`까지** 골랐다. `b`와 `i`도 각자의 부모 안에서는 첫 번째 자식이기 때문이다. 직계 자식만 고르려면 `#test > :first-child`, 특정 태그로 좁히려면 `#test > p:first-child`라고 쓴다.

수업에서 함께 본 Tailwind 이전 카드 예제의 `style.css`에도 같은 경고가 주석으로 있다.

> `.card:hover` 는 붙여 쓴다. 공백을 넣으면(`.card :hover`) 카드 "안쪽" 요소를 가리키는 완전히 다른 뜻이 된다

| 선택자 | 뜻 |
|--------|----|
| `#test:first-child` | `#test` 자신이 첫 자식일 때 `#test` |
| `#test :first-child` | `#test`의 모든 후손 중 첫 자식인 것들 |
| `#test > :first-child` | `#test`의 직계 자식 중 첫 번째 |

### 4. 형제, 부정, 문자 선택자

| 선택자 | 수업 결과 |
|--------|-----------|
| `#div-test1 + div` | 바로 다음 형제 `div-test2`만 초록 (`rgb(0,128,0)`) |
| `#div-test1 ~ div` | 뒤의 형제 전부. 수업에서는 주석 처리됨 |
| `#test2 p:not(:nth-child(even))` | 테스트1, 3, 5만 초록. `pre`(6번째)는 `p`가 아니라 제외 |
| `#test3 p::selection` | 드래그로 선택한 글자만 빨강 |

`:not(:nth-child(even))`은 "짝수 번째가 아닌 것"이므로 `:nth-child(odd)`와 같다. `::selection`처럼 콜론이 두 개인 것은 **가상 요소**다. 요소 전체가 아니라 요소의 일부(선택된 글자, 첫 글자, 첫 줄)를 가리킨다. 콜론 하나인 가상 클래스(`:hover`, `:first-child`)는 요소의 **상태나 위치**를 가리킨다.

## 결과 (Result)

| 줄 | 최종 색 | 이긴 규칙 |
|----|---------|-----------|
| 테스트1 | 파랑 (aqua 아님) | `:nth-child(2n-1)` |
| 테스트2 | cadetblue | `:nth-child(2)` |
| 테스트3 | coral | `:nth-last-child(4)` |
| 테스트4 | 투명 | 없음 |
| 테스트5 | 파랑 | `:nth-child(2n-1)` |
| 테스트6 | thistle | `:last-child` |

배운 점은 다음과 같다.

- 명시도가 같은 위치 선택자끼리는 **쓴 순서**로 결정된다. 예상 색을 표로 그려 보면 왜 그 색인지 설명할 수 있다.
- 선택자 사이의 공백은 "그 안에 있는"이라는 결합자다. `A :first-child`는 생각보다 많은 요소를 고른다.

## 더 학습하면 좋은 개념

- **`:nth-of-type()`** — 같은 태그끼리만 세는 버전. `p`와 `pre`가 섞인 이번 예제에서 `nth-child`와 결과가 달라진다.
- **`:nth-child(An+B of S)` 문법** — 특정 선택자에 맞는 형제 중 몇 번째인지 세는 Selectors Level 4 문법.
- **가상 요소 `::before`, `::after`** — 콘텐츠를 CSS로 추가하는 가상 요소. `::selection`과 같은 계열이다.
- **명시도 계산 규칙** — `:not()`의 명시도는 괄호 안 선택자로 계산된다.

## 참고 자료

- [Selectors Level 4 - Child-indexed pseudo-classes (:nth-child)](https://www.w3.org/TR/selectors-4/#child-index)
- [Selectors Level 4 - Descendant combinator](https://www.w3.org/TR/selectors-4/#descendant-combinators)
- [Selectors Level 4 - The negation pseudo-class :not()](https://www.w3.org/TR/selectors-4/#negation)
- [MDN - :nth-child()](https://developer.mozilla.org/en-US/docs/Web/CSS/:nth-child)
- [MDN - ::selection](https://developer.mozilla.org/en-US/docs/Web/CSS/::selection)

---

**요약**
1. 위치 선택자 다섯 개는 명시도가 모두 (1, 1, 0)이라 쓴 순서로 결정됐다. 그래서 테스트1의 `aqua`는 아래쪽 `:nth-child(2n-1)` 파랑에 덮였다.
2. 예상 색을 표로 그린 결과와 Chrome에서 측정한 여섯 줄의 색이 정확히 일치했다.
3. `#test :first-child`의 공백은 후손 결합자라서, `p` 안에 `b`, `i`를 넣으면 그 태그들까지 함께 골랐다.
