---
title: Effective Java 정리 Item10 [Eqauls는 일반 규약을 지켜 재정의하라]
date: 2023-08-26 +09:00
categories: [java]
tags: [effectivejava, java]
---
eqauls 메서드는 Object 클래스에 정의되어 있고 기본적으로 동일성(identity) 검사를 하게 됩니다. 즉, 가지고 있는 속성(멤버 변수)가 같아도 객체가 다르면 false를 반환한다는 말입니다. 이 Equals를 재정의 할 때는 고려해야할 것이 많고 구현하고 검증 과정을 반드시 거쳐야 하기 때문에 정말 eqauls를 구현해야 하는 것이 맞는지 고민을 해봐야 합니다. 책에서는 아래 4가지를 생각해보고 한 가지라도 해당이 된다면 Eqauls 메서드를 구현하지 말라고 합니다.

1. 각 인스턴스가 본질적으로 고유하다
2. 인스턴스의 '논리적 동치성'을 검사할 일이 없다. 여기서 논리적 동치성이란 동등성(eqauilty)를 말한다.
3. 상위 클래스에서 재정의한 eqauls가 하위 클래스에도 딱 들어맞는다.
4. 클래스가 private이거나 package-private이고 equals 메서드를 호출할 일이 없다. 생성자가 private이면 이 클래스는 객체로 만들 수 없는 생성자이고 상속또한 할 수 없기 때문에 eqauls 메서드를 부를 일이 없고 pacakge-private 생성자를 가지면서 equals 메서드를 호출할 일이 없다면 eqauls 메서드를 재정의할 필요가 없다.

eqauls 메서드는 구현할 때 아래 사항을 지키면서 구현해야 한다.
1. 반사성(reflexity) 
   - x.eqauls(x) = true
2. 대칭성(symmetry) 
   - if x.eqauls(y) = true, then y.eqauls(x) = true
3. 추이성(transitivity)
   - if x.eqauls(y) = true and y.eqauls(z) = true, then x.eqauls(z) = true
5. 일관성(consistency)
   - if x.eqauls(y) = true, then must x.eqauls(y) = true anytime
6. null-아님(not null)
   - x.equals(null) = false

위 5가지 사항을 지키지 않는다면 equals를 내부적으로 사용하는 라이브러리나 자료구조는 기대하던 대로 동작하지 않을 수 있습니다. 상속의 다형성을 지원하면서 equals 메서드를 재정의하는 방법은 없습니다. 아래를 예시로 든 코드는 세번째 조건 추이성에 위배됩니다.
```java
public class ColorPoint extends Point{
    private final Color color;

    public ColorPoint(int x, int y, Color color) {
        super(x, y);
        this.color = Objects.requireNonNull(color);
    }

    @Override
    public boolean equals(Object o) {
        if(!(o instanceof Point)) return false;
        if(!(o instanceof ColorPoint)) return super.equals(o);
        return color.equals(((ColorPoint) o).color) && super.equals(o);
    }
}
```

```java
@Test
public void transitivityTest() {
   ColorPoint colorPoint1 = new ColorPoint(1, 2, Color.RED);
   Point p = new Point(1, 2);
   ColorPoint colorPoint2 = new ColorPoint(1, 2, Color.BLUE);

   assertTrue(colorPoint1.equals(p));
   assertTrue(p.equals(colorPoint2));
   assertFalse(colorPoint1.equals(colorPoint2)); // 추이성 실패
}
```

ColorPoint 끼리 비교할 때랑 ColorPoint와 Point를 비교할 때는 비교하는 속성이 다르기 때문에 추이성 테스트가 실패했습니다. 그렇다면 ColorPoint를 아래와 같이 Point와 비교하지 못하게 만들면 어떨가요?

```java
@Override
public boolean equals(Object o) { 
   if(o == null | o.getClass() != getClass()) return false; // Point랑 비교 못하게 아예 막아버림, LSP 위배
   return color.equals(((ColorPointV2) o).color) && super.equals(o);
}
```

이렇게 된다면 추이성 테스트는 통과하겠지만 객체지향 설계원칙(SOLID)중 LSP를 위배하게 됩니다.

```java

@Test
public void LSPTest() {
   Set<Point> set = Set.of(new Point(1, 2));
   ColorPointV2 colorPoint = new ColorPointV2(1, 2, Color.RED);
   assertFalse(set.contains(colorPoint));
}
```

하위 타입은 상위 타입이 수행하던 책임을 수행할 수 있어야 합니다. 하지만 위 코드에서는 ColorPoint가 Point로 수행될 수 있지 않고 무조건 false를 반환하게 될 것 입니다.

따라서 상위 타입을 상속한 하위 타입이 속성을 추가한 상태에서는 eqauls 메서드를 재정의 할 수가 없게 됩니다. 그래서 상속의 다형성은 포기하고 컴포지션을 활용해서 eqauls 메서드를 정의하게 되는데요
```java
public class ColorPointV3 {
    private final Color color;
    private final Point point;

    public ColorPointV3(int x, int y, Color color) {
        this.point = new Point(x, y);
        this.color = Objects.requireNonNull(color);
    }

    public Point asPoint() {
        return this.point;
    }

    @Override
    public boolean equals(Object o) { // 상속을 안함
        if(!(o instanceof ColorPointV3)) return false;

        ColorPointV3 colorPoint = (ColorPointV3) o;

        return this.color.equals(colorPoint.color) && point.equals(colorPoint.point);
    }
}
```

이렇게 멤버 변수로 Point를 가지고 있고 이를 eqauls 메서드에서 비교하는데 사용하게 됩니다.


책에서 eqauls 메서드를 재정의하는데 가이드라인을 알려줍니다.
1. null이면 false
2. 자신과 비교해서 같은 객체라면 true
3. 타입이 같은지 또는 하위 타입인지 확인, 해당하지 않는다면 false
4. 자신과 같은 타입으로 형변환
5. 멤버 변수 비교
```java
/**
 * Best Practice
 */
public class PhoneNumber {
    private final short areaCode, prefix, lineNum;
    private int hashCode;

    public PhoneNumber(short areaCode, short prefix, short lineNum) {
        this.areaCode = areaCode;
        this.prefix = prefix;
        this.lineNum = lineNum;
    }

    private static short rangeCheck(int val, int max, String arg) {
        if(val < 0 || val > max) throw new IllegalArgumentException(arg + ": " + val);
        return (short) val;
    }

    @Override public boolean equals(Object o) {
        if( o == null) return false;
        if (o == this) return true;
        if (!(o instanceof PhoneNumber)) return false;
        PhoneNumber p = (PhoneNumber) o;
        return p.areaCode == areaCode && p.prefix == areaCode && p.lineNum == lineNum;
    }
}
```


> eqauls 메서드를 재정의할 때는 신중해야 하고 책에서 얘기해준 4가지를 떠올려보며 정말 재정의해야하는 상황인지 판단합니다. 그리고 상속을 하면서 멤버변수가 추가된 상황에서는 eqauls를 재정의할 수 없으므로 컴포지트를 이용합니다. eqauls 메서드를 재정의할 때는 가이드라인 대로 만들면 eqauls 일반 규약 5가지를 자연스럽게 지킬 수 있습니다.