---
layout: post
title: "map.get(12)는 banana인데 map.get(\"12\")는 null — HashMap의 키 비교와 덮어쓰기, Properties"
date: 2026-10-08 09:30:00 +0900
categories: [Java]
tags: [java, collection, map, hashmap, 자바기초]
---

## 들어가며 (Situation)

컬렉션 수업의 마지막은 `b_collection/c_map` 패키지였다. [ArrayList]({% post_url 2026-10-07-java-arraylist-bookdto %}), [HashSet·TreeSet]({% post_url 2026-10-07-java-hashset-treeset-lotto %})에 이어 세 번째 컬렉션인 `Map`이다.

| 파일 | 내용 |
|------|------|
| `Application01.java` | 타입을 정하지 않은 `Map`과 `Map<String, String>` |
| `Application02.java` | DB 접속 정보를 담는 `Properties` |

수업 주석의 정리는 이렇다.

> 1. Key-Value : 키-값 한 쌍으로 데이터를 저장한다.
> 2. Key 는 내부적으로 Set 방식으로 구성이 되어있다.

```java
Map map = new HashMap();
map.put("one", new Date());
map.put(12, "apple");
// key 는 중복되게 되면 나중에 작성한 값으로 덮여씌워지게 된다.
map.put(12, "banana");

System.out.println("banana 값 출력하기 : " + map.get(12));
```

## 문제 상황 (Task)

실행 결과는 주석대로였다.

```text
map = {one=Tue Oct 27 14:28:13 KST 2026, 12=banana}
banana 값 출력하기 : banana
map2 = {one=java, two=javascript, three=python}
map2 = {one=java, two=javascript}
```

그런데 첫 번째 `map`은 키 자리에 `"one"`(문자열)과 `12`(숫자)가 섞여 있다. 출력에서는 `12`가 문자처럼 보인다. 그래서 궁금했다.

1. 키 `12`와 `"12"`는 같은 키일까?
2. 덮어쓸 때 이전 값은 그냥 사라질까?
3. 없는 키를 꺼내면? 값이 `null`인 것과 키가 없는 것은 구분될까?
4. `HashSet`처럼 넣은 순서를 섞을까?

## 해결 과정 (Action)

### 1. 키 `12`, `"12"`, `12L`은 모두 다르다

```java
System.out.println("get(12)=" + map.get(12) + ", get(\"12\")=" + map.get("12") + ", get(12L)=" + map.get(12L));
```

```text
get(12)=banana, get("12")=null, get(12L)=null
```

출력에서는 똑같이 `12`로 보이지만 `map.get("12")`는 `null`이었다. `put(12, ...)`의 `12`는 **자동으로 `Integer` 객체가 된다**(오토박싱). `"12"`는 `String`, `12L`은 `Long`이다. `HashMap`은 키를 `hashCode()`와 `equals()`로 비교한다. `Integer.equals()`는 상대가 `Integer`일 때만 같다고 판단한다. 숫자 값이 같아도 타입이 다르면 다른 키다.

타입을 정하지 않은 `Map`(raw type)에서는 이런 실수를 컴파일러가 잡아 주지 못한다. 실제로 컴파일하면 경고가 나온다.

```text
warning: [unchecked] unchecked call to put(K,V) as a member of the raw type Map
        map.put("one", new Date());
```

`Map<String, String>`처럼 타입을 정하면 `map.get(12)`를 쓸 때부터 실수가 드러난다. 정확히는 `get(Object)`라서 컴파일은 되지만, IntelliJ가 "의심스러운 호출"로 표시한다. 그리고 `put(12, ...)`은 아예 컴파일되지 않는다. 수업 주석의 "Map 의 Key 값은 암묵적으로 String 타입으로 하는 것이 일반적이다"도 같은 맥락이다. 키 타입을 하나로 정해 두는 것이다.

### 2. `put`과 `remove`는 이전 값을 돌려준다

```java
Map<Integer, String> m = new HashMap<>();
System.out.println("첫 put=" + m.put(12, "apple") + ", 덮어쓰기 put=" + m.put(12, "banana") + ", 결과=" + m);
```

