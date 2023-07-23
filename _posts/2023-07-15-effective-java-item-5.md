---
title: Effective Java 정리 Item5
date: 2023-07-23 +09:00
categories: [java]
tags: [effectivejava, java]
---
Item5에서는 정적 유틸리티 클래스나 싱글톤 패턴을 잘못 사용한 예에 대해 설명하면서 의존성 주입을 사용하라고 권고합니다. 예시를 든 코드는 철자 검사기인데 이 철자 검사기 클래스는 정확한 맞춤법이 적혀진 사전을 필드에 가지고 있습니다. 이를 싱글톤이나 유틸리티 클래스로 만들면 하나의 사전만 시스템에서 사용할 수 있습니다. 하지만 현실에선 영어사전도 필요하고, 한국어 사전도 필요한것 처럼 여러 사전이 필요한 상황이 있습니다. 그래서 이 경우에는 싱글톤이나 유틸리티 클래스가 적합하지 않습니다. setter 함수로 바꿔가면서 사용할 수 있지만 이렇게 되면 불변객체가 아니게 되고 멀티 쓰레드 환경에서는 사용하기 어렵습니다. 그래서 처음 객체가 생성될 때 객체를 넘겨 그 객체를 사용하게 만드는 것이 좋습니다.

```java
public class SpellCheck {
    private final Lexicon dictionary;
    public SpellCheck(Lexicon dictionary) {
        this.dictionary = Objects.requireNonNull(dictionary);
    }
}
```

이 방법을 의존 객체 주입 패턴이라고 부릅니다. 이에 대한 변형이 팩토리 메서드 패턴입니다. 아래와 같이 특정 타입의 인스턴스를 반복해서 만들어 주는 객체(Supplier<T>)를 매개변수로 받고 이를 실행한 결과를 객체로 받는 방법입니다.

```java
public class SpellCheckFactoryMethodPattern <E extends Lexicon> {
    private final E dictionary;
    public SpellCheckFactoryMethodPattern(Supplier<E> dictionaryFactory) {
        this.dictionary = Objects.requireNonNull(dictionaryFactory.get());
    }

    public E getDictionary() {
        return dictionary;
    }
}
```

> 클래스 내부적으로 하나 이상의 자원에 의존하고, 그 자원이 클래스 내부 동작에 영향을 준다면 `싱글톤`이나 `유틸리티 클래스`는 사용하지 않는 것이 좋다. 이 자원들을 클래스가 직접 만들게 해서는 안되고 의존 객체 주입 패턴이나 팩토리 메서드 패턴을 쓰자.