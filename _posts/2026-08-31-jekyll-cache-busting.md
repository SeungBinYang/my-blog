---
layout: post
title: "배포해도 CSS가 안 바뀌는 문제, 캐시 버스팅으로 해결하기"
date: 2026-08-31 17:00:00 +0900
categories: [Web]
tags: [jekyll, github-pages, cache-busting, http-caching, front-end]
mermaid: true
---

## 들어가며 (Situation)

이 블로그는 GitHub Pages + Jekyll로 만들고 있는데, `_sass`나 `assets/js`를 고치고 push한 뒤 새로고침해도 화면이 바뀌지 않는 일이 반복됐습니다. 배포 로그를 보면 분명 새 빌드가 올라갔는데, 브라우저는 예전 CSS/JS 파일을 그대로 쓰고 있었습니다.

## 문제 상황 (Task)

풀어야 할 문제는 명확했습니다.

- CSS/JS 파일을 수정하고 배포해도, 브라우저가 이전 버전을 캐시에서 그대로 불러온다.
- 사용자(=나 자신)가 매번 하드 리프레시(캐시 무시 새로고침)를 하지 않아도, 새 버전이 자동으로 반영되어야 한다.
- GitHub Pages는 정적 호스팅이라 서버 응답 헤더를 직접 커스터마이징할 수 없다는 제약이 있었습니다.

## 해결 과정 (Action)

### 1. 검토한 대안

| 방법 | 원리 | 장점 | 단점 |
|---|---|---|---|
| 쿼리 스트링 버전 (`?v=`) | 파일 URL 뒤에 버전 값을 붙여 브라우저가 "다른 URL"로 인식하게 함 | Jekyll 변수만으로 구현 가능, 빌드 도구 추가 불필요 | 파일 내용이 실제로 바뀌지 않아도 빌드할 때마다 버전이 갱신됨 |
| 파일명 해시 (fingerprinting) | 파일 내용이 바뀔 때만 파일명 자체가 바뀜 (예: `main.a1b2c3.css`) | 캐시 무효화가 가장 정확함 | Jekyll 기본 기능만으로는 지원되지 않고 별도 플러그인/빌드 파이프라인 필요 |
| Cache-Control 헤더 조정 | 서버가 브라우저에게 캐시 유효기간을 짧게 지정 | 서버 설정만으로 해결, 프론트 코드 변경 불필요 | GitHub Pages는 응답 헤더를 직접 수정할 수 없음 |

GitHub Pages는 정적 호스팅이라 서버 헤더를 만질 수 없고, fingerprinting은 별도 플러그인이 필요해 지금 블로그 규모에는 과했습니다. 그래서 Jekyll 변수만으로 바로 적용할 수 있는 **쿼리 스트링 버전** 방식을 선택했습니다.

### 2. 구현

`_includes/head.html`에서 CSS/JS를 불러오는 줄에 `?v=` 쿼리를 붙이고, 값으로 Jekyll의 빌드 시각(`site.time`)을 유닉스 타임스탬프로 넣었습니다.

```html
<!-- Before -->
<link rel="stylesheet" href="{{ "/assets/css/main.css" | relative_url }}">
<script src="{{ "/assets/js/main.js" | relative_url }}"></script>

<!-- After -->
<link rel="stylesheet" href="{{ "/assets/css/main.css" | relative_url }}?v={{ site.time | date: '%s' }}">
<script src="{{ "/assets/js/main.js" | relative_url }}?v={{ site.time | date: '%s' }}"></script>
```

빌드할 때마다 `site.time`이 갱신되니, 매 배포마다 CSS/JS의 URL 자체가 달라져서 브라우저가 이전 캐시를 재사용하지 않고 새로 받아옵니다.

