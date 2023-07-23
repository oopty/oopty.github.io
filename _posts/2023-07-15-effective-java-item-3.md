---
title: Effective Java 정리 Item3
date: 2023-07-23 +09:00
categories: [java]
tags: [effectivejava, java]
---
item3는 싱글톤 객체를 만드는 방법에 대해 설명합니다. 싱글톤으로 만들면 그 객체를 mocking하기 어려워진다고 합니다. 시스템에 하나의 객체밖에 없도록 설계된 클래스라 mock 객체를 만들기 어렵기 때문입니다. 싱글톤 객체를 만드는 방법은 세가지 입니다. 필드 정적 멤버, 메서드 정적 멤버와 열거타입입니다.

### 필드 정적 멤버
```java
public class ElvisV1 {
    public static final ElvisV1 INSTANCE = new ElvisV1();

    public ElvisV1() {
    }
}
```
위 방식의 장점은 첫번째로 간단하다는 것입니다. 따로 접근을 위한 메서드가 필요하지 않습니다. 두번째 장점으로 Reflection을 이용하지 않는다면 싱글톤 객체라는 제약사항을 깨버릴수 없습니다. 처음 시스템이 실행될때 ElvisV1을 초기화 시킬수 있습니다.

### 메서드 정적 멤버
```java
public class ElvisV2 {
    private static final ElvisV2 INSTANCE = new ElvisV2();

    public ElvisV2() {
    }

    public static ElvisV2 getInstance() {
        return INSTANCE;
    }
}
```
위 방식은 첫번째보다 더 장점이 많습니다. 첫번째로 설계를 변화시킬 여지가 많다는 것입니다. 추후에 싱글톤 객체 제약사항을 없애고 싶으면 그렇게 바꿔도 클라이언트 코드에는 변경사항이 없습니다. 두번째로 제네릭 싱글톤 펙토리로 만들수도 있습니다. 아래는 List 클래스에서 제네릭 싱긅톤 팩토리가 쓰인 예시입니다.
```java
static <E> List<E> of() {
    return (List<E>) ImmutableCollections.EMPTY_LIST;
}
```
세번째로는 메서드 참조를 공급자(Supplier)로 사용할 수 있습니다. ElvisV2::getInstance를 Supplier<Elvis>에 사용할 수 있습니다.

### 열거 타입
public enum Elvis {
    INSTANCE;
}
훨씬 더 간결하고, Reflection 공격과 역직렬화 공격에도 안전합니다. 대부분의 상황에서는 원소가 하나뿐인 열거타입이 싱글톤을 만드는 가장 좋은 방법입니다. 단, 만들려는 싱글톤이 enum 외의 클래스를 상속해야 한다면 이 방법은 사용할 수 없습니다.
