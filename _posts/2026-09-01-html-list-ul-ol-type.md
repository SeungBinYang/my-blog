---
layout: post
title: "ol의 type=\"a\"는 CSS 속성으로 바뀐다 — 목록 태그 ul·ol과 list-style-type"
date: 2026-09-01 10:10:00 +0900
categories: [Frontend]
tags: [html, css, list, ul, ol]
---

## 들어가며 (Situation)

HTML 두 번째 실습 파일은 `02_목록관련태그.html`이다. 순서 없는 목록 `ul` 하나와, `type`만 다른 순서 있는 목록 `ol` 세 개가 들어 있다.

```html
<ul>
    <li>HTML</li>
    <li>CSS</li>
    <li>JS</li>
</ul>
<!-- ul>li*3 -->

<ol type="a"> ... </ol>
<ol type="A"> ... </ol>
<ol type="I"> ... </ol>
```

주석의 `ul>li*3`은 Emmet 약어다. 에디터에서 이것을 입력하고 Tab을 누르면 위의 `ul` 구조가 만들어진다.

## 문제 상황 (Task)

`type="a"`를 쓰면 번호가 `a. b. c.`로 바뀌었다. 그런데 CSS를 배우면서 "모양은 CSS로 바꾼다"고 들었기 때문에 질문이 생겼다.

1. HTML 속성 `type`은 CSS와 어떤 관계인가? 둘이 따로 노는가?
2. `type` 말고 `ol`에 쓸 수 있는 속성은 무엇이 있나?
3. `ul`에도 `type`을 쓸 수 있나?

## 해결 과정 (Action)

### 1. `type` 속성은 CSS 값으로 바뀐다

Chrome 헤드리스로 파일을 열어 각 목록의 계산된 `list-style-type`을 출력했다.

```text
UL type=null -> disc      paddingLeft=40px
OL type=a    -> lower-alpha paddingLeft=40px
OL type=A    -> upper-alpha paddingLeft=40px
OL type=I    -> upper-roman paddingLeft=40px
```

HTML 속성 `type`이 CSS 속성 `list-style-type`으로 그대로 바뀌어 있다. HTML 명세의 Rendering 장에도 `ol[type="a"]`에 `list-style-type: lower-alpha`를 주는 기본 규칙이 표로 정리돼 있다. 결국 **보이는 모양은 처음부터 CSS가 정하고 있었고**, `type` 속성은 그 값을 미리 넣어 두는 지름길이었다.

| `type` 값 | 계산된 `list-style-type` | 번호 모양 |
|-----------|-------------------------|-----------|
| (없음) | `decimal` | 1. 2. 3. |
| `a` | `lower-alpha` | a. b. c. |
| `A` | `upper-alpha` | A. B. C. |
| `i` | `lower-roman` | i. ii. iii. |
| `I` | `upper-roman` | I. II. III. |

목록 왼쪽의 들여쓰기도 기본값이 `padding-left: 40px`였다. 번호(마커)가 이 여백 안에 그려진다.

### 2. `ol`의 `type`은 지금도 표준 속성이다

그렇다면 `type`은 낡은 문법일까? HTML 명세를 확인하니 `ol`에 쓸 수 있는 속성은 세 가지다.

| 속성 | 뜻 | 예시 |
|------|-----|------|
| `reversed` | 번호를 거꾸로 센다 | 5, 4, 3, … |
| `start` | 시작 번호 | `start="3"` → 3, 4, 5 |
| `type` | 번호 종류 | `a`, `A`, `i`, `I`, `1` |

`type`이 남아 있는 이유는 명세에 적혀 있다.

> The type attribute can be used to specify the kind of marker to use in the list, in the cases where that matters (e.g., because items are to be referenced by their number/letter).

"3번 항목 참고"처럼 **번호 자체가 내용의 일부일 때** 쓰라는 것이다. 법 조항(가, 나, 다), 시험 보기(A, B, C)가 그런 예다. 단순히 보기 좋게 바꾸는 것이라면 CSS의 `list-style-type`이 맞다.

### 3. `ul`의 `type`은 폐지됐다

