---
layout: post
title: "홍길동과 같은 해에 가입한 사람이 120명 전부였다 — MySQL 서브쿼리(WHERE·FROM)"
date: 2026-10-13 09:50:00 +0900
categories: [Database]
tags: [mysql, sql, subquery, derived-table]
mermaid: true
---

## 들어가며 (Situation)

MySQL 수업 2일차. 오늘은 `08_subqueries.sql`부터 `14_transaction.sql`까지 파일 7개를 만들었다. 어제 6편에 이어 개념 단위로 7편을 더 쓴다.

| 편 | 주제 | 파일 |
|----|------|------|
| 7 | 서브쿼리 (이 글) | `08_subqueries.sql` |
| 8 | [DDL — CREATE·ALTER·DROP·TRUNCATE]({% post_url 2026-10-13-mysql-ddl-create-alter-drop-truncate %}) | `09_ddl.sql` |
| 9 | [DML — INSERT·UPDATE·DELETE와 제약 조건 에러]({% post_url 2026-10-13-mysql-dml-insert-update-delete %}) | `10_dml.sql` |
| 10 | [UNION과 UNION ALL]({% post_url 2026-10-13-mysql-union-union-all %}) | `11_union.sql` |
| 11 | [VIEW]({% post_url 2026-10-13-mysql-view-virtual-table %}) | `12_view.sql` |
| 12 | [INDEX와 EXPLAIN]({% post_url 2026-10-13-mysql-index-explain %}) | `13_index.sql` |
| 13 | [트랜잭션 — COMMIT·ROLLBACK]({% post_url 2026-10-13-mysql-transaction-commit-rollback %}) | `14_transaction.sql` |

모든 결과는 로컬 MySQL 8.0.46의 `wanted_lms` 데이터베이스에서 다시 실행한 실제 출력이다. 데이터를 바꾸는 실험은 트랜잭션 안에서 하고 `ROLLBACK`으로 되돌렸다.

수업 주석은 서브쿼리를 이렇게 정의했다.

> 서브 쿼리란, 문제의 조건이 충분하지 않을 때 2개 이상의 쿼리문을 조합해서 1개의 쿼리문으로 만드는 것을 의미한다.

## 문제 상황 (Task)

수업 파일에는 문제가 두 개 적혀 있었다.

1. SQL 기초 강의 수강생들의 평균 점수보다 높은 점수를 받은 학생의 이름은?
2. '홍길동'과 같은 년도에 가입한 모든 사용자의 이름과 가입일은?

2번은 수업에서 풀었는데 결과가 **120행**, 즉 `users` 테이블 전체가 나왔다. 조건을 걸었는데 전부 나오면 보통은 조건이 틀린 것이다. 1번은 주석만 있고 코드가 없었다. 그래서 확인할 것을 정리했다.

1. 120행은 쿼리가 틀린 것인가, 데이터가 원래 그런 것인가?
2. 서브쿼리가 값을 하나가 아니라 여러 개 돌려주면 어떻게 되는가?
3. FROM 절 서브쿼리는 JOIN이나 다른 문법으로 바꿀 수 있는가?
4. 비어 있는 1번 문제를 직접 풀 수 있는가?

## 해결 과정 (Action)

### 1. 두 단계로 나눠서 풀기

수업은 문제를 먼저 두 단계로 나눴다.

```sql
-- 1. '홍길동'의 가입 년도
select name, year(created_at)
from users
where name = '홍길동';
```

```text
+-----------+------------------+
| name      | year(created_at) |
+-----------+------------------+
| 홍길동    |             2024 |
+-----------+------------------+
1 row in set (0.00 sec)
```

2단계는 `where year(created_at) = 2024`처럼 1단계 결과를 직접 써넣으면 된다. 하지만 홍길동의 가입일이 바뀌면 숫자도 손으로 바꿔야 한다. 그래서 1단계 쿼리를 **괄호로 감싸 2단계 안에 넣는다**.

```sql
select name, created_at
from users
where year(created_at) = (
    select year(created_at)
    from users
    where name = '홍길동'
);
```

```text
+-----------+---------------------+
| name      | created_at          |
+-----------+---------------------+
| 홍길동    | 2024-01-15 09:00:00 |
| 김철수    | 2024-01-16 14:20:00 |
| 이영희    | 2024-01-17 18:00:00 |
| 박민수    | 2024-02-01 11:00:00 |
| ...       | ...                 |
+-----------+---------------------+
120 rows in set (0.00 sec)
```

