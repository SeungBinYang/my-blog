---
layout: post
title: "controls=\"false\"를 써도 컨트롤이 보이는 이유 — audio·video와 불리언 속성"
date: 2026-09-01 10:50:00 +0900
categories: [Frontend]
tags: [html, audio, video, boolean-attribute, autoplay]
---

## 들어가며 (Situation)

`06_미디어관련태그.html`은 14줄짜리 짧은 파일이다. 음악 파일과 동영상 파일을 페이지에 넣는다.

```html
<h3>오디오 관련 태그</h3>
<audio src="sample/audio/major.mp3" controls="controls" loop="loop">뮤직 큐~</audio>
<h3>비디오 관련 태그</h3>
<video src="sample/video/video1.mp4" controls="controls"></video>
```

## 문제 상황 (Task)

짧은 코드인데도 궁금한 점이 세 가지 있었다.

1. `controls="controls"`, `loop="loop"`처럼 속성 이름과 값이 같다. 값을 다르게 쓰면 어떻게 될까? `controls="false"`라고 쓰면 꺼질까?
2. `audio` 태그 사이에 쓴 "뮤직 큐~"는 언제 보일까?
3. 페이지를 열자마자 음악이 나오게 할 수 있을까?

## 해결 과정 (Action)

### 1. 브라우저가 읽은 미디어 정보

Chrome 헤드리스로 페이지를 열고 두 요소의 속성과 미디어 정보를 찍었다.

```text
audio controls=true loop=true autoplay=false duration=273.893875 readyState=4 childText="뮤직 큐~"
video controls=true 540x360 duration=272.950567 render=540x360 paused=true
```

| 항목 | `audio` | `video` |
|------|---------|---------|
| 파일 크기 | 4.4MB (`major.mp3`) | 15.5MB (`video1.mp4`) |
| 길이 | 약 273.9초 (4분 34초) | 약 273.0초 |
| 화면 크기 | (컨트롤 막대만) | 원본 해상도 540×360 그대로 |
| 재생 중? | - | `paused=true` |

`video`에 크기를 주지 않으면 원본 해상도(540×360)로 그려진다. [이미지]({% post_url 2026-09-01-html-img-src-alt-width-height %})와 같은 동작이다.

### 2. 불리언 속성은 "있느냐 없느냐"만 본다

`controls`, `loop`, `autoplay`, `muted`는 **불리언 속성**(boolean attribute)이다. HTML 명세의 정의는 이렇다.

> The presence of a boolean attribute on an element represents the true value, and the absence of the attribute represents the false value.

속성이 **있으면 참, 없으면 거짓**이다. 값은 보지 않는다. 확인해 봤다.

```javascript
const x = document.createElement("video");
x.setAttribute("controls", "false");
x.controls;               // true
x.removeAttribute("controls");
x.controls;               // false
```

`controls="false"`라고 써도 **컨트롤은 켜진다.** 끄는 방법은 속성을 지우는 것뿐이다. 명세가 허용하는 표기는 세 가지다.

| 표기 | 의미 |
|------|------|
| `<video controls>` | 켜짐 (가장 흔한 표기) |
| `<video controls="">` | 켜짐 |
| `<video controls="controls">` | 켜짐 (수업 표기, XHTML 시절 방식) |
| `<video controls="false">` | **켜짐** (명세 위반이지만 브라우저는 켜짐으로 처리) |
| `<video>` | 꺼짐 |

수업의 `controls="controls"`는 틀린 문법이 아니다. 예전 XHTML은 모든 속성에 값이 필요했기 때문에 이름을 값으로 반복해 썼고, 그 흔적이다. 지금은 `controls`만 써도 된다. [폼 태그]({% post_url 2026-09-02-html-form-input-types-validation %})의 `required`, `checked`, `disabled`, `multiple`도 모두 같은 규칙을 따른다.

### 3. 태그 사이의 글자는 대체 콘텐츠

`audio`의 자식 텍스트 "뮤직 큐~"는 DOM에 들어 있지만(`childText="뮤직 큐~"`) 화면에는 보이지 않았다. `audio`, `video` 태그 안의 내용은 **이 요소를 지원하지 않는 아주 오래된 브라우저**에만 보이는 대체 콘텐츠다. 파일 형식을 재생할 수 없을 때 대신 보이는 것이 아니다. 그 경우를 대비하려면 `source`를 여러 개 두어 형식을 고르게 한다. (**예시 코드**)