`ul`에 `type="square"`를 쓰는 예제도 인터넷에 많다. 하지만 HTML 명세의 Obsolete features 목록에는 `ul` 요소의 `type` 속성이 들어 있다. 브라우저가 아직 처리해 주더라도 표준 문법은 아니다. `ul`의 마커는 CSS로 바꾼다. (**예시 코드**)

```css
ul { list-style-type: square; }
nav ul { list-style: none; padding-left: 0; }  /* 메뉴처럼 마커가 필요 없을 때 */
```

두 번째 줄은 나중에 CSS 실습에서 자주 쓰게 될 패턴이다. 메뉴는 의미상 "항목 목록"이므로 `ul`로 만든다. 그리고 마커와 들여쓰기만 CSS로 지운다.

### 4. `ul`과 `ol`을 고르는 기준

| | `ul` | `ol` |
|---|------|------|
| 의미 | 순서가 바뀌어도 뜻이 같은 목록 | 순서가 의미를 가지는 목록 |
| 예 | 배운 언어, 메뉴, 태그 목록 | 설치 순서, 순위, 레시피 단계 |
| 기본 마커 | `disc` (●) | `decimal` (1.) |

수업 파일의 `HTML, CSS, JS`는 배우는 순서로 보면 `ol`, 단순 나열로 보면 `ul`이다. 둘 중 무엇을 쓸지는 **모양이 아니라 순서가 의미가 있는지**로 정한다. [글자 태그 실습]({% post_url 2026-09-01-html-text-tags-strong-b-pre %})과 같은 원칙이다.

## 결과 (Result)

| 확인 항목 | 결과 |
|-----------|------|
| `ol type="a"/"A"/"I"` | `list-style-type`이 각각 `lower-alpha` / `upper-alpha` / `upper-roman` |
| 목록 기본 들여쓰기 | `padding-left: 40px` |
| `ol`의 `type` | 표준 속성 (번호가 내용의 일부일 때 사용) |
| `ul`의 `type` | 폐지된(obsolete) 속성, CSS로 대체 |

배운 점은 다음과 같다.

- 보이는 모양은 결국 CSS가 정한다. HTML 속성은 그 기본값을 넣어 주는 통로일 뿐이다.
- `ul`이냐 `ol`이냐는 순서가 의미를 가지는지로 고른다.

## 더 학습하면 좋은 개념

- **`::marker` 가상 요소** — 번호나 점의 색, 크기만 따로 바꿀 수 있다.
- **CSS 카운터(`counter-reset`, `counter-increment`)** — `1-1, 1-2`처럼 번호 체계를 직접 만들 수 있다.
- **설명 목록 `dl`, `dt`, `dd`** — 용어와 설명을 짝지어 나열하는 세 번째 목록 태그.
- **Emmet** — `ul>li*3` 같은 약어로 HTML을 빠르게 만드는 문법. 이후 실습 파일 곳곳에 주석으로 남아 있다.

## 참고 자료

- [HTML Living Standard - The ol element](https://html.spec.whatwg.org/multipage/grouping-content.html#the-ol-element)
- [HTML Living Standard - Rendering: Lists](https://html.spec.whatwg.org/multipage/rendering.html#lists)
- [HTML Living Standard - Obsolete features](https://html.spec.whatwg.org/multipage/obsolete.html#non-conforming-features)
- [MDN - list-style-type](https://developer.mozilla.org/en-US/docs/Web/CSS/list-style-type)
- [Emmet Documentation - Abbreviations Syntax](https://docs.emmet.io/abbreviations/syntax/)

---

**요약**
1. `ol`의 `type="a"`, `"A"`, `"I"`는 계산된 스타일에서 `list-style-type: lower-alpha`, `upper-alpha`, `upper-roman`으로 바뀌어 있었다.
2. `ol`의 `type`은 번호가 내용의 일부일 때 쓰는 표준 속성이지만, `ul`의 `type`은 폐지되어 CSS로 바꿔야 한다.
3. `ul`과 `ol` 중 무엇을 쓸지는 모양이 아니라 순서가 의미를 가지는지로 정한다.
