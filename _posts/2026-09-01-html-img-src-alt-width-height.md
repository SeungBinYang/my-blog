---
layout: post
title: "height=\"100rem\"이 100px이 된 이유 — img의 src·alt와 width·height 속성 파싱"
date: 2026-09-01 10:40:00 +0900
categories: [Frontend]
tags: [html, img, alt, responsive, accessibility]
---

## 들어가며 (Situation)

`05_이미지관련태그.html`에서는 `img` 태그와 크기 속성을 배웠다. 수업에서는 크기를 두 가지로 나눴다.

> 고정 크기: 화면의 크기가 변경되어도 변하지 않는다.
> 가변 크기: 화면의 크기에 따라 사이즈가 변경된다.

```html
<img src="sample/image/city1.PNG" alt="도시">

<h4>고정 크기</h4>
<img src="sample/image/flower1.PNG" width="200px" height="100px">
...
<h4>가변 크기</h4>
<img src="sample/image/flower1.PNG" width="20%" height="100rem">
<img src="sample/image/flower2.PNG" width="20%" height="100px">
```

## 문제 상황 (Task)

다시 보니 가변 크기의 첫 번째 꽃 사진만 `height="100rem"`이다. `rem`은 CSS 단위다. 1rem이 16px이라면 100rem은 1600px이어야 한다. 하지만 화면의 세 사진은 높이가 같아 보였다.

1. `height="100rem"`은 실제로 몇 px로 그려질까?
2. `width="200px"`처럼 HTML 속성에 단위를 붙이는 것은 맞는 문법일까?
3. `alt`가 없는 사진은 무엇이 달라질까?

## 해결 과정 (Action)

### 1. 렌더링 크기 측정

Chrome 헤드리스로 페이지를 열고 각 이미지의 속성값, 실제 그려진 크기, 원본 크기를 찍었다. (창 폭 754px)

```text
city1.PNG   attr w=null h=null   -> render 1104x632 natural 1104x632 alt="도시"
flower1.PNG attr w=200px h=100px -> render 200x100  natural 741x492  alt=null
flower1.PNG attr w=20% h=100rem  -> render 145x100  natural 741x492  alt=null
flower2.PNG attr w=20% h=100px   -> render 145x100  natural 598x429  alt=null
```

`height="100rem"`인 사진의 높이는 1600px이 아니라 **100px**였다. 옆 사진의 `100px`과 똑같다.

### 2. HTML 속성은 CSS가 아니다

이유는 HTML 명세의 **크기 값 파싱 규칙**(rules for parsing dimension values)에 있다. `img`의 `width`, `height` 속성은 원래 **음이 아닌 정수**(px 단위로 해석) 하나만 받는다. 브라우저는 앞쪽 숫자를 읽고, 숫자가 아닌 글자가 나오면 거기서 읽기를 멈춘다.

| 속성값 | 브라우저가 읽은 값 | 결과 |
|--------|-------------------|------|
| `200px` | `200` (뒤의 `px` 무시) | 200px |
| `100rem` | `100` (뒤의 `rem` 무시) | **100px** |
| `20%` | `20%` (퍼센트는 예외로 허용) | 부모 폭의 20% |

`200px`이 잘 동작했던 이유도 같다. `px`가 의미가 있어서가 아니라 무시됐기 때문이다. 표준 문법은 단위 없이 `width="200"`이다. `rem`, `em`, `vw` 같은 단위를 쓰고 싶다면 HTML 속성이 아니라 CSS로 써야 한다.

```css
img.flower { height: 6.25rem; }   /* CSS에서는 단위가 그대로 동작 (예시 코드) */
```

### 3. 고정 크기와 가변 크기 확인

창 폭을 바꿔 가며 같은 페이지를 열었다.

| 창 폭 | 고정 (`width="200px"`) | 가변 (`width="20%"`) | city1 (속성 없음) | 가로 스크롤 |
|-------|------------------------|----------------------|-------------------|-------------|
| 574px | 200px | 109px | 1104px | **생김** |
| 754px | 200px | 145px | 1104px | 생김 |
| 1254px | 200px | 245px | 1104px | 없음 |

`20%`는 화면에 따라 변했고 `200px`은 그대로였다. 수업 설명이 맞다. 그런데 크기를 주지 않은 `city1.PNG`는 원본 크기 1104px로 그려져서, 창이 좁으면 **가로 스크롤**이 생겼다. 원본 크기가 화면보다 큰 사진에는 보통 CSS로 상한을 둔다. (**예시 코드**)

```css
img { max-width: 100%; height: auto; }
```

또 하나, 가변 크기 사진은 폭만 퍼센트이고 높이는 100px로 고정이라 **비율이 깨진다.** 원본 741×492 사진이 145×100으로 그려졌다(원본 비율이면 145×96). 폭을 퍼센트로 줬다면 높이는 `auto`로 두는 것이 안전하다.

