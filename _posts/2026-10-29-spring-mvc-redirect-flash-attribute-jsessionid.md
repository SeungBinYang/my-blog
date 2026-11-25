---
layout: post
title: "flash attribute를 쓰는 첫 클릭만 404가 났다 — redirect, RedirectAttributes, 그리고 ;jsessionid"
date: 2026-10-29 11:15:00 +0900
categories: [Java]
tags: [java, spring, spring-mvc, redirect, session]
mermaid: true
---

## 들어가며 (Situation)

`chap03-view-resolver`에서는 핸들러 메서드가 화면(View)을 돌려주는 방법을 정리했다.

| 반환 타입 | View 이름이 정해지는 방법 |
|-----------|--------------------------|
| `void` | 요청 URL이 곧 View 경로 |
| `String` | 반환한 문자열 (`"result"` → `templates/result.html`) |
| `ModelAndView` | `mv.setViewName("result")` |

기본은 **forward**다. redirect가 필요하면 문자열 앞에 `redirect:`를 붙인다. [Servlet에서 PRG 패턴]({% post_url 2026-10-22-servlet-sendredirect-prg %})을 위해 `sendRedirect()`를 직접 불렀던 일을 Spring에서는 문자열 하나로 한다.

```java
@GetMapping("/string-redirect")
public String redirect(Model model) {
    System.out.println("회원가입 완료!!");
    // redirect 되면서 사라지므로 main 화면에 출력되지 않는다.
    model.addAttribute("flashMessage", "Model 에 담은 값");
    return "redirect:/";
}

@GetMapping("/string-redirect-attr")
public String redirectFlashAttr(RedirectAttributes rttr) {
    // flashAttribute 는 세션에 잠시 보관되었다가 redirect 된 요청에서 한 번 사용 후 삭제된다.
    rttr.addFlashAttribute("flashMessage", "리다이렉트 된 페이지에서도 유지되는 값!");
    return "redirect:/";
}
```

메인 화면(`main.html`)은 맨 아래에 `<h3 th:text="${flashMessage}">`를 출력한다.

## 문제 상황 (Task)

수업 코드가 말하는 내용을 확인하고 싶었다.

1. `Model`에 담은 값은 redirect 뒤에 정말 사라질까?
2. flash attribute는 정말 "한 번만" 보일까?

그런데 브라우저로 확인하다가 예상 못 한 결과가 나왔다. 새로 연 브라우저에서 메인 화면을 열고 flash 버튼을 누르면 **404**가 났다. 주소창은 `http://localhost:8080/;jsessionid=92ECB6...`였다. 뒤로 가서 같은 버튼을 다시 누르면 이번에는 정상으로 메시지가 보였다.

같은 버튼인데 첫 클릭만 실패하는 이유를 찾아야 했다.

## 해결 과정 (Action)

### 1. 응답 헤더부터 보기

curl로 응답 헤더를 확인했다.

```bash
curl -s -i localhost:8080/string-redirect
curl -s -i localhost:8080/string-redirect-attr
```

```text
# /string-redirect
HTTP/1.1 302
Location: http://localhost:8080/

# /string-redirect-attr
HTTP/1.1 302
Set-Cookie: JSESSIONID=8DB08D3962F2B10483177D10B20AD4E2; Path=/; HttpOnly
Location: http://localhost:8080/;jsessionid=8DB08D3962F2B10483177D10B20AD4E2
```

두 응답은 하나만 달랐다. flash attribute를 쓴 쪽만 **세션 쿠키를 새로 만들고**, `Location`에 `;jsessionid=...`가 붙었다.

### 2. flash attribute는 세션에 저장된다

flash attribute는 redirect 전 요청과 후 요청 사이에 값을 넘겨야 한다. 두 요청은 서로 다른 요청이라 `Model`(요청 범위)로는 넘길 수 없다. 그래서 Spring은 값을 **세션에 잠깐** 보관한다. 문서도 그렇게 설명한다.

> Flash attributes are saved temporarily before the redirect (typically in the session) to be made available to the request after the redirect and are removed immediately.

메인 화면은 세션을 만들지 않는다. 그래서 flash 버튼을 누른 그 요청에서 **처음으로** 세션이 생긴다.

### 3. 세션이 처음 생기면 Tomcat은 URL에도 세션 ID를 붙인다

Tomcat은 브라우저가 쿠키를 받아 줄지 아직 모른다. 그래서 세션을 처음 만든 응답에서는 쿠키도 보내고, redirect 주소에도 `;jsessionid=...`를 붙인다(URL rewriting). 브라우저가 쿠키를 받지 않아도 세션을 이어 가기 위한 장치다.

### 4. 그런데 Spring은 `/;jsessionid=...`를 `/`로 보지 않았다

경로 파라미터가 붙은 주소를 직접 요청해 봤다.

```bash
curl -s -o /dev/null -w "%{http_code}" "localhost:8080/;jsessionid=abc"      # 404
curl -s -o /dev/null -w "%{http_code}" "localhost:8080/main;jsessionid=abc"  # 200
curl -s -o /dev/null -w "%{http_code}" "localhost:8080/"                     # 200
```

`/main;jsessionid=abc`는 `/main`으로 잘 매칭됐다. 그런데 **루트 `/`** 뒤에 붙은 경우만 404였다. 수업 코드의 redirect 대상이 하필 `/`였다.

정리하면 첫 클릭은 이렇게 흘러갔다.

