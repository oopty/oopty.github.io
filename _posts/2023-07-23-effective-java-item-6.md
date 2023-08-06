---
title: Effective Java 정리 Item6 [불필요한 객체 생성을 피하라]
date: 2023-07-23 +09:00
categories: [java]
tags: [effectivejava, java]
---
Item6에서는 불필요한 객체를 만들어내는 실수에 대해 다룹니다. 예를 들어, `new String("bikini")`와 `"bikini"`는 기능적으로 완전히 똑같지만 메모리와 cpu 자원을 낭비합니다. (라고 책에서 말하지만, Java 컴파일러가 잡아서 최적화 시켜줄 거 같습니다 ㅎㅎ) 또한 생성자보단 정적 팩토리 메서드를 사용해서 불필요한 객체 생성을 막는 것도 좋은 습관이 될 거 같습니다.
 생성 비용이 비싼 객체는 더욱 중복된 객체가 만들어지는 것에 주의해야합니다. 예를 들어 아래 `Pattern`객체는 한번 생성될 때 유한 상태 머신도 같이 만들어 지기 때문에 비용이 많이 듭니다.

 ```java
static boolean isRomanNumeralV1(String s) {
    return s.matches("^(?=.)M*(C[MD]|D?C{0,3})" + "(X|[CL]|L?X{0,3})(I[XV]|V?I{0,3})$");
}
 ```

 위에는 메서드가 실행될 때마다 `Pattern`객체를 만듭니다. 아래는 개선된 버전입니다.

 ```java
 public class RomanNumerals {
    private static final Pattern ROMAN = Pattern.compile(
            "^(?=.)M*(C[MD]|D?C{0,3})" + "(X|[CL]|L?X{0,3})(I[XV]|V?I{0,3})$");
    static boolean isRomanNumeralV1(String s) {
        return s.matches("^(?=.)M*(C[MD]|D?C{0,3})" + "(X|[CL]|L?X{0,3})(I[XV]|V?I{0,3})$");
    }

    static boolean isRomanNumeralV2(String s) {
        return ROMAN.matcher(s).matches();
    }
}
 ```

미리 클래스 필드로 패턴객체를 선언해 재사용하는 코드입니다. 제 컴퓨터에서 100번 반복 실행했을 때 5.755 milliseconds와 0.870 milliseconds로 약 7배정도 차이가 나네요.
이보다 더 개선하는 방법은 지연 초기화(`lazy initialization`)이 있는데 추가적인 코드가 들어가고 성능 개선은 크지 않기 때문에 비추천한다고 하네요.

불필요한 객체를 만들어내는 또 다른 예는 오토박싱입니다.
```java
private static Long sum() {
    Long sum = 0;
    for(int i = 0; i < Integer.MAX_VALUE; i++) {
        sum += i;
    }

    return sum;
}
```
위 코드는 i가 sum에 더해질 때마다 오토 박싱이 일어납니다. 박싱된 타입보다는 기본 타입을 사용하고 의도치 않게 오토박싱이 숨어들지 않도록 주의해야 합니다.

> 불필요한 객체 생성때문에 일어나는 성능 저하를 얘기 했지만, 필요한 객체 생성을 하지않는 일은 없어야 합니다. 필요한 객체 생성을 하지않아서 생기는 버그나 보안 구멍은 단순한 성능 저하 문제보다 훨씬 복잡하고 피해가 큽니다. 객체를 생성할 때는 꼭 방어적 복사를 수행하고 불필요한 객체를 선별하는 과정에서 신중합시다