그렇다고 `width`, `height` 속성을 아예 빼라는 뜻은 아니다. 두 속성을 원본 크기로 적어 두면, 브라우저는 이미지를 받기 전에 **비율만큼 자리를 미리 잡는다.** 그래서 사진이 늦게 로드되어도 아래 글자가 밀려 내려가지 않는다(레이아웃 이동, CLS 방지).

### 4. `alt`는 선택이 아니다

측정 결과를 보면 꽃 사진 6장은 모두 `alt=null`, 즉 `alt` 속성이 없다. 수업 메모에는 `alt`를 "사진을 읽을 수 없을 때 대체하는 텍스트"라고 적었다. 그 역할도 맞지만 더 중요한 역할이 있다.

| 상황 | `alt`의 역할 |
|------|--------------|
| 화면 낭독기 사용자 | 이미지 대신 `alt`를 읽어 준다. 없으면 파일 이름(`flower1.PNG`)을 읽기도 한다. |
| 이미지 로드 실패 | 깨진 이미지 아이콘 옆에 `alt` 글자가 보인다. |
| 검색 엔진 | 이미지 내용을 이해하는 단서가 된다. |

HTML 명세는 `img`의 `alt`를 원칙적으로 필수로 본다. 다만 장식용 이미지는 `alt=""`(빈 값)로 "읽지 않아도 된다"고 알린다. **속성이 없는 것과 빈 값은 다르다.**

### 5. 파일 이름 대소문자

이 파일들의 확장자는 `.PNG`(대문자)다. 내 Windows 컴퓨터에서는 `flower1.png`라고 써도 열리지만, 이 블로그가 올라가는 GitHub Pages 같은 리눅스 서버는 대소문자를 구분한다. 로컬에서는 보이던 이미지가 배포 후 사라지는 흔한 원인이다. 경로는 실제 파일 이름과 대소문자까지 맞춘다.

## 결과 (Result)

| 확인 항목 | 결과 |
|-----------|------|
| `height="100rem"` | 1600px이 아니라 **100px** (숫자 뒤 `rem` 무시) |
| `width="200px"` | 200px (뒤의 `px`가 무시된 결과) |
| `width="20%"` | 창 574 / 754 / 1254px에서 109 / 145 / 245px |
| 크기 없는 1104px 사진 | 창이 좁으면 가로 스크롤 발생 |
| `alt` 없는 이미지 | 7장 중 6장 |

배운 점은 다음과 같다.

- HTML 속성의 `width`, `height`는 **px 숫자만** 받는다. 단위는 CSS에서 쓴다.
- `alt`가 없는 것과 `alt=""`는 다르다. 의미 있는 사진에는 설명을, 장식 사진에는 빈 값을 준다.

## 더 학습하면 좋은 개념

- **반응형 이미지 (`srcset`, `sizes`, `picture`)** — 화면 크기마다 다른 해상도의 파일을 내려 주는 방법.
- **CLS(Cumulative Layout Shift)** — `width`, `height` 속성으로 자리를 미리 잡아야 하는 이유가 되는 성능 지표.
- **`object-fit`** — 비율이 다른 칸에 이미지를 넣을 때 자르거나 맞추는 CSS 속성.
- **CSS 단위 (`px`, `em`, `rem`, `%`, `vw`)** — 속성 대신 CSS로 크기를 줄 때 고를 단위들.

## 참고 자료

- [HTML Living Standard - The img element](https://html.spec.whatwg.org/multipage/embedded-content.html#the-img-element)
- [HTML Living Standard - Dimension attributes](https://html.spec.whatwg.org/multipage/embedded-content-other.html#dimension-attributes)
- [HTML Living Standard - Rules for parsing dimension values](https://html.spec.whatwg.org/multipage/common-microsyntaxes.html#rules-for-parsing-dimension-values)
- [HTML Living Standard - Requirements for providing text to act as an alternative for images](https://html.spec.whatwg.org/multipage/images.html#alt)
- [MDN - img](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/img)
- [web.dev - Cumulative Layout Shift (CLS)](https://web.dev/articles/cls)

---

**요약**
1. `height="100rem"`은 1600px이 아니라 100px로 그려졌다. HTML 크기 속성은 앞쪽 숫자만 읽고 `rem`·`px` 같은 단위는 버린다.
2. `width="20%"`는 창 폭에 따라 109~245px로 변했지만, 크기를 주지 않은 1104px 사진은 좁은 창에서 가로 스크롤을 만들었다.
3. 꽃 사진 6장은 `alt`가 없었다. 의미 있는 사진에는 설명을, 장식 사진에는 `alt=""`를 준다.