```text
첫 put=null, 덮어쓰기 put=apple, 결과={12=banana}
```

| 메서드 | 반환값 |
|--------|--------|
| 새 키로 `put` | `null` |
| 있는 키로 `put` (덮어쓰기) | **이전 값** (`apple`) |
| `remove("three")` | 지운 값 (`python`) |
| 없는 키로 `remove` | `null` |

덮어쓰기는 에러 없이 조용히 일어난다. 대신 반환값으로 이전 값을 알려 준다. [HashSet의 `add()`가 `false`로 중복을 알려 준 것]({% post_url 2026-10-07-java-hashset-treeset-lotto %})과 비슷하다. 실수로 덮어쓰는 것을 막고 싶다면 `putIfAbsent()`를 쓴다.

### 3. `null`이 두 가지 뜻을 가진다

```text
없는 키 get=null, getOrDefault=-
값이 null: get("one")=null, containsKey=true
```

`HashMap`은 값으로 `null`을 넣을 수 있다. 그래서 `get()`이 `null`을 돌려주면 두 경우를 구분할 수 없다.

| 상황 | `get(key)` | `containsKey(key)` |
|------|------------|---------------------|
| 키가 없음 | `null` | `false` |
| 키는 있고 값이 `null` | `null` | `true` |

`null`을 받아서 바로 메서드를 호출하면 [NullPointerException]({% post_url 2026-10-08-java-exception-try-catch-finally-throw %})이 난다. 없는 키에 대비하려면 `getOrDefault(key, 기본값)`이나 `containsKey()`를 쓴다.

### 4. 순서: HashMap, LinkedHashMap, TreeMap

같은 다섯 단어를 세 가지 `Map`에 넣었다. (확인용 **예시 코드**)

```text
HashMap       {banana=6, date=4, apple=5, cherry=6, elderberry=10}
LinkedHashMap {banana=6, apple=5, cherry=6, date=4, elderberry=10}
TreeMap       {apple=5, banana=6, cherry=6, date=4, elderberry=10}
```

| 구현체 | 순서 | 짝이 되는 Set |
|--------|------|---------------|
| `HashMap` | 해시값 순서 (넣은 순서와 무관) | `HashSet` |
| `LinkedHashMap` | 넣은 순서 | `LinkedHashSet` |
| `TreeMap` | 키 정렬 순서 | `TreeSet` |

[HashSet 글]({% post_url 2026-10-07-java-hashset-treeset-lotto %})에서 본 순서 규칙과 똑같다. 실제로 `HashSet`은 내부에 `HashMap`을 갖고 있고, 원소를 그 **키**로 저장한다. 수업 주석의 "Key 는 내부적으로 Set 방식"이라는 말이 이 관계를 가리킨다. 키는 중복될 수 없고, `keySet()`은 실제로 `Set`을 돌려준다.

수업의 `map2` 출력이 `{one, two, three}` 순서로 나온 것은 우연이다. 키 세 개의 해시값이 마침 그 순서였을 뿐, `HashMap`은 순서를 보장하지 않는다.

### 5. Map이 잘하는 일: 개수 세기

키로 바로 찾을 수 있다는 장점을 살린 예다. (**예시 코드**)

```java
Map<String, Integer> cnt = new HashMap<>();
for (String w : "java python java sql java python".split(" ")) {
    cnt.merge(w, 1, Integer::sum);   // 없으면 1, 있으면 기존 값 + 1
}
```

```text
단어 세기 {python=2, java=3, sql=1}
```

리스트였다면 단어마다 처음부터 끝까지 찾아야 하지만, `HashMap`은 키의 해시값으로 칸을 바로 찾는다. `HashMap` 문서는 `get`, `put`이 **상수 시간**에 동작한다고 설명한다.

### 6. Properties는 문자열 전용 Map

```java
Properties prop = new Properties();
prop.setProperty("driver","cj.jdbc.driver.mysql");
prop.setProperty("url", "jdbc:mysql://localhost/menudb");
prop.setProperty("username", "wanted");
prop.setProperty("password", "wanted");
```

