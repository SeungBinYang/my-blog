---
layout: post
title: "JSX에 class를 썼더니 화면은 멀쩡하고 콘솔에만 경고 — createElement와 Babel이 바꾼 JSX의 정체"
date: 2026-11-23 10:35:00 +0900
categories: [Frontend]
tags: [react, jsx, babel, createelement, classname]
mermaid: true
---

## 들어가며 (Situation)

React 첫날은 설치 없이 CDN 스크립트로 시작했다. 첫 실습(`01_createElement.html`)은 JSX 없이 함수로 요소를 만드는 것이었다.

```html
<script crossorigin src="https://unpkg.com/react@18.3.1/umd/react.development.js"></script>
<script crossorigin src="https://unpkg.com/react-dom@18.3.1/umd/react-dom.development.js"></script>
```

```javascript
/*
  1번째 인자 : 만들려는 요소의 타입(ex) h1, div, pre)
  2번째 인자 : 요소의 프로퍼티를 표현
  3번째 인자 : 자식 요소 표현
*/
const hiElement = React.createElement('h1', {className: 'hi'} , 'helloworld!!');
ReactDOM.createRoot(document.getElementById('root')).render(hiElement);
```

요소를 만들 때마다 `createElement`를 부르는 것은 번거롭다. 그래서 다음 실습(`01_jsx-intro.html`)에서 Babel을 추가하고 JSX를 썼다.

```html
<script crossorigin src="https://unpkg.com/@babel/standalone@8.1.3/babel.min.js"></script>
<script type="text/babel">
  function formatName(user) {
    return `${user.lastName} ${user.firstName}`;
  }
  const user = { firstName : '승빈', lastName : '양' };

  // jsx 내에서 중괄호를 활용하여 모든 js 표현식을 사용할 수 있다.
  const element = <h1 class="text">안녕! {formatName(user)}</h1>

  ReactDOM.createRoot(document.getElementById('root')).render(element);
</script>
```

## 문제 상황 (Task)

화면에는 "안녕! 양 승빈"이 잘 나왔다. 그런데 첫 실습에서는 `className`을 썼는데, JSX 실습에서는 HTML처럼 `class`를 썼다. 둘 다 되는 걸까?

그리고 JSX가 "JavaScript를 XML처럼 쓰는 문법"이라면, 브라우저는 `<h1>`이 섞인 JavaScript를 어떻게 실행할까? `createElement`와는 어떤 관계일까?

## 해결 과정 (Action)

### 1. 콘솔을 열어 보니 경고가 있었다

Playwright로 같은 페이지를 열고 콘솔 메시지를 확인했다.

```text
[ERROR] Warning: Invalid DOM property `class`. Did you mean `className`?
    at h1
[WARNING] You are using the in-browser Babel transformer. Be sure to precompile your scripts for production
```

만들어진 HTML은 `<h1 class="text">안녕! 양 승빈</h1>`이었다. React 18은 `class`를 그대로 DOM에 넣어 주면서도, 개발 모드에서 "`className`을 쓰라"고 경고했다. 화면만 봐서는 알 수 없고, 콘솔을 열어야 보이는 문제였다.

JSX의 속성은 HTML 속성이 아니라 **DOM 프로퍼티 이름**을 따른다. DOM에서 클래스는 `element.className`이고, `class`는 JavaScript의 예약어이기도 하다. 그래서 `className`, `htmlFor`(label의 `for`)처럼 이름이 다르다.

| HTML 속성 | JSX 속성 |
|-----------|----------|
| `class` | `className` |
| `for` | `htmlFor` |
| `onclick="..."` (문자열) | `onClick={함수}` (camelCase, 함수 전달) |
| `style="color: red"` (문자열) | `style={객체}` (camelCase 프로퍼티) |

### 2. Babel이 JSX를 무엇으로 바꾸나

페이지에 로드된 Babel로 JSX 몇 줄을 직접 변환해 봤다.

```javascript
Babel.transform('const el = <h1 className="text">안녕! {formatName(user)}</h1>;',
                { presets: [['react', { runtime: 'classic' }]] }).code
```

```javascript
const el = /*#__PURE__*/React.createElement("h1", {
  className: "text"
}, "안녕! ", formatName(user));
```

JSX는 결국 **`React.createElement` 호출**이었다. 첫 실습에서 손으로 쓴 코드와 같은 모양이다. `{formatName(user)}`는 세 번째 이후 인자(자식)로 들어갔다. 중괄호 안에 "표현식"만 쓸 수 있는 이유도 여기서 보인다. 함수의 인자 자리에는 `if`나 `for` 같은 문장이 들어갈 수 없다.