<pre class="mermaid">
flowchart LR
    A["파일 수정 후 push"] --> B["Jekyll 빌드 (site.time 갱신)"]
    B --> C["main.css?v=새타임스탬프"]
    C --> D{"브라우저에 같은 URL 캐시 있음?"}
    D -->|No, URL이 달라짐| E["새 파일 다운로드"]
    D -.이전엔 URL 고정.-> F["캐시된 예전 파일 재사용"]
</pre>

### 3. 시행착오 — 커밋 하나로 되돌린 걸 못 알아채고 삽질

쿼리 스트링을 추가하는 커밋을 만든 직후, 뒤이은 커밋에서 실수로 그 변경을 다시 원래대로 되돌려버렸습니다. 의도한 롤백이 아니라 착오로 되돌린 것이었는데, 커밋 메시지만 보면 둘 다 "캐시 버스팅 기능 추가"라 diff를 직접 열어보기 전까지는 알아채지 못했습니다. 결국 `head.html`을 다시 열어 `?v=` 쿼리가 사라진 걸 확인하고, 같은 수정을 한 번 더 적용해서 되돌린 내용을 복구했습니다.

이 과정에서 얻은 교훈은, **커밋 메시지가 같아 보여도 실제 diff는 다를 수 있다**는 것입니다. 특히 짧은 시간에 비슷한 작업을 반복할 때는 커밋 전에 `git diff`로 실제 변경 내용을 한 번 더 확인하는 습관이 필요하다고 느꼈습니다.

## 결과 (Result)

| 항목 | Before | After |
|---|---|---|
| CSS/JS 수정 후 배포 시 반영 여부 | 하드 리프레시 없이는 반영 안 됨 | 새로고침만으로 바로 반영됨 |
| 캐시 무효화 방식 | 없음 (파일 URL 고정) | 빌드마다 `?v=` 쿼리 값 변경 |
| 확인 방법 | 육안으로 "안 바뀌었다"고만 인지 | 배포 후 실제로 새 CSS/JS가 반영되는 것을 확인 |

배포 후 새로고침만으로 최신 스타일과 스크립트가 바로 적용되는 것을 확인했습니다. 다만 이 방식은 파일 내용이 실제로 바뀌지 않아도 빌드할 때마다 버전 값이 갱신된다는 한계가 있어서, 트래픽이 커지면 불필요한 재다운로드가 늘어날 수 있다는 점은 감안해야 합니다.

## 더 학습하면 좋은 개념

- **HTTP 캐싱 (Cache-Control, ETag)** — 지금은 쿼리 스트링으로 우회했지만, 원래 캐시 정책은 서버가 응답 헤더로 지정하는 것이 정석입니다. 브라우저가 캐시를 언제 쓰고 언제 버리는지 이해하면 이번 우회 방법의 한계도 명확해집니다.
- **파일 fingerprinting / content hash** — 파일 내용이 실제로 바뀔 때만 캐시를 무효화하는 더 정교한 방식입니다. 지금 쓴 시각 기반 버전 방식과 비교하며 학습하면 좋습니다.
- **Jekyll 빌드 변수 (`site.time` 등)** — `site.time`은 "빌드 시각"이지 "파일 수정 시각"이 아니라는 차이를 이해하면, 왜 매 빌드마다 버전이 바뀌는지 정확히 설명할 수 있습니다.
- **GitHub Pages의 정적 호스팅 제약** — 서버 헤더 커스터마이징이 안 되는 이유와, 그로 인해 캐시 관련 문제를 왜 프론트엔드 쪽 우회로 풀어야 하는지 알아두면 비슷한 제약이 있는 다른 정적 호스팅(Netlify, Vercel 등)과 비교하기 좋습니다.

## 참고 자료

- [MDN - HTTP Caching](https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching)
- [Jekyll 공식 문서 - Variables (site.time)](https://jekyllrb.com/docs/variables/)
- [Jekyll 공식 문서 - Liquid Filters (date)](https://jekyllrb.com/docs/liquid/filters/)
- [GitHub Pages 공식 문서](https://docs.github.com/en/pages)
