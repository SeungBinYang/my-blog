---
layout: post
title: "이번 주에 배운 Git 명령어 정리"
date: 2026-08-31
mermaid: true
---

안녕하세요! 이번 주에 처음으로 Git과 GitHub를 배웠죠?
명령어가 한꺼번에 쏟아지면 헷갈리기 마련이라, 이번 주에 배운 것만 순서대로 다시 정리해보려고 합니다.
"이걸 왜 쓰는지"부터 이해하면 명령어는 저절로 외워지니, 하나씩 천천히 따라와 보세요.

## 1. 큰 그림부터 보기

Git으로 코드를 관리할 때, 파일은 네 개의 공간을 순서대로 이동합니다.
아직 이름이 낯설겠지만, 아래 그림을 보면서 "아, 이런 흐름이구나" 정도만 느껴보세요.

<pre class="mermaid">
flowchart LR
    A["작업 디렉터리<br/>(내가 파일을 수정하는 곳)"] -- "git add" --> B["스테이징 영역<br/>(커밋할 파일을 고르는 곳)"]
    B -- "git commit" --> C["Local Repository<br/>(내 컴퓨터 안의 저장소)"]
    C -- "git push" --> D["원격 저장소<br/>(GitHub)"]
    D -- "git pull" --> A
</pre>

이 그림 하나만 이해해도 이번 주 수업의 절반은 끝난 셈입니다.
이제 각 단계를 하나씩 짚어볼게요.

## 2. 저장소 만들기

내 폴더를 Git이 관리하게 하려면, 먼저 그 폴더를 "저장소(repository)"로 선언해야 합니다.
이 명령어를 실행하면 폴더 안에 숨김 폴더(`.git`)가 생기는데, 여기에 앞으로의 모든 기록이 저장됩니다.

|명령어 | 언제 쓰나|
|---|---|
|`git init`| 새 폴더를 Git 저장소로 만들고 싶을 때 (딱 한 번만 실행)|

## 3. 스테이징

파일을 수정했다고 해서 바로 기록(커밋)되는 것은 아닙니다.
"이번 기록에 포함할 파일"을 먼저 골라야 하는데, 이 과정을 스테이징이라고 부릅니다.

|명령어 | 언제 쓰나|
|---|---|
|`git add 파일명`| 특정 파일 하나만 이번 기록에 포함하고 싶을 때|
|`git add .`| 수정한 파일을 전부 이번 기록에 포함하고 싶을 때|

> 왜 굳이 골라야 할까요? 예를 들어 파일 두 개를 고쳤는데, 그중 하나만 아직 완성이 안 됐다면 완성된 것만 먼저 스테이징해서 기록할 수 있기 때문입니다.

## 4. Local Repository (로컬 저장소)

스테이징한 내용을 `git commit`으로 기록하면, 그 기록은 내 컴퓨터 안에 있는 **Local Repository**에 차곡차곡 쌓입니다.
아직 GitHub에는 아무것도 올라가지 않은 상태예요. 즉, Local Repository는 "내 컴퓨터 안에서만 존재하는 나만의 기록 보관함"이라고 생각하면 됩니다.

<pre class="mermaid">
flowchart LR
    subgraph Local["내 컴퓨터"]
        direction LR
        S["스테이징 영역"] -- "git commit" --> R["Local Repository<br/>(커밋 1) → (커밋 2) → (커밋 3)"]
    end
</pre>

## 5. 커밋

커밋은 "지금까지 스테이징한 내용을 하나의 기록으로 남기는 것"입니다.
사진을 찍듯이, 그 순간의 상태를 저장한다고 생각하면 이해하기 쉽습니다.

|명령어 | 언제 쓰나|
|---|---|
|`git commit -m "메시지"`| 스테이징한 내용을 하나의 기록으로 남기고 싶을 때|

메시지는 "무엇을 왜 바꿨는지"를 짧게 남기는 부분이에요. 나중에 기록을 되돌아볼 때 큰 도움이 됩니다.

## 6. 브랜치

브랜치는 원본(main)을 건드리지 않고, 안전하게 실험할 수 있는 "또 다른 작업 공간"입니다.
main 브랜치는 그대로 둔 채, 새 브랜치에서 마음껏 작업하다가 문제가 없으면 나중에 합치면 됩니다.

|명령어 | 언제 쓰나|
|---|---|
|`git branch 브랜치명`| 새로운 브랜치를 만들고 싶을 때|
|`git checkout 브랜치명`| 만들어둔 브랜치로 이동(전환)하고 싶을 때|