```mermaid
flowchart LR
    J["JSX 코드<br/>(h1 태그, className, 중괄호 표현식)"] -->|"Babel (변환)"| C["React.createElement('h1',<br/>{ className: 'text' }, '안녕! ', 이름)"]
    C -->|"실행"| E["React 엘리먼트<br/>(가벼운 JS 객체)"]
    E -->|"ReactDOM 렌더링"| D["실제 DOM의 h1 요소<br/>(class 속성)"]
```

`<script type="text/babel">`이 필요한 이유도 같다. 브라우저는 `type="text/babel"`인 스크립트를 실행하지 않고, Babel standalone이 그 내용을 찾아 변환한 뒤 실행한다. 콘솔의 두 번째 경고처럼 이 방식은 매번 브라우저에서 변환하므로 연습용이다. 실제 프로젝트는 Vite 같은 빌드 도구가 미리 변환한다.

### 3. 엘리먼트와 컴포넌트

두 번째 실습(`02_component.html`)에서는 엘리먼트를 **반환하는 함수**를 만들었다.

```javascript
function Helloworld() {
  return React.createElement('h1', {className: 'hi'} , 'helloworld!!');
}
ReactDOM.createRoot(document.getElementById('root'))
  .render(React.createElement(Helloworld));
```

| | 엘리먼트 | 컴포넌트 |
|---|---|---|
| 정체 | 화면에 무엇을 그릴지 적은 객체 | 엘리먼트를 반환하는 함수 |
| 만드는 법 | `createElement('h1', ...)` / JSX | `function Helloworld() { return ... }` |
| `createElement`의 첫 인자 | 문자열 `'h1'` | 함수 `Helloworld` |

첫 인자가 문자열이면 HTML 태그를, 함수면 그 함수를 불러 나온 엘리먼트를 그린다. 이 차이가 다음 실습의 "컴포넌트 이름은 대문자" 규칙과 이어진다.

## 결과 (Result)

| 확인 항목 | 결과 |
|-----------|------|
| JSX에 `class` 사용 | 화면 정상, 콘솔에 `Invalid DOM property 'class'` 경고 |
| 만들어진 DOM | `<h1 class="text">안녕! 양 승빈</h1>` |
| Babel 변환 결과 | `React.createElement("h1", { className: "text" }, "안녕! ", formatName(user))` |
| 환경 | React 18.3.1 UMD, @babel/standalone 8.1.3, Playwright Chromium |

배운 점은 다음과 같다.

- JSX는 새 언어가 아니라 `React.createElement` 호출을 짧게 쓰는 문법이다. 변환 결과를 보면 규칙의 이유가 보인다.
- JSX 속성은 HTML이 아니라 DOM 프로퍼티 이름을 따른다(`className`). React는 개발 모드에서 콘솔로 알려 주므로, 화면이 멀쩡해도 콘솔을 확인한다.

## 더 학습하면 좋은 개념

- **가상 DOM과 재조정(Reconciliation)** — 엘리먼트 객체를 비교해 바뀐 부분만 실제 DOM에 반영하는 방식.
- **새 JSX 변환(automatic runtime)** — React 17부터 `import React` 없이 JSX를 쓸 수 있게 된 이유.
- **트랜스파일러와 번들러** — Babel과 Vite가 각각 하는 일.
- **props** — 컴포넌트 함수에 값을 넘기는 방법. `createElement`의 두 번째 인자다.

## 참고 자료

- [React - createElement](https://react.dev/reference/react/createElement)
- [React - Writing Markup with JSX](https://react.dev/learn/writing-markup-with-jsx)
- [React - JavaScript in JSX with Curly Braces](https://react.dev/learn/javascript-in-jsx-with-curly-braces)
- [React - Common components (className)](https://react.dev/reference/react-dom/components/common)
- [Babel - @babel/standalone](https://babeljs.io/docs/babel-standalone)

---

**요약**
1. JSX에 `class`를 쓰자 화면은 정상이었지만 콘솔에 `Invalid DOM property 'class'. Did you mean 'className'?` 경고가 떴다. JSX 속성은 DOM 프로퍼티 이름을 따른다.
2. Babel로 직접 변환해 보니 JSX는 `React.createElement("h1", { className: "text" }, ...)` 호출이었고, 그래서 중괄호 안에는 표현식만 쓸 수 있다.
3. 엘리먼트는 그릴 내용을 담은 객체, 컴포넌트는 엘리먼트를 반환하는 함수이며, `createElement`의 첫 인자가 문자열인지 함수인지로 구분된다.