```mermaid
flowchart LR
    A["안쪽 쿼리<br/>홍길동의 가입 년도"] -->|"2024 (값 1개)"| B["바깥 쿼리<br/>where year(created_at) = 2024"]
    B --> C["결과 120행"]
```

괄호 안의 쿼리가 먼저 값 `2024` 하나를 만들고, 바깥 쿼리는 그 값을 일반 상수처럼 쓴다. 공식 문서는 이렇게 값 하나를 돌려주는 서브쿼리를 **스칼라 서브쿼리**라고 부른다([MySQL 8.0 Reference - The Subquery as Scalar Operand](https://dev.mysql.com/doc/refman/8.0/en/scalar-subqueries.html)).

### 2. 120행은 정답이었다

**1번 질문 확인**: 조건이 안 먹힌 건지 확인하려고 가입 년도별로 사용자를 세어봤다. (확인용 **예시 코드**)

```sql
select year(created_at) y, count(*) from users group by y;
```

```text
+------+----------+
| y    | count(*) |
+------+----------+
| 2024 |      120 |
+------+----------+
1 row in set (0.00 sec)
```

실습 데이터의 사용자 120명이 **전부 2024년에 가입**했다. 쿼리는 맞았고, 데이터가 조건을 거를 수 없는 모양이었다. 결과가 이상할 때 쿼리를 의심하기 전에 `group by`로 데이터 분포부터 보는 습관이 필요하다는 걸 배웠다.

### 3. 서브쿼리가 여러 행을 돌려주면

**2번 질문 확인**: '홍길동'은 한 명이라 값이 하나였다. '김'씨로 바꾸면 여러 명이 나온다. (확인용 **예시 코드**)

```sql
select name from users
where year(created_at) = (
    select year(created_at) from users where name like '김%'
);
```

```text
ERROR 1242 (21000): Subquery returns more than 1 row
```

`=`는 값 하나와 비교하는 연산자인데 서브쿼리가 여러 행을 돌려줘서 에러가 났다. 여러 값 중 하나와 같은지 보려면 [3편]({% post_url 2026-10-12-mysql-where-condition-operators %})에서 배운 `IN`을 쓴다.

```sql
select name, created_at from users
where year(created_at) in (
    select year(created_at) from users where name like '김%'
);
```

```text
+-----------+---------------------+
| name      | created_at          |
+-----------+---------------------+
| 홍길동    | 2024-01-15 09:00:00 |
| 김철수    | 2024-01-16 14:20:00 |
| 이영희    | 2024-01-17 18:00:00 |
| ...       | ...                 |
+-----------+---------------------+
```

반대로 서브쿼리 결과가 **0행**이면 에러가 아니라 `NULL`이 된다. 없는 이름으로 실행하면 `where year(created_at) = NULL`이 되고, NULL과의 비교는 UNKNOWN이라 결과가 0행이다.

| 서브쿼리 결과 | `=` 사용 시 | 대처 |
|---------------|-------------|------|
| 1행 1열 | 정상 동작 | — |
| 여러 행 | ERROR 1242 | `IN`, `ANY`, `ALL` 사용 |
| 0행 | NULL과 비교 → 결과 0행 | 에러가 안 나므로 결과를 확인해야 함 |

### 4. FROM 절 서브쿼리 — 가상 테이블

수업의 두 번째 예제는 서브쿼리를 `FROM`(정확히는 `JOIN`) 자리에 넣었다.

```sql
select
    u.name as '교육자 이름',
    aut.course_count as '교육자 개설 과정 수'
from users as u
join (
    select author_id, count(*) as course_count
    from courses
    group by author_id
) as aut on u.user_id = aut.author_id
order by `교육자 개설 과정 수` desc;
```

```text
+------------------+-----------------------------+
| 교육자 이름      | 교육자 개설 과정 수         |
+------------------+-----------------------------+
| 홍길동           |                           4 |
| 김철수           |                           3 |
| 박민수           |                           3 |
| 최수진           |                           3 |
| 이영희           |                           2 |
| 정준호           |                           2 |
| ...              | ...                         |
+------------------+-----------------------------+
15 rows in set (0.00 sec)
```

[5편]({% post_url 2026-10-12-mysql-aggregate-group-by-having %})에서 `group by author_id`의 결과는 번호뿐이었고, [6편]({% post_url 2026-10-12-mysql-join-inner-left-right %})에서 JOIN으로 이름을 붙이는 법을 배웠다. 이 쿼리는 그 둘을 합쳤다. 안쪽 쿼리의 결과(`author_id`, `course_count` 15행)를 `aut`라는 테이블처럼 쓰고 `users`와 조인한다. 공식 문서는 이것을 **파생 테이블(derived table)**이라 부른다([MySQL 8.0 Reference - Derived Tables](https://dev.mysql.com/doc/refman/8.0/en/derived-tables.html)).

`as aut`를 빼면 어떻게 될까? (확인용 **예시 코드**)

```sql
select * from (select author_id, count(*) as course_count from courses group by author_id);
```

```text
ERROR 1248 (42000): Every derived table must have its own alias
```

파생 테이블에는 **별칭이 필수**다. 이름이 있어야 바깥에서 `aut.course_count`처럼 가리킬 수 있기 때문이다.

### 5. 같은 결과를 내는 다른 방법

**3번 질문 확인**: 같은 결과를 세 가지 방법으로 만들어봤다. (확인용 **예시 코드**)

```sql
-- (1) JOIN 후 GROUP BY
select u.name, count(*) cnt
from users u join courses c on c.author_id = u.user_id
group by u.user_id, u.name
order by cnt desc;

-- (2) WITH (CTE)
with aut as (
    select author_id, count(*) as course_count
    from courses group by author_id
)
select u.name, aut.course_count
from users u join aut on u.user_id = aut.author_id
order by aut.course_count desc;
```

세 방법 모두 상위가 `홍길동 4, 김철수 3, 박민수 3`으로 같았다.

| 방법 | 특징 |
|------|------|
| FROM 서브쿼리 (수업) | "먼저 집계하고 나중에 붙인다"는 순서가 코드에 드러난다 |
| JOIN 후 GROUP BY | 가장 짧다. 대신 이름(`u.name`)도 `group by`에 넣어야 한다 |
| `WITH` (CTE) | 서브쿼리에 이름을 붙여 위로 뺀 형태. 길어질수록 읽기 쉽다 |

CTE는 MySQL 8.0부터 지원하는 문법이다([MySQL 8.0 Reference - WITH (Common Table Expressions)](https://dev.mysql.com/doc/refman/8.0/en/with.html)).

### 6. 비어 있던 1번 문제 풀기

**4번 질문 확인**: "SQL 기초 강의 수강생의 평균 점수"를 구하려면 점수 테이블 `grades`를 봐야 한다.

```sql
desc grades;
```

```text
+--------------------+--------------+------+-----+---------+----------------+
| Field              | Type         | Null | Key | Default | Extra          |
+--------------------+--------------+------+-----+---------+----------------+
| grade_id           | bigint       | NO   | PRI | NULL    | auto_increment |
| user_id            | bigint       | NO   | MUL | NULL    |                |
| grader_id          | bigint       | NO   | MUL | NULL    |                |
| gradable_item_type | varchar(50)  | NO   |     | NULL    |                |
| gradable_item_id   | bigint       | NO   |     | NULL    |                |
| score              | decimal(5,2) | NO   |     | NULL    |                |
| feedback           | text         | YES  |     | NULL    |                |
+--------------------+--------------+------+-----+---------+----------------+
```

`grades`에는 `course_id`가 없다. 점수는 과제(`assignment`)나 퀴즈(`quiz`)에 매겨지고, 그 과제·퀴즈가 어느 강좌 것인지는 `모듈 → 섹션 → 강좌` 순서로 테이블 3개를 더 타고 올라가야 알 수 있다. 8건의 점수를 모두 따라가 보니 **퀴즈 점수 3건이 전부 3번 강좌 'SQL 기본부터 활용까지'** 것이었다. 그래서 이 데이터에서는 퀴즈 점수만 보면 된다. (확인용 **예시 코드**)

```sql
select u.name, g.score
from grades g
join users u on u.user_id = g.user_id
where g.gradable_item_type = 'quiz'
  and g.score > (
      select avg(score) from grades where gradable_item_type = 'quiz'
  );
```

```text
+-----------+--------+
| name      | score  |
+-----------+--------+
| 신가은    | 100.00 |
| 신가은    | 100.00 |
+-----------+--------+
2 rows in set (0.00 sec)
```

평균은 `(100 + 66.67 + 100) / 3 = 88.89`이고, 이보다 높은 점수는 신가은 학생의 퀴즈 2개였다. 다만 `gradable_item_type = 'quiz'`는 **이 실습 데이터에서만 맞는 지름길**이다. 다른 강좌에 퀴즈가 추가되면 틀린 답이 나오므로, 정석은 모듈·섹션 테이블까지 조인해서 `course_id = 3`으로 거르는 것이다.

## 결과 (Result)

| 확인한 내용 | 결과 |
|-------------|------|
| 홍길동과 같은 해 가입자 | 120행 — 전원 2024년 가입이라 정답 |
| 서브쿼리가 여러 행 + `=` | ERROR 1242 → `IN`으로 해결 |
| 서브쿼리가 0행 | 에러 없이 결과 0행 |
| FROM 서브쿼리 별칭 생략 | ERROR 1248 |
| 같은 결과 3가지 방법 | 서브쿼리·JOIN+GROUP BY·CTE 모두 상위 4·3·3 |
| SQL 기초 평균(88.89) 초과 | 신가은 100점 2건 |

배운 점은 다음과 같다.

- 서브쿼리는 "모르는 값을 먼저 구하는 쿼리"를 괄호에 넣는 것이다. 수업처럼 단계를 나눠 따로 실행해본 뒤 합치면 실수가 줄어든다.
- 결과가 이상하면 쿼리보다 데이터 분포를 먼저 본다. 120행은 버그가 아니었다.
- `=` 뒤의 서브쿼리는 반드시 값 하나여야 한다. 여러 개일 수 있으면 `IN`을 쓴다.

## 더 학습하면 좋은 개념

- **상관 서브쿼리(Correlated Subquery)** — 바깥 쿼리의 행마다 안쪽 쿼리가 다시 실행되는 형태. "각 강좌별 평균보다 높은 점수"처럼 그룹마다 기준이 다를 때 필요하다.
- **`EXISTS` / `NOT EXISTS`** — "수강생이 한 명이라도 있는 강좌"처럼 존재 여부만 볼 때 `IN`보다 의도가 분명하다.
- **SELECT 절의 스칼라 서브쿼리** — 열 자리에 서브쿼리를 넣어 행마다 값을 계산하는 방법과, 이것이 JOIN보다 느려질 수 있는 이유.
- **CTE와 재귀 CTE** — `WITH RECURSIVE`로 카테고리 계층처럼 자기 자신을 참조하는 데이터를 펼칠 수 있다.
- **함수를 씌운 열과 인덱스** — `year(created_at)`처럼 열에 함수를 씌우면 인덱스를 쓰지 못한다. [12편]({% post_url 2026-10-13-mysql-index-explain %})과 연결된다.

## 참고 자료

- [MySQL 8.0 Reference - Subqueries](https://dev.mysql.com/doc/refman/8.0/en/subqueries.html)
- [MySQL 8.0 Reference - The Subquery as Scalar Operand](https://dev.mysql.com/doc/refman/8.0/en/scalar-subqueries.html)
- [MySQL 8.0 Reference - Subqueries with ANY, IN, or SOME](https://dev.mysql.com/doc/refman/8.0/en/any-in-some-subqueries.html)
- [MySQL 8.0 Reference - Derived Tables](https://dev.mysql.com/doc/refman/8.0/en/derived-tables.html)
- [MySQL 8.0 Reference - WITH (Common Table Expressions)](https://dev.mysql.com/doc/refman/8.0/en/with.html)

---

**요약**
1. 서브쿼리는 모르는 값을 먼저 구하는 쿼리를 괄호로 감싸 바깥 쿼리에 넣는 것이고, 홍길동 문제의 120행은 사용자 전원이 2024년 가입이라 나온 정답이었다.
2. `=` 뒤 서브쿼리가 여러 행이면 ERROR 1242가 나므로 `IN`을 쓰고, 0행이면 에러 없이 결과가 비므로 직접 확인해야 한다.
3. FROM 절 서브쿼리(파생 테이블)는 별칭이 필수(ERROR 1248)이며, JOIN+GROUP BY나 CTE(`WITH`)로도 같은 결과를 만들 수 있다.
