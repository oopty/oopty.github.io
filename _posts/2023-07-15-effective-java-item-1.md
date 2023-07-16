---
title: Effective Java 정리 Item1
date: 2023-07-15 +09:00
categories: [java]
tags: [effectivejava, java]
---
안녕하세요. 최근에 Effective Java를 읽으면서 자바의 고급 스킬에 대해 많이 배웠는데요 이를 글로 적어 다시 복습하려고 합니다.

## item1. 생성자 대신 정적 펙토리 메서드를 고려하라
정적 펙터리 메서드는 아래와 같이 private 생성자를 통해 객체를 생성하는 방법입니다.
```java
class MyClass {
    int myField1;
    String myField2;

    private MyClass(int myField1, String myField2) {
        this.myField1 = myField1;
        this.myField2 = myField2;
    }

    public MyClass getInstance(int myField1, String myField2) {
        return new MyClass(myField1, myField2);
    }
}
public 
```

이 정적 펙터리 메서드의 장점은 5가지입니다.
## 장점
### 1. 이름을 가질 수 있다.
생성자는 이름을 짓지 못하는데 정적 펙토리 메서드는 이름을 가질 수 있습니다. 이런 이름은 대부분 from, of, valueOf, getInstance 등이 있습니다. 관례적으로 사용하는 이름이 있는데 정리하면 아래와 같습니다.
- from: 한 개의 매개변수를 받고 객체를 반환하는 형변환 메서드
- of: 여러 개의 매개변수를 받고 적합한 타입의 객체를 반환하는 집계 메서드
- valueOf: from과 of의 더 자세한 버전
- instance 혹은 getInstance: 매개 변수로 명시한 객체를 생성해 반환하지만 매개 변수가 같다고 같은 객체인지는 보장하지 않음
- create 혹은 newInstance: instance와 같지만 매번 새로운 객체가 반환됨을 보장함
- getType: instance와 같지만 속해 있는 클래스와 다른 클래스를 반환함 `Type`은 반환하려는 객체의 클래스를 나타냄
- newType: getType과 같지만 매번 다른 객체를 반환함을 보장
- type: getType과 newType의 간결한 버전

### 2. 호출할 때마다 객체를 새로 생성하지 않아도 됨
싱글톤 객체를 만들거나 각 객체의 최대 개수를 지정하고 싶을 때 생성자는 이를 해결하지 못하지만 펙토리 메서드로는 해결할 수 있습니다.

### 3. 반환 타입의 하위 타입 객체를 반환할 수 있음
상위 타입의 정적 펙토리 메서드에서 하위타입의 객체를 반환할 수 있습니다. 또한 이는 클라이언트 코드에서는 어떤 하위 타입인지 알지 못하며 이는 클라이언트 코드와 결합도를 낮춰 설계의 자유로움을 가져다 줍니다.

### 4. 입력 매개변수에 따라 매번 다른 클래스의 객체를 반환할 수 있음
3번과 비슷한 내용으로 입력 매개변수에 따라 반환되는 객체의 클래스를 다르게 할 수 있고 이는 다음 릴리즈에 변경해도 클라이언트 코드에는 영향이 없습니다. 예를들어, EnumSet 클래스의 정적 펙토리 메서드는 원소가 64개 이하면 RegularEnumSet을 그보다 초과하면 JomboEnumSet을 반환합니다. 이는 성능적인 이슈 때문이고 나중에 성능 이슈가 해결이되어 하나의 클래스로 수정해야 한다면 자유롭게 수정을 해도 클라이언트 코드를 변경하지 않아도 수정할 수 있습니다.

### 5. 정적 펙토리 메서드를 만들 시점에 반환할 클래스가 존재하지 않아도 됩니다.
이건 처음봤을때 무슨 말인지 이해가 가지 않았습니다. 뒤에서 서비스 제공자 프레임워크를 구현할 때 이 개념이 나오게 되는데 이때 책에서 자세히 설명을 해줍니다. 서비스 제공자 프레임워크(Service Provider Framework)는 3개의 핵심 컴포넌트로 이루어 집니다. 등록 API, 서비스 접근 API, 서비스 인터페이스입니다. 이렇게 3개를 구현해 놓으면 `서비스 인터페이스`를 구현한 다른 클래스를 `등록 API`로 등록하고 `서비스 접근 API`를 통해 사용할 수 있습니다. JDBC의 Connection관련 로직인 DriverManager와 Driver가 대표적인 예이고 구현된 코드를 보면 아래와 같습니다.
```java
public static void registerDriver(java.sql.Driver driver,
        DriverAction da) // 등록 API
    throws SQLException {

    /* Register the driver if it has not already been added to our list */
    if (driver != null) {
        registeredDrivers.addIfAbsent(new DriverInfo(driver, da));
    } else {
        // This is for compatibility with the original DriverManager
        throw new NullPointerException();
    }

    println("registerDriver: " + driver);

}


private static Connection getConnection(
    String url, java.util.Properties info, Class<?> caller) throws SQLException { // 서비스 접근 API

    ...

    for (DriverInfo aDriver : registeredDrivers) {
        // If the caller does not have permission to load the driver then
        // skip it.
        if (isDriverAllowed(aDriver.driver, callerCL)) {
            try {
                println("    trying " + aDriver.driver.getClass().getName());
                Connection con = aDriver.driver.connect(url, info);
                if (con != null) {
                    // Success!
                    println("getConnection returning " + aDriver.driver.getClass().getName());
                    return (con);
                }
            } catch (SQLException ex) {
                if (reason == null) {
                    reason = ex;
                }
            }

        } else {
            println("    skipping: " + aDriver.driver.getClass().getName());
        }

    }

    ...

}
```

## 단점
### 1. 상속을 하려면 public이나 protected인 생성자가 필요한데 정적 팩토리 메서드만 제공하면 만들수 가 없음
이에 대한 해결방법은 따로 Types 클래스로(ex, Collections) 정적 펙토리 메서드를 모아서 사용하면 됩니다. 하지만 이는 상속관계보다는 Compostion 관계를 사용하라는 지침을 자연스럽게 따를 수 있고 완전한 불변 객체를 만들기 위해서 지켜야하는 점이라는 것에서 큰 단점은 아닐수 있습니다.


### 2. 사용하려는 클라이언트가 정적 펙토리 메서드를 찾아야 함
이는 내가 실무에서도 겪고있는 문제이고 정적 펙토리 메서드를 찾아서 사용해야 하는 점은 불편하긴 합니다. 설계자 입장에서는 Javadoc에 명확히 설명을 적어주고 위에 설명한 정적 펙토리 메서드 이름을 관례적으로 짓는 것으로 클라이언트 프로그래머에게 도움을 줄 수 있습니다.

정적 펙토리 메서드와 public 생성자는 장단점이 있으니 적절히 섞어서 사용하되 대부분의 경우에는 정적 펙토리 메서드가 유리하니까 생성자보단 정적 펙토리 메서드를 고려하면서 사용하도록 하는 편이 좋습니다.