```text
prop = {password=wanted, driver=cj.jdbc.driver.mysql, url=jdbc:mysql://localhost/menudb, username=wanted}
```

`Properties`도 넣은 순서를 지키지 않았다. 내부가 해시 테이블이기 때문이다. 수업 주석은 "Key-Value 모두 String 문자열"이라고 했는데, 확인해 보니 이 규칙은 **`setProperty()`를 쓸 때만** 지켜진다.

```text
getProperty("n")=null, get("n")=1
```

`Properties`는 `Hashtable<Object, Object>`를 상속해서 `put("n", 1)`처럼 숫자를 넣을 수도 있다. 하지만 그렇게 넣은 값은 `getProperty()`로 꺼내지지 않는다(`null`). 공식 문서도 `put`을 쓰지 말고 `setProperty`를 쓰라고 권한다. 이 `Properties`는 이후 [JDBC 수업에서 설정 파일을 읽을 때]({% post_url 2026-10-19-java-jdbc-classpath-config-try-with-resources %}) 그대로 쓰였다.

참고로 수업 예제의 드라이버 이름 `cj.jdbc.driver.mysql`은 설명용 값이다. 실제 MySQL 드라이버 클래스 이름은 `com.mysql.cj.jdbc.Driver`다.

## 결과 (Result)

| 확인 항목 | 결과 |
|-----------|------|
| `get(12)` / `get("12")` / `get(12L)` | `banana` / `null` / `null` (타입이 다르면 다른 키) |
| 덮어쓰기 `put` 반환값 | 이전 값 `apple` |
| 값이 `null`인 키 | `get = null`, `containsKey = true` |
| 순서 | `HashMap` 해시 순 / `LinkedHashMap` 넣은 순 / `TreeMap` 정렬 순 |
| `Properties.put("n", 1)` | `getProperty("n") = null` |
| 환경 | Java 21 (Temurin 21.0.12) |

배운 점은 다음과 같다.

- `Map`의 키는 `equals()`로 비교되므로 **값뿐 아니라 타입까지** 같아야 같은 키다. 제네릭으로 키 타입을 고정하면 이런 실수를 줄일 수 있다.
- `get()`의 `null`은 "키 없음"과 "값이 null"을 구분하지 못한다. `getOrDefault`나 `containsKey`로 의도를 분명히 한다.

## 더 학습하면 좋은 개념

- **`equals()`와 `hashCode()` 규약** — 직접 만든 클래스를 키로 쓸 때 둘 다 재정의해야 하는 이유.
- **해시 충돌과 버킷** — 다른 키가 같은 칸에 들어갈 때 `HashMap`이 처리하는 방식(연결 리스트, 트리).
- **`entrySet()` 순회와 `Map.of()`** — 키와 값을 함께 순회하는 방법과 수정할 수 없는 Map 만들기.
- **`ConcurrentHashMap`** — 여러 스레드가 함께 쓰는 Map. Spring AOP 실습의 인메모리 저장소(`HashMap`)를 실제 서버에서 쓰면 생길 문제와 이어진다.

## 참고 자료

- [Java SE 21 API - Map](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/Map.html)
- [Java SE 21 API - HashMap](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/HashMap.html)
- [Java SE 21 API - Integer.equals()](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/Integer.html#equals(java.lang.Object))
- [Java SE 21 API - Properties](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/Properties.html)
- [The Java Tutorials - The Map Interface](https://docs.oracle.com/javase/tutorial/collections/interfaces/map.html)

---

**요약**
1. 출력은 똑같이 `12`로 보여도 `Integer` 12, `String` "12", `Long` 12L은 `equals()`가 달라서, `get("12")`와 `get(12L)`은 `null`이었다.
2. 같은 키로 `put`하면 조용히 덮어쓰고 이전 값을 반환한다. `get()`의 `null`은 "키 없음"과 "값이 null"을 구분하지 못한다.
3. `HashMap`·`LinkedHashMap`·`TreeMap`의 순서는 `Set` 구현체와 같았다. `Properties`는 `setProperty`로 넣은 문자열만 `getProperty`로 꺼낼 수 있었다.