```html
<video controls width="540">
    <source src="video1.webm" type="video/webm">
    <source src="video1.mp4" type="video/mp4">
    <a href="video1.mp4">동영상 내려받기</a>
</video>
```

브라우저는 `source`를 위에서부터 보고 재생할 수 있는 첫 번째 파일을 고른다.

### 4. 자동 재생은 브라우저가 막는다

음악이 바로 나오도록 `autoplay`를 붙이면 될 것 같지만, 요즘 브라우저는 **소리가 나는 자동 재생**을 기본으로 막는다. Chrome의 자동 재생 정책 요약은 이렇다.

| 조건 | 자동 재생 |
|------|-----------|
| `muted`와 함께 `autoplay` | 항상 허용 |
| 소리 있는 `autoplay` | 사용자가 그 사이트를 클릭·탭한 적이 있거나, 자주 방문해 미디어 참여도가 높을 때만 허용 |

그래서 배경 동영상은 보통 `autoplay muted loop playsinline`을 함께 쓴다. 소리가 꼭 필요한 음악은 사용자가 재생 버튼을 누르게 하는 것이 맞다. 페이지를 열자마자 소리가 나는 것은 사용자 경험에도 좋지 않다.

### 5. 파일 크기와 `preload`

`readyState=4`(HAVE_ENOUGH_DATA)는 브라우저가 재생에 충분한 데이터를 이미 받았다는 뜻이다. 재생 버튼을 누르지 않아도 미디어를 받기 시작했다. 로컬 파일이라 문제가 없었지만, 실제 서버라면 방문자마다 15MB짜리 동영상을 받을 수도 있다. `preload` 속성으로 미리 받는 양을 정할 수 있다. 다만 브라우저는 이 값을 힌트로만 다룬다.

| `preload` 값 | 의미 |
|--------------|------|
| `none` | 재생 전까지 받지 않음 |
| `metadata` | 길이, 크기 같은 정보만 |
| `auto` | 브라우저 판단에 맡김 (전체를 받을 수도 있음) |

## 결과 (Result)

| 확인 항목 | 결과 |
|-----------|------|
| `setAttribute("controls", "false")` | `controls === true` (컨트롤 켜짐) |
| `removeAttribute("controls")` | `controls === false` |
| `audio` 안의 "뮤직 큐~" | DOM에는 있지만 화면에는 안 보임 |
| 크기 없는 `video` | 원본 540×360으로 렌더링 |
| 재생 전 `readyState` | 4 (이미 데이터를 받음) |

배운 점은 다음과 같다.

- 불리언 속성은 **존재 자체가 참**이다. 끄려면 값을 바꾸지 말고 속성을 지운다.
- 소리 있는 자동 재생은 브라우저 정책 때문에 기본으로 막힌다고 생각하고 설계한다.

## 더 학습하면 좋은 개념

- **`source`와 `type`, 코덱** — 브라우저마다 지원하는 형식이 달라 여러 파일을 준비하는 이유.
- **`track` 요소와 자막(WebVTT)** — 동영상 접근성을 위한 자막 파일을 붙이는 방법.
- **HTMLMediaElement API (`play()`, `pause()`, `currentTime`)** — JavaScript로 직접 재생을 제어하는 방법.
- **스트리밍 (HLS, DASH)** — 긴 동영상을 통째로 받지 않고 조각으로 내려받는 방식.

## 참고 자료

- [HTML Living Standard - Boolean attributes](https://html.spec.whatwg.org/multipage/common-microsyntaxes.html#boolean-attributes)
- [HTML Living Standard - The video element](https://html.spec.whatwg.org/multipage/media.html#the-video-element)
- [HTML Living Standard - The audio element](https://html.spec.whatwg.org/multipage/media.html#the-audio-element)
- [MDN - video: preload](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/video#preload)
- [Chrome for Developers - Autoplay policy in Chrome](https://developer.chrome.com/blog/autoplay)

---

**요약**
1. `controls`, `loop` 같은 불리언 속성은 값과 상관없이 있으면 참이라서, `controls="false"`도 컨트롤을 켰다. 끄려면 속성을 지워야 한다.
2. `audio` 안의 "뮤직 큐~"는 태그를 지원하지 않는 브라우저용 대체 콘텐츠라 DOM에만 있고 화면에는 보이지 않았다.
3. 소리 있는 `autoplay`는 Chrome 정책상 기본으로 막힌다. 재생 전에도 `readyState=4`일 만큼 미디어를 미리 받으므로 `preload`도 고려한다.
