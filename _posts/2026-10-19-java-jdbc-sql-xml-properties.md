---
layout: post
title: "길어지는 SQL 문자열을 XML 파일로 분리하기 — Properties.loadFromXML 사용기"
date: 2026-10-19 12:00:00 +0900
categories: [Java]
tags: [java, jdbc, xml, properties, file-io]
mermaid: true
---

## 들어가며 (Situation)

`b_preparedstatements`의 세 번째 실습(`Application03.java`)은 자바 파일에 직접 쓰던 SQL을 **XML 파일로 빼내는** 내용이었다. 수업 주석이 이유를 설명한다.

> SQL 구문의 경우 한 문장으로 끝나면 문자열로 작성해도 큰 무리가 없다. 다만, Join 을 하거나 Where 조건절이 길어지게 된다면 문자열 합치기를 통한 SQL 구문 작성이 굉장히 복잡해진다.

앞의 실습에서는 `"SELECT MENU_CODE, MENU_PRICE, ORDERABLE_STATUS FROM TBL_MENU WHERE MENU_NAME = ?"`처럼 한 줄로 썼다. 이 정도면 괜찮지만, 여러 줄로 나눠 쓰면 `"... " + "FROM ..." + " WHERE ..."` 식으로 이어 붙이다가 공백 하나를 빠뜨려 오류가 나기 쉽다.

## 문제 상황 (Task)

목표는 SQL을 자바 코드에서 분리하되, 새 라이브러리 없이 JDK만으로 해결하는 것이었다. 확인할 것은 다음과 같다.

1. XML 파일의 SQL을 어떻게 읽어 `PreparedStatement`에 넘기는가?
2. 줄바꿈과 들여쓰기가 들어간 SQL이 그대로 실행되는가?
3. 이 방식의 약점은 무엇인가? (키를 잘못 쓰면? 경로가 틀리면?)

## 해결 과정 (Action)

### 1. 선택지 비교

| 방식 | 장점 | 단점 |
|------|------|------|
| 자바 문자열 | 파일 하나로 끝, 추적 쉬움 | 길어지면 `+` 연결과 공백 실수 |
| `.properties` 파일 | 간단, 키=값 | 여러 줄 SQL을 쓰려면 `\` 줄바꿈 이스케이프 필요 |
| **XML(`Properties` 형식)** | 여러 줄 SQL을 그대로 작성, JDK 내장 | 파일이 하나 더 생김, 키 오타는 실행 중에 발견 |
| MyBatis 등 외부 도구 | 기능 풍부 | 의존성 추가 필요 |

수업은 XML을 골랐다. `Properties`가 `loadFromXML()`을 제공해서 **JDK만으로** 쓸 수 있기 때문이다. 이 선택의 근거는 수업 주석("XML 파일을 이용한 File I/O 로 SQL 구문 따로 분리하기")과 아래 확인에서 나왔다.

### 2. `menu-query.xml` 작성

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
<properties>

    <entry key="findMenuByMenuName">
        SELECT
            *
        FROM
            TBL_MENU
        WHERE
            MENU_NAME = ?
    </entry>

    <entry key="modifyMenuPriceByMenuName">
        UPDATE TBL_MENU
        SET MENU_PRICE = ?
        WHERE MENU_NAME = ?
    </entry>

</properties>
```

수업 주석에 따르면 DOCTYPE을 `properties`로 선언하면 이 파일을 `Properties`처럼 쓸 수 있다. 각 `<entry>`의 `key`가 이름이고, 태그 안의 텍스트가 값(SQL)이다. 키 이름은 `find...`, `modify...` 처럼 **무슨 일을 하는 쿼리인지** 드러나게 지었다.

### 3. 자바에서 읽기

