---
layout: post
title: "git fetch로 원격 브랜치 가져오기, 그리고 checkout까지"
date: 2026-08-30
---

Git으로 협업을 하다 보면 원격 저장소(remote repository)에 새로운 브랜치가 생기거나, 팀원이 커밋을 추가하는 일이 자주 일어납니다. 이럴 때 로컬 저장소를 최신 상태로 맞추기 위해 사용하는 명령어가 바로 `git fetch`입니다. 이번 글에서는 `git fetch`의 동작 원리와, 가져온 브랜치를 로컬에서 실제로 사용하기 위한 `git checkout` 과정까지 정리해보겠습니다.

## 1. git fetch란?

`git fetch`는 원격 저장소의 커밋, 브랜치, 태그 정보를 **로컬 저장소로 다운로드**하는 명령어입니다. 여기서 중요한 포인트는, fetch는 원격 저장소의 변경 사항을 가져오기만 할 뿐 **현재 작업 중인 브랜치(작업 디렉터리)에는 아무 영향을 주지 않는다**는 점입니다.

```bash
git fetch origin
```

이 명령어를 실행하면 `origin`이라는 원격 저장소의 모든 브랜치 정보가 로컬로 내려옵니다. 이때 로컬에는 `origin/브랜치명` 형태의 **원격 추적 브랜치(remote-tracking branch)**가 생성되거나 갱신됩니다.

### fetch 이후 무슨 일이 일어나는가

예를 들어 원격 저장소에 `feature/login`이라는 브랜치가 있다면, fetch 이후 로컬에는 다음과 같은 참조가 생깁니다.

```
origin/feature/login
```

이 브랜치는 어디까지나 "원격 저장소의 상태를 가리키는 포인터"일 뿐, 여러분이 직접 커밋을 추가하거나 작업할 수 있는 로컬 브랜치는 아닙니다.

## 2. 가져온 브랜치 목록 확인하기

fetch로 받아온 원격 브랜치들은 다음 명령어로 확인할 수 있습니다.

```bash
git branch -r
```

출력 예시:

```
origin/main
origin/feature/login
origin/feature/payment
```

## 3. git checkout으로 로컬 브랜치 만들기

fetch만으로는 해당 브랜치에서 작업을 할 수 없습니다. 원격 추적 브랜치를 기반으로 **로컬 브랜치를 새로 만들어서 전환(checkout)**해야 실제로 코드를 수정하고 커밋할 수 있는 상태가 됩니다.

```bash
git checkout feature/login
```

Git 최신 버전에서는 로컬에 `feature/login` 브랜치가 없고, `origin/feature/login`이라는 원격 추적 브랜치가 존재하면, 자동으로 그 브랜치를 추적하는 로컬 브랜치를 만들어줍니다. 즉 위 명령어는 아래와 동일하게 동작합니다.

```bash
git checkout -b feature/login origin/feature/login
```

- `-b feature/login` : `feature/login`이라는 이름의 새 로컬 브랜치를 생성
- `origin/feature/login` : 이 원격 추적 브랜치를 기준점으로 삼음

이렇게 하면 로컬에 `feature/login` 브랜치가 생기고, 해당 브랜치의 최신 커밋 상태로 작업 디렉터리가 전환됩니다.

## 4. 최신 Git이라면 git switch도 고려해보기

`checkout`은 브랜치 전환, 파일 복원 등 여러 역할을 함께 수행하는 다소 다목적 명령어입니다. Git 2.23 이후로는 브랜치 전환 전용 명령어인 `git switch`가 추가되어, 아래처럼도 사용할 수 있습니다.

```bash
git switch feature/login
```

기능적으로는 위의 `checkout` 예시와 동일하게, 원격 추적 브랜치를 기반으로 로컬 브랜치를 자동 생성하며 전환됩니다.

## 5. 전체 흐름 정리

1. `git fetch origin` → 원격 저장소의 최신 정보를 로컬로 다운로드 (원격 추적 브랜치 갱신)
2. `git branch -r` → 가져온 원격 브랜치 목록 확인
3. `git checkout 브랜치명` (또는 `git switch 브랜치명`) → 원격 추적 브랜치를 기반으로 로컬 브랜치 생성 및 전환

```bash
git fetch origin
git branch -r
git checkout feature/login
```

## 6. git pull과의 차이 (참고)

많이 헷갈리는 부분이라 짧게만 짚고 넘어가면, `git pull`은 사실 `git fetch` + `git merge`(또는 `rebase`)를 한 번에 수행하는 명령어입니다. `fetch`가 "받아오기만" 한다면, `pull`은 "받아오고 현재 브랜치에 바로 합치기"까지 한다는 점에서 차이가 있습니다. 원격 상태를 먼저 확인하고 신중하게 병합하고 싶다면 `fetch` → 확인 → `merge`(또는 `checkout`) 순서로 진행하는 것이 더 안전한 방법입니다.

---

`git fetch`와 `git checkout`(또는 `switch`)의 관계를 이해하면, "받아오기"와 "실제로 그 브랜치에서 작업하기"가 별개의 단계라는 Git의 설계 철학도 자연스럽게 이해할 수 있습니다. 다음 학습 단계로는 `git merge`, `git rebase`, 그리고 fetch한 상태를 병합하는 과정까지 다뤄보면 좋을 것 같습니다.