---
title: Java의 공통 메서드
date: 2023-01-08 +09:00
categories: [java]
tags: [java, effective java, Object]
---

안녕하세요, 오랜만에 글을 남기는데 회사에서 부서이동도 하고 출장도 갔다오면서 바쁜 나날을 보내 블로그 포스팅은 잠시 접어 두었네요. Effective Java를 읽으면서 내용 정리를 해보려고 합니다. 공부 기록 목적도 있고 복습 차원에서 글을 남기는 점 고려하시고 틀린 내용은 지적 부탁드리겠습니다~!

먼저 자바에서 기본 타입이 아닌 모든 객체는 `Object`를 상속하고 있습니다. 따라서 이 `Object`가 제공하는 `equals`, `hashCode`, `toString`메서드는 모든 객체가 사용할 수 있습니다.
`clone`, `compare` 메서드는 각각 `Clonable`, `Comparable` 인터페이스를 구현하면 사용할 수 있습니다. 각 메서드를 오버라이드해서 재정의하는 편이 좋은지 아니면 상위 클래스가 구현한 대로 사용하는 편이 좋은지 판단하는 방법을 말씀드릴 겁니다.
먼저 `equals`부터 살펴보겠습니다~

## eqauls
Object의 eqauls 메서드 구현은 아래와 같습니다.

```java
public boolean equals(Object obj) {
  return (this == obj);
}
```

소스 코드에서 보듯이 메모리에 정확히 같이 위치한 객체만이 동일하다고 판단합니다. 이렇게 비교하면 모든 중요 필드를 비교하지 않아도 한번이 비교로 판단할 수 있으니 효과적입니다. 하지만 이렇게 동일성을 비교하는 것이 아닌 동등성을 비교하는 것이라면 아래와 같이 equals 매서드를 오버라이드해서 재정의 해줘야 합니다.

```java
public boolean eqauls(Object obj) {
  if(!obj instaceOf MyClass) return false;
  MyClass o = (MyClass) obj;
  return this.field1 == o.field1 && this.field2 == o.field2; // 중요 필드 비교
}
```

이렇게 재정의를 할 때 주의해야할 부분이 있습니다. 바로 eqauls 생성 규칙에 맞아야 하는데요 이걸 지키지 않으면 이를 사용하는 Collection에서 기대하는 대로 동작하지 않을 수 있습니다.

1. reflexive (반사성), x.eqauls(x)는 항상 참이어야 한다.
2. symmetric (대칭성), x.equals(y)가 참일 때만, y.equals(x)가 참이어야 한다
3. transitive (추이성) 만약 x.eqauls(y)와 y.equals(z)가 참이 라면, x.equals(z)가 참이어야 한다.
4. consistent (일관성) if x.eqauls(y)가 참이거나 거짓이라면 객체 안의 값이 변경되지 않는 한 계속 참이거나 거짓이어야 한다.
5. x.eqauls(null)는 항상 거짓이어야 한다.

그리고 상속을 통해 값을 추가하고 `equals`메서드를 재정의하려고 해서는 안됩니다. 위 5가지를 지키면서 `equals`메서드를 재정의하는 방법이 없기 때문입니다. 값을 추가하고 싶다면 상속이 아닌 해당 클래스를 필드로 두어서 해결해야 합니다.
자바 라이브러리에서도 이를 지키짐 못한 클래스가 있는데 바로 `java.util.Date`를 확장한 `java.sql.Timestamp`클래스입니다. `nanoseconds`필드를 추가해서 eqauls의 대칭성이 위배되었습니다. 그래서 Date 클래스와 같이 쓸 때 주의해야 합니다.
equals를 재정의할 때 각 필드마다 어떻게 비교를 해야하는지 팁을 주자면 float와 double을 제외한 기본 타입을 비교할 때는 == 연산자를 사용하고 float와 double은 Float.compare(float, float), Double.compare(double, double)을 사용합니다. 객체는 equals 메서드를 재귀적으로 타면 됩니다.

