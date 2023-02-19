---
title: 왜 제네릭을 쓰는 걸까?
date: 2023-02-18 +09:00
categories: [java]
tags: [java, generic, invariance, covariance, contravariance]
---
 자바 소스코드를 보다보면 제네릭을 사용한 코드를 많이 볼 수 있습니다. 제네릭에 대해서는 단순 타입을 지정해서 사용하는 것이다 정도만 알고 있으니 해당 코드에 대해 정확히 이해하지 못하고 제가 제네릭을 활용하려 할 때 한계가 있었습니다 그래서 제네릭이 언제 도입되었는지 왜 사용하는지를 나름 정리해서 업무에 활용해보려고 합니다.

### 제네릭을 사용하는 이유
 제네릭은 다이아몬드 연산자 안에 타입을 입력하면 해당 타입으로 클래스를 사용할 수 있습니다. 자바 컬렉션과  조합이 매우 좋아 대부분의 컬렉션에서 제네릭을 사용했습니다.

 ```java
 List<String> list = List.of("solana", "oopty", "lisa");
 list.add("alex");
 ```

그러면 제네릭은 특정 클래스를 여러 타입으로 사용할 수 있게 재사용하려고 만든 걸까요? 제네릭이 없으면 이런 구현이 불가능 할까요? 그렇지 않습니다. 객체를 Object에 담고 타입 캐스팅으로 바꿔서 사용하면 구현 가능하겠죠 하지만 이렇게 하면 타입 캐스팅을 하는 불편함이 존재하고 타입 캐스팅을 할 때 담긴 객체가 해당 타입과 공변성을 가지지 못한다면 런타임 환경에 에러가 납니다. 이는 실제 운영환경에서 버그가 하나 추가되는 겁니다. 이를 type safety 하지 않다고 하는데 이를 해결한 것이 제네릭입니다.

#### Generic은 언제 나왔을까?
제네릭은 2004년에 J2SE 5.0 스펙으로 나왔습니다. 제네릭의 특징은 type removal인데요. type removal는 컴파일 타임에 지정한 타입을 지우고 형변한 연산자로 대체합니다. 이렇게 한 이유는 전 버전과의 호환 때문이라고 하네요 1.4 스펙에서는 아래와 같이 구현을 했고 기존 레거시 코드가 1.5로 넘어올 때 쉽게 넘어올 수 있게 만드려고 했다고 합니다.

```java
List zoo = new ArrayList();
for (int i = 0; i < zoo.size(); i++) {
  if (!(zoo.get(i) instanceof Animal)) {
    continue;
  } else {
    Animal animal = (Animal) zoo.get(i);
    feed(animal);
  }
}
```


#### 자바 Array의 공변성
Array는 기본적으로 Type safety하지 않습니다. 타입에 맞지 않은 것을 대입하려 하면 런타임 에러를 발생시키는데요. 예를 들어 아래 코드와 같이 type이 맞지 않는 경우 `ArrayStroeException`이 일어납니다. (Sub1과 Sub2는 Super의 하위 클래스)

```java
Super[] superArray = new Sub1[3];
superArray[0] = new Sub2(); // ArrayStoreException
```

컴파일 타임에 이를 알 수 있다면 정말 좋겠죠? 그래서 제네릭은 아래와 같이 제네릭이 다른 타입의 클래스는 상속관계에 상관없이 대입이 불가능하게 만듭니다. 이를 무공변성(invariance)라고 합니다.

```java
List<Super> superList = new LinkedList<Sub1>(); // compile error; cannot convert List<Sub1> to List<Super>
```

### 제네릭에서 공변(covariance)와 반공변(contravariance)를 사용할 수는 없을까?

제네릭의 무공변성은 객체지향 프로그래밍의 많은 이점 중 하나인 다형성(polymorphism)을 잃게 됩니다. 너무 많은 걸 잃어버렸네요 ㅜㅜ 그래서 type-safety를 어느정도 지키면서 다형성도 얻을 수 있게 약간 유연성을 줄 수 있습니다. `extends`와 `super` 키워드인데요 아래 코드를 보면서 설명하겠습니다.

```java
public addToList(List<? extends Super> list) {
  Super super = list.get(0);
  list.add(new Super()) // compile error
}
```

위 `addToList`함수는 `List<Super>`와 Super를 구현한 하위 클래스 타입의 리스트들을(`List<Sub1>`, `List<Sub2>`) 매개변수로 받을 수 있습니다.

그리고 함수 반환값은 Super객체로 변환됩니다. 매개 변수로 Super 객체를 넘겨주려하면 compile error가 발생합니다. 이렇게 하는 이유는 type-safe를 유지하려고 하기 때문입니다.

또 아래와 같이 형변환이 가능한데 이렇게 하면 type-safe하지 않아서 지양합시다.

```java
public addToList(List<? extends Super> list) {
  List<Sub1> sub1List = list; // successfully compiled
}
```

### T extends Comparable<? super T>
 Sort 함수를 살펴보면 상당히 복잡하게 생긴 제네릭을 볼 수 있습니다. 
 ```java
public static <T extends Comparable<? super T>> void sort(List<T> list) {
    list.sort(null);
}
 ```
 위 제네릭의 T는 어떻게 해석해야 할 까요? 일단 `T extends Comparable` 부분은 T가 Comparable을 상속해야한다고 알려줍니다. 만약 상속하지 않았으면 컴파일 에러가 발생합니다. `<? super T>`는 T의 수퍼클래스 타입의 Comparable 클래스를 말합니다. 이게 왜 필요할까요? 아래와 같이 Super 클래스에서 Comparable을 구현해 놓고 List<Sub1> 객체를 정렬하려 한다면 컴파일이 될까요?

 ```java
 class Super implements Comparable<Super> {
    @Override
    public int compareTo(Super o) {
        // 비교 연산
    }
 }

 // in main method
 List<Sub1> list = new LinkedList();
 Collections.sort(list) // compile error;
 ```
당연히 안되겠죠 제네릭은 무공변성이니까! 여기서 반공변성을 주고 싶으면 super 키워드를 줘서 `<? super T>`를 사용한 것 입니다.

### 제네릭을 사용하는 관례
제네릭 이름은 아래와 같이 특정 용도에 따라 관례적으로 사용하는 이름이 있으며 꼭 이걸 따를 필요는 없습니다. 그리고 한 글자로 정하는게 대부분이지만 두 글자 이상이어도 상관없다고 합니다.

| 타입 | 설명    |
| :--- | :------ |
| \<T>  | Type    |
| \<E>  | Element |
| \<K>  | Key     |
| \<V>  | Value   |
| \<N>  | Number  |