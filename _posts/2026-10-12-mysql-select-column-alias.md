---
layout: post
title: "별칭에 띄어쓰기를 넣으면 왜 에러가 날까 — MySQL SELECT와 별칭(as) 기초"
date: 2026-10-12 14:00:00 +0900
categories: [Database]
tags: [mysql, sql, select, alias, sql기초]
mermaid: true
---

## 들어가며 (Situation)

자바 챕터를 마치고 오늘부터 MySQL 수업이 시작됐다. 수업 폴더 `04_MySQL`에는 SQL 파일 7개가 생겼고, 두 개의 실습 데이터베이스를 사용했다.

| 데이터베이스 | 주요 테이블 | 사용한 파일 |
|-------------|-------------|-------------|
| `menudb` | `tbl_menu`(메뉴 36개), `tbl_category`(카테고리 12개) | `01_select` ~ `05_limit` |
| `wanted_lms` | `users`(120명), `courses`(30개), `enrollments`, `grades` 등 | `06_grouping`, `07_join` |

양이 많아서 개념 단위로 6편으로 나눠 정리한다.

| 편 | 주제 |
|----|------|
| 1 | SELECT와 별칭 (이 글) |
| 2 | [ORDER BY와 NULL 정렬]({{ site.baseurl }}{% post_url 2026-10-12-mysql-order-by-null-sorting %}) |
| 3 | [WHERE 조건절과 연산자]({{ site.baseurl }}{% post_url 2026-10-12-mysql-where-condition-operators %}) |
| 4 | [DISTINCT와 LIMIT 페이징]({{ site.baseurl }}{% post_url 2026-10-12-mysql-distinct-limit-paging %}) |
| 5 | [집계 함수, GROUP BY, HAVING]({{ site.baseurl }}{% post_url 2026-10-12-mysql-aggregate-group-by-having %}) |
| 6 | [JOIN]({{ site.baseurl }}{% post_url 2026-10-12-mysql-join-inner-left-right %}) |

모든 결과는 로컬 MySQL 8.0.46에서 수업 쿼리를 다시 실행해서 얻은 실제 출력이다.

## 문제 상황 (Task)

첫 파일 `01_select.sql`의 주석은 이렇게 시작한다.

> select 절은 MySQL 의 가장 기본적 명령어로 원하는 데이터를 조회할 수 있는 명령어이다.

그리고 별칭(alias)을 붙이는 부분에 이런 주석을 달았다.

> 별칭 안에 띄어쓰기가 포함되는 경우 꼭 따옴표로 감싸줘야 함.

복습하면서 확인하고 싶은 것이 세 가지 생겼다.

1. 따옴표 없이 띄어쓰기를 넣으면 정확히 어떻게 되는가? `as`는 꼭 써야 하는가?
2. `select now();`처럼 `from`이 없는 SELECT는 어떻게 동작하는가?
3. 파일을 처음부터 실행했더니 첫 쿼리부터 에러가 났다. 왜?

## 해결 과정 (Action)

### 1. SELECT의 기본 골격 — "무엇을, 어디에서"

수업 주석이 문장을 그대로 SQL에 대응시켜 준 게 이해하기 좋았다.

```sql
select       -- 조회해줘
    menu_name -- 무엇을? (컬럼)
from         -- 어디에서
    tbl_menu; -- (데이터베이스 테이블)
```

```text
+----------------------+
| menu_name            |
+----------------------+
| 열무김치라떼         |
| 우럭스무디           |
| 생갈치쉐이크         |
| ...                  |
+----------------------+
36 rows in set (0.00 sec)
```

결과는 **표 모양의 결과 집합(result set)**으로 돌아온다. 원본 테이블은 열이 5개인데 내가 고른 `menu_name` 한 열만 왔다.

```mermaid
flowchart LR
    A["tbl_menu<br/>(5개 열 × 36행)"] -->|"select menu_name"| B["결과 집합<br/>(1개 열 × 36행)"]
```

열을 여러 개 고를 때는 쉼표로 나열하고, 나열한 순서대로 결과에 나온다.

```sql
select
    menu_code,
    category_code,
    menu_price
from
    tbl_menu;
```

```text
+-----------+---------------+------------+
| menu_code | category_code | menu_price |
+-----------+---------------+------------+
|         1 |             8 |       4500 |
|         2 |            10 |       5000 |
|       ... |           ... |        ... |
+-----------+---------------+------------+
36 rows in set (0.00 sec)
```

### 2. `*`(애스터리스크)는 편하지만 조심해야 한다

`*`는 모든 열을 뜻한다.

```sql
select * from tbl_menu;
```

```text
+-----------+--------------+------------+---------------+------------------+
| menu_code | menu_name    | menu_price | category_code | orderable_status |
+-----------+--------------+------------+---------------+------------------+
|         1 | 열무김치라떼 |       4500 |             8 | Y                |
|       ... | ...          |        ... |           ... | ...              |
+-----------+--------------+------------+---------------+------------------+
36 rows in set (0.00 sec)
```