equals 매서드는 동등성 검사를 꼭 해야하는 상황이 아니면 재정의하지 않는 편이 좋다고 합니다. 그리고 재정의 할 때는 위 다섯가지 규칙을 꼭 지켜야 합니다.

## hashCode
`equals`를 재정의했다면 `hashCode`도 재정의를 해야합니다. 그렇지 않다면 아래 코드가 기대와 다르게 동작할 수 있습니다.
```java
Map<MyKey, Integer> map = new HashMap();
map.put(new MyKey("mytest"), 1);

map.get(new MyKey("mytest")); // null을 반환
```

첫번째 객체와 두번째 객체의 hashCode가 다르기 때문에 값이 다른 버킷에 들어가 있고 원하는 결과인 1이 나오지 않습니다.

hashCode를 재정의할 때는 아래와 같이 전에 계산했던 해시값에 31을 곱해주면서 반복적으로 값을 계산하는 방식이 대중적입니다. `Objects.hash`는 느려서 사용을 지양합니다.

```java
@Override public int hashCode() {
  int result = Short.hashCode(field1);
  result = 31 * result + Short.hashCode(field2);
  result = 31 * result + Short.hashCode(field3);
  return result;
}
```

그리고 불변 객체라면 hashCode가 바뀌지 않기 때문에 이를 캐싱해 놓을 수 있는데 이 때는 thread-safe하게 작성해야 합니다.
```java
@Override public int hashCode() {
  int result = hashCode;
  if(result == 0) {
    int result = Short.hashCode(field1);
    result = 31 * result + Short.hashCode(field2);
    result = 31 * result + Short.hashCode(field3);
    hashCode = result;
  }
  
  return result;
}
```
hashCode 반환 값과 만드는 규칙은 최대한 client에게서 감춰야 합니다. 그렇게 해야 client 코드와 상관없이 hashCode를 유연하게 고칠 수 있습니다.

## toString
toString은 항상 재정의를 해줘서 디버깅을 쉽게 도와주게 해주는게 좋습니다. 객체가 가진 거의 모든 정보를 반환해주는게 좋고 포맷은 정확히 표시하고 추후 바뀌지 않을 것이라고 확신이 들면 정확한 설명을 해야합니다.

그렇지 않다면 "해당 toString의 상세 형식은 정해지지 않았으며 향후 변경될 수 있습니다"라는 문구를 추가해주는 것이 좋습니다.
```java
/** 상세 형식은 정해지지 않았으며 향후 변경될 수 있습니다.
* "[약물 #9: 유형=사랑, 냄새=내레빈유, 겉모습=먹물]"
*/
@Override public String toString() {...}
```
이렇게 하면서 toString의 반환값을 파싱해서 클라이언트가 사용하는 것을 막을 수 있습니다.

## clone
clone 메서드는 Object 클래스에 명세가 있고 사용 여부는 Clonable 인터페이스 구현으로 제어하는 방식입니다.
Clonable 인터페이스를 구현하면 구현한 클래스의 상위 클래스의 clone 메서드를 수정합니다. 상위 클래스의 모든 필드를 복사 후 반환하고 구현하지 않는다면 상위 클래스의 clone 메서드를 호출 시 CloneNotSupportedException을 던집니다. 매우 예외적으로 인터페이스를 사용한 경우라 따라하시면 안됩니다.
clone 메서드를 구현할 때는 기존 `protected`로 되어 있던 접근 제어자를 `public`으로 바꿔주고 상위 클래스의 clone 메서드를 호출 후 자신의 클래스로 형 변환을 해주어 클라이언트에서 편리하게 사용할 수 있게 해줍니다. 클래스에 추가 필드가 있다면 해당 필드도 복사하는 로직을 넣어줍니다. 그리고 CloneNotSupportedException는 clone을 구현했으므로 try-catch로 묶어서 unchecked exception으로 변환해서 던집니다.(`AssertionError`)
만약 가변 객체를 필드로 가지고 있다면 문제가 복잡해집니다. 해당 가변 객체를 deep copy하는 로직을 직접 작성해야 하며 이로 인해 성능 이슈가 생길 수 있으니 신중히 작성해야 합니다. 그리고 가변 객체 필드를 final로 작성할 수 없어서 `모든 가변 객체는 final이어야 한다`는 원칙을 위배하는 경우입니다.
이펙티브 자바에서 말하기를 clone 메서드 구현보다 복사 생성자나 복사 팩터리를 구현하는게 더 좋은 방법이라고 합니다. 이 경우에는 가변 객체 필드를 final로 선언할 수 있고 불필요한 형변환을 해야할 필요가 없으며 예외 검사를 할 필요도 없습니다.