```java
Properties prop = new Properties();

prop.loadFromXML(
        new FileInputStream("src/main/java/com/wanted/b_preparedstatements/menu-query.xml")
);

String query = prop.getProperty("findMenuByMenuName");   // 읽은 파일에서 어떤 Key 를 가져올지

pstmt = con.prepareStatement(query);
pstmt.setString(1, menuName);
rset = pstmt.executeQuery();
```

```mermaid
flowchart LR
    X["menu-query.xml<br/>key → SQL"] -->|"loadFromXML"| P["Properties 객체"]
    P -->|"getProperty(key)"| Q["SQL 문자열"]
    Q -->|"prepareStatement"| PS["PreparedStatement"]
    PS -->|"setString + executeQuery"| R["ResultSet"]
    R -->|"한 행"| D["MenuDTO"]
```

조회된 한 행은 `MenuDTO`에 담는다.

```java
MenuDTO menu = null;
if (rset.next()) {
    menu = new MenuDTO();
    menu.setMenuCode(rset.getInt("MENU_CODE"));
    menu.setMenuName(rset.getString("MENU_NAME"));
    menu.setMenuPrice(rset.getInt("MENU_PRICE"));
    menu.setCategoryCode(rset.getInt("CATEGORY_CODE"));
    menu.setOrderableStatus(rset.getString("ORDERABLE_STATUS").charAt(0));
}
System.out.println("menu = " + menu);
```

`Application02`의 `while`과 다르게 `if (rset.next())`를 쓴 점이 눈에 띄었다. 메뉴 이름으로 찾는 결과는 한 행이라고 가정했기 때문이다. `CHAR(1)` 컬럼인 `ORDERABLE_STATUS`는 `getString(...).charAt(0)`으로 `char`에 담았다.

`열무김치라떼`로 실행한 결과다.

```text
메뉴 이름 : menu = MenuDTO{menuCode=1, menuName='열무김치라떼', menuPrice=4500, categoryCode=8, orderableStatus=Y}
```

없는 메뉴(`없는메뉴`)를 넣으면 `menu = null`이 출력됐다. `menu`를 `null`로 초기화했기 때문에 에러 없이 "없다"는 상태를 표현한다.

### 4. 여러 줄 SQL이 그대로 실행되는 이유

XML 안의 SQL은 들여쓰기와 줄바꿈이 모두 값에 들어 있다. 값을 출력해 확인했다. (확인용 **예시 코드**)

```text
raw = [
        SELECT
            *
        FROM
            TBL_MENU
        WHERE
            MENU_NAME = ?
    ]
```

앞뒤로 줄바꿈과 공백이 붙어 있다. 그래도 `prepareStatement(query)`는 문제없이 실행됐다. SQL 문법에서 줄바꿈·공백은 토큰 사이의 구분자로만 쓰이기 때문이다. 필요하면 `trim()`으로 앞뒤 공백을 지울 수 있고, chap03의 `QueryUtil`이 실제로 그렇게 한다([다음 글]({% post_url 2026-10-19-java-jdbc-classpath-config-try-with-resources %}) 참고).

### 5. 약점 확인 — 키 오타와 경로

**키를 잘못 쓰면?** 존재하지 않는 키를 넣어 봤다. (확인용 **예시 코드**)

```text
getProperty("findMenuByName")   -> null
prepareStatement(null)          -> SQLException
null.trim()                     -> NullPointerException
```

컴파일러가 잡아 주지 않고 **실행해야** 발견된다. SQL이 자바 코드 밖으로 나온 대가다.

**경로가 틀리면?** `Application03`은 `"src/main/java/com/wanted/b_preparedstatements/menu-query.xml"` 같은 **상대 경로**로 파일을 연다. 이 상대 경로는 프로그램을 실행한 위치(현재 작업 디렉토리) 기준이다. 프로젝트 폴더가 아닌 다른 곳에서 실행하니 이렇게 실패했다.

```text
Exception in thread "main" java.lang.RuntimeException: java.io.FileNotFoundException:
src\main\java\com\wanted\config\jdbc-config.properties (지정된 경로를 찾을 수 없습니다)
```

