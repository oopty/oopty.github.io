---
layout: post
title:  "스프링 부트 예외처리 완벽 가이드"
date:   2022-01-16 21:40:00 +0900
categories: Spring
---
![404에러](/assets/images/20220116_404-not-found.jpg)

회사에서 `Spring Boot` 관련 프로젝트를 진행하면서 스프링에서 지원하는 어노테이션 스펙이 궁금했고 우리 프로젝트엔 어떻게 적용되어 있는지 살펴보고 싶었다. 때마침 기술관련된 내용을 잘 정리한 사이트를 찾아서 예외처리 관련된 주제를 번역하면서 정리하려고 합니다. 아래 내용은 [Complete Guide to Exception Handling in Spring Boot](https://reflectoring.io/spring-boot-exception-handling/#responsestatus)에 있던 내용을 제가 이해한 수준으로 번역한 결과입니다.

예외 처리는 탄탄한 어플리케이션을 만드는데 중요한 부분을 차지합니다. 스프링 부트에선 여러가지 방법을 제시합니다. 이 글에선 여러가지 방법을 시험해보고 다른 방법보다 더 선호되는 방법이 있을 땐 제안도 드릴겁니다.

## 예시 코드
이 글은 GitHub에 있는 [동작 코드](https://github.com/thombergs/code-examples/tree/master/spring-boot/exception-handling)를 제공합니다.

## 소개
스프링 부트는 단순한 `try=catch`를 넘어선 예외처리 도구를 제공합니다. 이러한 도구를 이용하기 위해 우리는 관점을 분리로서 예외처리를 다루는 몇가지 어노테이션을 사용합니다.

- `@ResponseStatus`
- `@ExceptionHandler`
- `@ControllerAdvice`

어노테이션들을 보기 전에 우리는 먼저 스프링이 어떻게 웹 컨트롤러에서 던저진 에러를 처리하는지 살펴볼것입니다 - 에러를 처리하는 마지막 단계

또한 기본 설정을 바꾸기위해 스프링 부트에서 제공하는 몇가지 설정도 살펴볼 예정입니다.

우리가 예외를 처리하면서 겪는 어려움을 인지하고나서 우리는 어노테이션으로 해결할 겁니다.

## 스프링 부트 기본 예외처리 메커니즘

주어진 id가 없을 때 `NoSuchElementException` 런타임 에러를 던지는 `getProduct` 메서드를 가지고 있는 `ProductController`가 있다고 가정해봅시다.

```java
@RestController
@RequestMapping("/product")
public class ProductController {
  private final ProductService productService;
  //constructor omitted for brevity...
  
  @GetMapping("/{id}")
  public Response getProduct(@PathVariable String id){
    // this method throws a "NoSuchElementFoundException" exception
    return productService.getProduct(id);
  }
  
}
```

우리가 유효하지 않은 id로 `/product` API를 호출하면 `NoSuchElementFoundException` 런타임 에러를 발생시킬거고 다음 Response를 얻게될 겁니다.

```json
{
  "timestamp": "2020-11-28T13:24:02.239+00:00",
  "status": 500,
  "error": "Internal Server Error",
  "message": "",
  "path": "/product/1"
}
```

페이로드는 올바른 형식의 에러 메시지 외에 어떤 정보도 주지 않습니다. 심지어 `message`필드는 비어있지만 우리는 "Item with id 1 not found"같은 것을 포함하길 기대합니다.

이 에러 메시지 이슈를 고쳐봅시다.

스프링 부트는 우리가 에러 메시지, 에러 클래스 또는 Response 페이로드의 스택 트레이스 일부를 더할 수 있는 몇가지 속성을 제공합니다.

```yml
server:
  error:
  include-message: always
  include-binding-errors: always
  include-stacktrace: on_trace_param
  include-exception: false
```