```mermaid
sequenceDiagram
    participant B as 브라우저 (쿠키 없음)
    participant S as Spring + Tomcat
    B->>S: GET /string-redirect-attr
    Note over S: flash 값을 세션에 저장<br/>→ 세션 최초 생성
    S-->>B: 302 Set-Cookie: JSESSIONID<br/>Location: /;jsessionid=...
    B->>S: GET /;jsessionid=...
    S-->>B: 404 (루트 + 경로 파라미터 매칭 실패)
    Note over B: 두 번째 클릭부터는 쿠키가 있어<br/>Location이 그냥 / → 200
```

두 번째 클릭에서는 브라우저가 이미 쿠키를 보내므로 Tomcat이 URL rewriting을 하지 않는다. `Location: /`로 가서 정상이었다. Playwright로 새 브라우저를 열어 똑같이 해 보니 첫 클릭은 404, 두 번째 클릭은 "리다이렉트 된 페이지에서도 유지되는 값!"이 보였다.

### 5. 해결: 세션 추적을 쿠키로만 하기

URL에 세션 ID를 붙이는 방식은 세션 ID가 주소창, 로그, Referer 헤더로 새어 나갈 수 있어서 요즘은 잘 쓰지 않는다. Spring Boot에서는 세션 추적 방식을 설정 하나로 바꿀 수 있다.

```yaml
server:
  servlet:
    session:
      tracking-modes: cookie
```

이 설정으로 다시 띄워 보니(실행 인자 `--server.servlet.session.tracking-modes=cookie`) 결과가 달라졌다.

```text
HTTP/1.1 302
Location: http://localhost:8080/
<h3>리다이렉트 된 페이지에서도 유지되는 값!</h3>
```

### 6. 처음 질문 확인: Model 값은 사라지고, flash 값은 한 번만 보인다

| 요청 | 메인 화면의 `flashMessage` |
|------|----------------------------|
| `/string-redirect` (Model에 담음) | 빈 값 |
| `/string-redirect-attr` 직후 | "리다이렉트 된 페이지에서도 유지되는 값!" |
| 그 상태에서 새로고침 | 빈 값 (한 번 쓰고 삭제됨) |

`Model`에 담은 값은 redirect 주소에도 붙지 않았다(`Location: /`). 옛 Spring 버전은 `Model`의 단순 값을 redirect URL의 쿼리 스트링으로 붙이기도 했지만, Spring 6부터는 기본으로 붙이지 않는다.

## 결과 (Result)

| 상황 | 결과 |
|------|------|
| 세션 없는 상태에서 flash redirect (기본 설정) | `Location: /;jsessionid=...` → **404** |
| 세션 쿠키가 있는 상태 | `Location: /` → 200, 메시지 표시 |
| `tracking-modes: cookie` | 첫 클릭부터 `Location: /` → 200 |
| `/main;jsessionid=abc` | 200 (루트 경로만 문제) |
| 환경 | Spring Boot 4.1.2, spring-webmvc 7.0.10, Tomcat 11.0.27 |

배운 점은 다음과 같다.

- flash attribute는 **세션**에 저장된다. 그래서 세션이 처음 생기는 요청에서는 Tomcat의 URL rewriting(`;jsessionid`)까지 함께 일어난다.
- 같은 버튼이 첫 번째에만 실패하면 "처음에만 있는 것"을 찾는다. 이번에는 세션 생성이었다. 응답 헤더(`Set-Cookie`, `Location`)를 비교하니 원인이 바로 보였다.

## 더 학습하면 좋은 개념

- **세션 고정(Session Fixation) 공격** — URL에 세션 ID가 드러나면 생기는 대표적인 보안 문제. Spring Security가 로그인 시 세션 ID를 바꾸는 이유다.
- **forward vs redirect** — 요청이 1번인지 2번인지에 따라 `Model`이 살아남는지가 결정된다.
- **`ViewResolver`의 prefix/suffix** — `"result"`가 `classpath:/templates/result.html`이 되는 과정.
- **쿠키 속성 (`HttpOnly`, `Secure`, `SameSite`)** — `JSESSIONID` 쿠키에 붙은 `HttpOnly`의 의미.

## 참고 자료

- [Spring Framework Reference - Flash Attributes](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller/ann-methods/flash-attributes.html)
- [Spring Framework Reference - Redirecting to a URL](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller/ann-methods/redirecting-passing-data.html)
- [Spring Framework Reference - View Resolution (redirect: prefix)](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-servlet/viewresolver.html)
- [Spring Boot - Common Application Properties (server.servlet.session.tracking-modes)](https://docs.spring.io/spring-boot/appendix/application-properties/index.html)
- [Jakarta Servlet API - SessionTrackingMode](https://jakarta.ee/specifications/servlet/6.0/apidocs/jakarta.servlet/jakarta/servlet/sessiontrackingmode)

---

**요약**
1. flash attribute는 세션에 저장되므로, 세션이 없던 첫 클릭에서 Tomcat이 `Location: /;jsessionid=...`로 redirect했고 루트 경로라서 404가 났다.
2. 두 번째 클릭부터는 쿠키가 있어 `Location: /`가 되어 정상이었고, `server.servlet.session.tracking-modes: cookie`로 첫 클릭 문제도 사라졌다.
3. `Model`에 담은 값은 redirect 뒤에 사라지고, flash 값은 redirect 직후 한 번만 보이고 새로고침하면 사라진다.