이 오류는 XML보다 먼저 연결 설정 파일(`jdbc-config.properties`)에서 났지만, XML을 읽는 줄도 같은 상대 경로 방식이라 똑같이 실패한다. IDE에서는 프로젝트 루트에서 실행하기 때문에 지금까지 문제가 보이지 않았다. 이 약점은 chap03에서 **클래스패스 기반 읽기**로 해결된다.

### 6. 예외 처리가 길어진 이유

`Application03`의 `catch`가 네 개(`SQLException`, `FileNotFoundException`, `InvalidPropertiesFormatException`, `IOException`)로 늘었다. `loadFromXML()`이 `IOException`과 `InvalidPropertiesFormatException`을 던지기 때문이다. 여기서는 모두 `RuntimeException`으로 감싸 던지는 최소 처리만 했다. `FileNotFoundException`과 `InvalidPropertiesFormatException`은 모두 `IOException`의 하위 클래스라 `IOException` 하나로 묶을 수도 있다.

## 결과 (Result)

| 확인 항목 | 결과 |
|-----------|------|
| XML에서 SQL 읽어 실행 | `열무김치라떼` 조회 성공, `MenuDTO` 한 건 생성 |
| 없는 메뉴 입력 | `menu = null` (에러 없음) |
| 줄바꿈·들여쓰기가 있는 SQL | 그대로 실행됨 |
| 존재하지 않는 키 | `null` → 실행 중 예외 |
| 다른 폴더에서 실행 | 상대 경로라서 `FileNotFoundException` |

배운 점은 다음과 같다.

- 긴 SQL은 XML로 분리하면 읽기 쉽고, `Properties.loadFromXML()`이라 JDK만으로 가능하다.
- 대신 **SQL의 키 이름과 파일 경로가 문자열 약속**이 되어, 오타가 실행 시점에야 드러난다.
- 상대 경로는 IDE에서는 잘 되지만 실행 위치가 바뀌면 깨진다. 이 때문에 chap03에서 방식을 바꿨다.

## 더 학습하면 좋은 개념

- **클래스패스(Classpath)와 `getResourceAsStream`** — 실행 위치와 무관하게 리소스를 읽는 방법. 오늘 상대 경로의 약점을 해결한다.
- **XML과 DTD** — `DOCTYPE properties SYSTEM ...`이 무엇을 정의하고 `Properties`가 이를 어떻게 해석하는지 이해하면 파일 형식 오류의 원인을 찾기 쉽다.
- **MyBatis의 SQL 매퍼 XML** — 오늘 방식의 발전형이다. 자바 코드와 SQL을 분리한다는 같은 발상을 훨씬 체계적으로 쓴다.
- **try-with-resources** — `FileInputStream`을 닫는 코드를 오늘은 생략했다. 자원 해제를 자동화하는 문법이다.
- **텍스트 블록(`"""`)** — Java 15부터 표준인 여러 줄 문자열. SQL을 파일로 분리하지 않고 코드 안에서 읽기 좋게 쓰는 대안이다.

## 참고 자료

- [Java SE 21 API - Properties (loadFromXML)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/Properties.html)
- [Java SE 21 API - PreparedStatement](https://docs.oracle.com/en/java/javase/21/docs/api/java.sql/java/sql/PreparedStatement.html)

---

**요약**
1. 길어지는 SQL은 `menu-query.xml`에 키별로 분리하고 `Properties.loadFromXML()`로 읽어 `prepareStatement()`에 넘기면 JDK만으로 처리할 수 있다.
2. XML 값의 줄바꿈·들여쓰기는 SQL 실행에 문제가 없지만, 키 오타(`null`)와 상대 경로 문제는 실행해야 발견된다.
3. 상대 경로로 파일을 읽으면 실행 위치에 따라 실패하므로 클래스패스 방식으로 바꾸는 것이 다음 과제다.