<pre class="mermaid">
flowchart TD
    M1["main: 커밋 1"] --> M2["main: 커밋 2"]
    M2 --> M3["main: 커밋 3"]
    M2 -- "git branch feature" --> B1["feature: 커밋 A"]
    B1 --> B2["feature: 커밋 B"]
</pre>

main과 feature 브랜치가 커밋 2 지점부터 갈라져서, 서로 다른 기록을 쌓아가는 모습입니다.

## 7. push와 pull

지금까지는 전부 내 컴퓨터(Local) 안에서만 일어난 일입니다. 이제 GitHub(원격 저장소)와 주고받을 차례예요.

|명령어 | 언제 쓰나|
|---|---|
|`git push`| 내 Local Repository의 기록을 GitHub에 올리고 싶을 때|
|`git pull`| GitHub에 있는 최신 기록을 내 컴퓨터로 받아오고 싶을 때|

<pre class="mermaid">
flowchart LR
    L["Local Repository<br/>(내 컴퓨터)"] -- "git push" --> Rmt["원격 저장소<br/>(GitHub)"]
    Rmt -- "git pull" --> L
</pre>

## 8. merge

merge는 서로 다른 브랜치에 쌓인 기록을 하나로 합치는 작업입니다.
보통은 feature 브랜치에서 작업을 끝낸 뒤, main 브랜치로 이동해서 merge를 실행합니다.

|명령어 | 언제 쓰나|
|---|---|
|`git merge 브랜치명`| 다른 브랜치의 기록을 지금 브랜치로 합치고 싶을 때|

<pre class="mermaid">
flowchart LR
    M3["main: 커밋 3"] -- "git merge feature" --> M4["main: 커밋 4<br/>(feature 내용이 합쳐짐)"]
    B2["feature: 커밋 B"] --> M4
</pre>

## 9. 되돌리기

실수로 잘못된 내용을 커밋했을 때, 이전 상태로 되돌릴 수 있습니다.

|명령어 | 언제 쓰나|
|---|---|
|`git reset 커밋 지점`| 특정 커밋 시점으로 기록 자체를 되돌리고 싶을 때|
|`git revert 커밋 지점`| 기존 기록은 남겨둔 채, 그 변경 내용만 취소하는 새 커밋을 남기고 싶을 때|

<pre class="mermaid">
flowchart LR
    C1["커밋 1"] --> C2["커밋 2"] --> C3["커밋 3 (실수!)"]
    C3 -. "git reset" .-> C2
</pre>

> `reset`은 기록을 아예 그 시점으로 되돌리고, `revert`는 "취소했다"는 기록을 새로 남긴다는 차이가 있습니다. 아직은 두 가지가 있다는 것만 기억해도 충분해요.

## 10. 전체 명령어 요약표

|명령어 | 언제 쓰나|
|---|---|
|`git init`| 폴더를 Git 저장소로 만들고 싶을 때|
|`git add 파일명`| 파일을 스테이징 영역에 올리고 싶을 때|
|`git commit -m "메시지"`| 스테이징한 내용을 Local Repository에 기록하고 싶을 때|
|`git branch 브랜치명`| 새 브랜치를 만들고 싶을 때|
|`git checkout 브랜치명`| 다른 브랜치로 이동하고 싶을 때|
|`git push`| Local Repository의 기록을 GitHub에 올리고 싶을 때|
|`git pull`| GitHub의 최신 기록을 내 컴퓨터로 받아오고 싶을 때|
|`git merge 브랜치명`| 다른 브랜치의 기록을 합치고 싶을 때|
|`git reset` / `git revert`| 이전 상태로 되돌리고 싶을 때|

## 마무리

이번 주는 "혼자 파일을 저장하는 법(저장소 만들기 → 스테이징 → 커밋)"과 "여럿이 함께 작업하는 법(브랜치 → push → pull → merge)", 그리고 "실수를 되돌리는 법"까지 Git의 뼈대를 전부 훑었습니다.

처음에는 명령어 순서가 헷갈릴 수 있지만, 결국은 아래 흐름 하나로 귀결됩니다.

<pre class="mermaid">
flowchart LR
    Init["git init"] --> Add["git add"]
    Add --> Commit["git commit"]
    Commit --> Branch["git branch / checkout"]
    Branch --> Push["git push"]
    Push --> Pull["git pull"]
    Pull --> Merge["git merge"]
    Merge --> Reset["git reset / revert<br/>(필요할 때만)"]
</pre>

다음에는 이 흐름을 직접 손으로 몇 번 더 반복해보면서 익숙해지는 게 좋겠습니다.