## Comparable
Comparable 인터페이스는 compareTo(T o) 시그니처 형태에 메서드로 객체의 순서를 지정해주는 역할을 합니다. 이는 Collection에서 쓰이기 해서 순서를 고려해야 하는 클래스이면 꼭 작성해야 합니다. compareTo 메서드 일반 규약은 equals 메서드와 비슷합니다.
- 모든 x, y에 대해 sgn(x.compareTo(y)) == -sgn(y.compareTo(x)) 입니다.
- 추이성으로 보장해야 합니다. x.compareTo(y) > 0 && y.compareTo(z) > 0 이면 x.compare(z) > 0 입니다.
- 모든 z에 대해 x.compareTo(y) = 0 이면 x.compare(z) = y.compareTo(z) 입니다.
- 이거는 꼭 지킬 필요는 없지만 권고사항입니다. x.compareTo(y) = 0 이면 x.eqauls(y)는 참입니다. 이를 지키지 않는다면 명시를 해야합니다.

네번째 룰을 어긴 것이 BigInteger입니다. 사용할 때 주의가 조금 필요합니다.

compareTo를 구현할 때 >, < 같은 관계연산자를 사용하면 에러가 날 수 있으므로 java7 이상을 사용한다면 박싱된 기본 타입의 compare메서드를 이용하셔야 합니다. 이를 이용해서 구현하면 아래와 같은 패턴으로 작성할 수 있습니다.
```java
public int compareTo(PhoneNumber pn) {
  int result = Short.compare(areaCode, pn.areaCode);
  if (result == 0) {
    result = Short.compare(prefix, pn.prefix);
    if (result == 0)
      result = Short.compare(lineNum, pn.lineNum);
  }
  return result;
}
```

이를 아래와 같이 Comparator 인터페이스의 비교자 생성 메서드로 구현하면 간단하다.

```java
private static final Comparator<PhoneNumber> COMPARATOR =
  comparingInt((PhoneNumber pn) -> pn.areaCode)
    .thenComparingInt(pn -> pn.prefix)
    .thenComparingINt(pn -> pn.lineNum);

public int compareTo(PhoneNumber pn) {
  return COMPARATOR.compare(this, pn);
}
```

이따금 '값의 차'를 이용해 compareTo를 구현하려고 하는 경우가 있습니다. 저도 이렇게 많이 했는데 이러면 정수 오버플로우를 일으켜 추이성이 깨지거나 IEEE 754 부동소수점 계산방식에 의해 오류가 날 수 있습니다. 그래서 아래와 같이 박싱된 기본 타입의 compare 메서드로 구현을 합시다

```java
static Comparator<Object> hashCodeOrder = new Comparator<>() {
  public int compare(Object o1, Object o2) {
    return Integer.compare(o1.hashCode(), o2.hashCoe());
  }
}
```

이렇게 Object 클래스에 있는 기본메서드와 Comparable 인터페이스에 대해 알아보았는데 각 메서드의 일반 규약을 살펴보고 해당 규약에 맞춰 개발해야 한다는게 이번 챕터의 메인 주제였습니다. 이펙티브 자바를 보면서 기존에 제가 작성했던 코드에 대해 생각하게 되고 좀 더 좋은 코드를 작성할 수 있게 제 손가락에 내재화 시켜야겠습니다. 🤔