테이블 구조를 처음 볼 때는 편하다. 그런데 `06_grouping.sql`에서 `select * from users;`를 실행했을 때 `password_hash` 열까지 전부 화면에 나왔다. 실습용 가짜 값이지만, 실제 서비스였다면 필요 없는 민감한 열까지 꺼내온 셈이다. 그래서 정리한 기준은 이렇다.

| 상황 | 선택 |
|------|------|
| 처음 보는 테이블 구조 확인 | `select *` |
| 실제로 쓸 데이터 조회 | 필요한 열만 이름으로 나열 |

### 3. 별칭(as)과 따옴표 실험

수업 코드는 `as`와 작은따옴표로 별칭을 붙였다.

```sql
select
    menu_name as '메뉴 이름',
    menu_price as '가격'
from
    tbl_menu;
```

```text
+----------------------+--------+
| 메뉴 이름            | 가격   |
+----------------------+--------+
| 열무김치라떼         |   4500 |
| ...                  |    ... |
+----------------------+--------+
36 rows in set (0.00 sec)
```

**1번 질문 확인**: 따옴표를 빼고 띄어쓰기를 넣어봤다. (확인용 **예시 코드**)

```sql
select menu_name as 메뉴 이름 from tbl_menu limit 2;
```

```text
ERROR 1064 (42000): You have an error in your SQL syntax; ... near '이름 from tbl_menu limit 2'
```

`메뉴`까지만 별칭으로 읽고, 그 뒤의 `이름`을 해석할 수 없는 단어로 보고 문법 에러를 냈다. SQL은 공백으로 단어를 구분하기 때문에 공백이 들어간 이름은 따옴표로 묶어 "한 덩어리"라고 알려줘야 한다.

그럼 `as`는 필수일까? `06_grouping.sql`에는 `avg(score) '평균'`처럼 `as` 없이 쓴 줄이 있는데 잘 동작했다. 띄어쓰기 없는 별칭으로 다시 확인해봤다. (확인용 **예시 코드**)

```sql
select menu_name 메뉴이름 from tbl_menu limit 2;
```

```text
+--------------------+
| 메뉴이름           |
+--------------------+
| 열무김치라떼       |
| 우럭스무디         |
+--------------------+
2 rows in set (0.00 sec)
```

`as`는 생략할 수 있다([MySQL 8.0 Reference - SELECT Statement](https://dev.mysql.com/doc/refman/8.0/en/select.html)). 다만 열 이름 두 개 사이에 쉼표를 빠뜨려도 뒤의 것이 별칭으로 처리되어 에러 없이 이상한 결과가 나올 수 있다. 의도를 드러내기 위해 `as`를 쓰는 편이 낫겠다.

| 쓰는 방법 | 결과 |
|-----------|------|
| `menu_name as '메뉴 이름'` | 성공 |
| `menu_name as 메뉴이름` | 성공 (공백 없음) |
| `menu_name 메뉴이름` | 성공 (`as` 생략) |
| `menu_name as 메뉴 이름` | **ERROR 1064** |

### 4. 따옴표는 두 종류다 — `'`와 `` ` ``

`02_orderby.sql`에서는 별칭을 백틱(`` ` ``)으로 감쌌다.

```sql
menu_code * menu_price as `코드*가격`
```

MySQL에서 작은따옴표 `'...'`는 **문자열 값**, 백틱 `` `...` ``은 테이블·열·별칭 같은 **이름(식별자)**을 감싸는 기호다([MySQL 8.0 Reference - Schema Object Names](https://dev.mysql.com/doc/refman/8.0/en/identifiers.html)). `as` 바로 뒤에서는 둘 다 별칭으로 받아주기 때문에 SELECT 절에서는 차이가 보이지 않는다. 그런데 이 별칭을 `ORDER BY`에서 다시 부를 때는 결과가 완전히 달라졌다. 이 실험은 [2편 ORDER BY]({{ site.baseurl }}{% post_url 2026-10-12-mysql-order-by-null-sorting %})에 정리했다.

### 5. FROM 없는 SELECT — 값 계산기

**2번 질문 확인**: 수업 마지막에 MySQL 기본 함수를 실행했다.

```sql
select now();
select concat('조', '평훈');
```

```text
+---------------------+
| now()               |
+---------------------+
| 2026-10-12 15:35:28 |
+---------------------+

+-------------------------+
| concat('조', '평훈')    |
+-------------------------+
| 조평훈                  |
+-------------------------+
```

테이블을 읽지 않고 식의 값만 계산해서 한 행짜리 결과로 돌려준다. 몇 가지를 더 넣어봤다. (확인용 **예시 코드**)

```sql
select 1 + 1, concat('조', null), concat('조', '평훈');
```

```text
+-------+---------------------+-------------------------+
| 1 + 1 | concat('조', null)  | concat('조', '평훈')    |
+-------+---------------------+-------------------------+
|     2 | NULL                | 조평훈                  |
+-------+---------------------+-------------------------+
```

눈에 띈 건 `concat('조', null)`이 `'조'`가 아니라 `NULL`이라는 점이다. 공식 문서도 인자 중 하나라도 `NULL`이면 `CONCAT()`은 `NULL`을 반환한다고 설명한다([MySQL 8.0 Reference - CONCAT](https://dev.mysql.com/doc/refman/8.0/en/string-functions.html#function_concat)). 열 값을 이어붙일 때 빈 값이 하나라도 있으면 결과 전체가 사라질 수 있다는 뜻이다.

### 6. "No database selected" — 01번 파일만 실패한 이유

**3번 질문 확인**: 터미널에서 데이터베이스를 지정하지 않고 첫 쿼리를 실행하면 이렇게 나온다.

```text
ERROR 1046 (3D000): No database selected
```

`tbl_menu`가 어느 데이터베이스에 있는지 모르기 때문이다. 수업 중에는 Workbench에 기본 스키마가 선택된 상태여서 문제가 없었던 것으로 보이고, `03_where.sql`부터는 파일 맨 위에 직접 지정했다.

```sql
use menudb;
```

`USE`는 이후 쿼리의 기본 데이터베이스를 정한다([MySQL 8.0 Reference - USE Statement](https://dev.mysql.com/doc/refman/8.0/en/use.html)). 파일만 따로 실행해도 동작하게 하려면 `01`, `02` 파일에도 맨 위에 `use menudb;`를 넣어두는 게 맞다.

## 결과 (Result)

| 확인한 내용 | 결과 |
|-------------|------|
| `select menu_name from tbl_menu` | 5개 열 중 1개 열 × 36행 |
| `select * from users` | `password_hash`까지 노출 → 필요한 열만 고르기로 |
| 공백 있는 별칭, 따옴표 없음 | ERROR 1064 |
| `as` 생략 | 정상 동작 (가독성 때문에 `as` 사용) |
| `concat('조', null)` | `NULL` |
| DB 지정 없이 실행 | ERROR 1046 → `use menudb;` 추가 |

배운 점은 다음과 같다.

- SELECT는 "어떤 열을 가져올 것인가"를 정하고, 결과는 항상 표 모양의 결과 집합이다.
- 작은따옴표는 문자열, 백틱은 이름이다. SELECT 절에서는 차이가 숨겨져 있을 뿐이다.
- `NULL`은 연산에 섞이면 결과 전체를 `NULL`로 만든다. 이후 WHERE, ORDER BY, 집계 함수에서도 계속 등장하는 주제였다.

## 더 학습하면 좋은 개념

- **SQL 논리적 실행 순서** — SELECT는 맨 앞에 쓰지만 FROM보다 나중에 처리된다. 별칭을 어디서 쓸 수 있는지가 이 순서로 정해진다([5편]({{ site.baseurl }}{% post_url 2026-10-12-mysql-aggregate-group-by-having %})에서 다룸).
- **식별자 따옴표와 `ANSI_QUOTES` 모드** — MySQL은 기본적으로 큰따옴표를 문자열로 보지만, 설정에 따라 이름으로 볼 수도 있다. 다른 DB와 쿼리를 공유할 때 중요하다.
- **`CONCAT_WS()`와 `IFNULL()`** — `NULL`이 섞여도 문자열을 안전하게 이어붙이는 방법이다.
- **스키마(Schema)와 데이터베이스** — MySQL에서 `menudb`, `wanted_lms` 같은 단위를 왜 스키마라고도 부르는지 알면 Workbench 화면이 이해된다.

## 참고 자료

- [MySQL 8.0 Reference - SELECT Statement](https://dev.mysql.com/doc/refman/8.0/en/select.html)
- [MySQL 8.0 Reference - Schema Object Names](https://dev.mysql.com/doc/refman/8.0/en/identifiers.html)
- [MySQL 8.0 Reference - String Functions (CONCAT)](https://dev.mysql.com/doc/refman/8.0/en/string-functions.html#function_concat)
- [MySQL 8.0 Reference - Date and Time Functions (NOW)](https://dev.mysql.com/doc/refman/8.0/en/date-and-time-functions.html#function_now)
- [MySQL 8.0 Reference - USE Statement](https://dev.mysql.com/doc/refman/8.0/en/use.html)

---

**요약**
1. SELECT는 "어떤 열을 가져올 것인가"를 정하는 문법이고, `*`는 편하지만 `password_hash` 같은 불필요한 열까지 가져온다.
2. 공백이 있는 별칭은 따옴표가 없으면 ERROR 1064가 나고, `as`는 생략할 수 있지만 의도를 드러내려고 쓴다.
3. `concat('조', null)`은 `NULL`이 되고, DB를 지정하지 않으면 ERROR 1046이 나서 `use menudb;`가 필요하다.
