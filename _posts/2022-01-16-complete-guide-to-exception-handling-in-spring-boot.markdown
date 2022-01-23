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
[스프링 부트 서버 속성](https://docs.spring.io/spring-boot/docs/current/reference/html/application-properties.html#application-properties.server)을 사용하면 에러 메시지를 어느 정도 바꿀 수 있습니다.

지금 유효하지 않은 id로 `/product` API를 요청하면 다음 리스폰스를 받게 됩니다.
```json
{
  "timestamp": "2020-11-29T09:42:12.287+00:00",
  "status": 500,
  "error": "Internal Server Error",
  "message": "Item with id 1 not found",
  "path": "/product/1"
} 
```

우리가 `include-stacktrace`를 `on_trace_param`로 설정한 것을 기억하세요. 이것이 의미하는 바는 url에 trace 파라미터를 포함하는 경우(?trace=true), 
리스폰스 페이로드에서 스텍 트레이스를 얻을 수 있다는 것입니다.

```json
{
  "timestamp": "2020-11-29T09:42:12.287+00:00",
  "status": 500,
  "error": "Internal Server Error",
  "message": "Item with id 1 not found",
  "trace": "io.reflectoring.exception.exception.NoSuchElementFoundException: Item with id 1 not found...", 
  "path": "/product/1"
} 
```

우리는 적어도 운영 제품에서는 `include-stacktrace`를 `never`로 유지하고 싶을 수 있습니다. 우리 어플리케이션의 내부 동작을 들어낼 수 있기 때문입니다.

문제는 계속 있습니다! 상태 코드(500)은 서버 코드에서 무언가 이상하다는 것을 가르키지만, 실제로는 클라이언트가 유효하지 않은 id를 전달했기 때문입니다.

우리의 현재 에러 메시지 상태 코드는 이런 정보를 정확히 전달하지 않습니다. 불행히도 `server.error` 설정 속성으로 하는 한 이정도만 적용되므로, 스프링 부트에서 제공하는 어노테이션들을 살펴봐야 합니다.

## ResponseStatus

이름 그대로, `@ResponseStatus` 응답 리스폰스의 HTTP status를 수정할 수 있게 해줍니다. 아래 위치에 적용 가능합니다.

- 예외 클래스에서
- `@ExceptionHanlder` 어노테이션을 가진 메서드에서
- `@ControllerAdvice` 어노테이션을 가진 클래스에서

이번 섹션에선, 첫번째 경우를 살펴볼 것 입니다.

다시 문제상황으로 돌아와보면, 우리 에러 리스폰스는 더 설명적인 status code 대신 항상 HTTP status 500인 것 입니다.

이 문제를 해결하기 위해, 우리는 예외 클래스에 `@ResponseStatus`를 함께 사용하고 `value` 속성에 원하는 HTTP status를 넣어주면 됩니다.

```java
@ResponseStatus
public class NoSuchElementFoundException extends RuntimeException {
  ...
}
```

만약 우리가 유효하지 않은 id로 controller를 호출한다면, 이 수정사항이 더 나은 리스폰스 가져올 것입니다.

```json
{
  "timestamp": "2020-11-29T09:42:12.287+00:00",
  "status": 404,
  "error": "Not Found",
  "message": "Item with id 1 not found",
  "path": "/product/1"
} 
```

같은 결과를 만드는 또 다른 방법은 `ReponseStatusException` 클래스를 상속받는 것입니다

```java
public class NoSuchElementFoundException extends ResponseStatusException {

  public NoSuchElementFoundException(String message){
    super(HttpStatus.NOT_FOUND, message);
  }

  @Override
  public HttpHeaders getResponseHeaders() {
      // return response headers
  }
}
```

`getResponseHeaders()` 메서드를 오버라이드 할 수 있기 때문에 HTTP header를 조종하려고 할 때 이 접근방법은 유용합니다.

하지만 리스폰스 페이로드 구조도 수정하고 싶다면 어떻게 해야할 까요?

다음 섹션에서 어떻게 해결할 수 있는지 살펴봅시다.

## @ExceptionHandler

`@ExceptionHandler` 어노테이션은 예외 처리에 관해서 많은 유연성을 줍니다. 초보자로서 이것을 사용하려면, `@ControllerAdvice` 클래스나 컨트롤러 자체에서 매서드 하나를 만들면 됩니다. 그리고 그 메서드에 `@ExceptionHandler` 어노테이션을 붙혀줍니다.

```java
@RestController
@RequestMapping("/product")
public class ProductController {
  private final ProductService productService;

  // Constructor omitted for breviy...

  @Getmapping("/{id}")
  public Response getProduct(@PathVariable String id) {
    return productService.getProduct(id);
  }

  @ExceptionHandler(NoSuchElementFoundException.class)
  @ResponseStatus(HttpStatus.NOT_FOUND)
  public ResponseEntity<String> handleNoSuchElementNotFoundException(
    NoSuchElementNotFoundException exception
  ) {
    return ReponseEntity
      .status(HttpStatus.NOT_FOUND)
      .body(exception.getMessage());
  }
}
```
예외 핸들러 메서드는 정의된 메서드에서 다루기를 원하는 예외나 예외 리스트를 인자로 받습니다. 이 메서드를 `@ResponseHandler`와 `@ResponseStatus`와 함께 지정해서 우리가 다루기 원하는 예외와 반환하려는 상태코드를 정의합니다.

만약 이러한 어노테이션을 사용하고 싶지 않다면, 메서드의 인자로 간단히 정의하는 방법도 똑같이 동작합니다.
```java
@ExceptionHandler
public ResponseEntity<String> handleNoSuchElementNotFoundexception(
  NoSuchElementFoundException exception
)
```

하지만 메서드 시그니쳐에서 이미 예외 클래스를 정의했지만 어노테이션에 예외 클래스를 정의하는 것은 더 좋은 가독성을 위해 좋은 생각입니다.

또한 핸들러 메서드에 `ResponseStatus(HTTPStatus.NOT_FOUND)` 어노테이션은 ResponseEntity로 전달한 HTTP 상태가 우선되므로 필요 없습니다. 하지만 가독성을 위해 우리는 그대로 두겠습니다.

예외처리 파라미터와는 별도로, `HttpServletRequest`, `WebRequest` or `HttpSession`을 파라미터로 받을 수 있습니다.

더불어, 핸들러 메서드는 `ResponseEntity`, `String` 또는 `void`처럼 다양한 반환 타입을 지원합니다.

더 많은 입력과 반환 타입을 [@ExceptionHandler java documentation](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/bind/annotation/ExceptionHandler.html)에서 찾아보세요.

예외처리에서 입력과 반환 타입의 형태에 다양한 옵션을 이용 가능하므로, 우리는 에러 응답을 완벽히 제어할 수 있습니다.

이제 우리 API의 에러 리스폰스를 완성해봅시다. 어떤 경우에는, 사용자는 두 가지를 기대합니다.
 - 에러 코드는 사용자에게 어떤 종류의 에러인지 알려줍니다. 에러 코드는 클라이언트 단에서 어떤 비지니스 로직을 수행할 지 결정합니다. 대게 에러코드는 표준 HTTP status 코드를 사용하지만, E001처럼 사용자 지정 에러 코드를 반환하는 것도 보았습니다.
 - 에러에 대해 더 많은 정보를 주는 추가적인 인간 친화적 메시지이고 어떻게 고칠지에 대한 힌트와 API 링크입니다.

우리는 또 `stackTrace` 필드를 더하므로써 개발환경에서 디버깅을 도와줄 수 있습니다.

마지막으로 우리가 검증 에러에 대해 다루고 싶다면 빈 검증 시스템에 대해 [Handling Validations with Spring Boot](https://reflectoring.io/bean-validation-with-spring-boot/)에서 더 많은 정보를 찾을 수 있습니다.

위 상황을 계속 상기하면서 에러 응당에 대해 다음 페이로드를 사용할 것입니다.

```java
@Getter
@Setter
@RequiredArgsConstructor
@JsonInclude(JsonInclude.Include.NON_NULL)
public class ErrorResponse {
  private final int status;
  private final String message;
  private String stackTrace;
  private List<ValidationError> errors;

  @Getter
  @Setter
  @RequiredArgsConstructor
  private static class ValidationError {
    private final String field;
    private final String message;
  }

  public void addValidationError(String field, String message){
    if(Objects.isNull(errors)){
      errors = new ArrayList<>();
    }
    errors.add(new ValidationError(field, message));
  }
}
```
이제 이거를 `NoSuchElementNotFoundException` 핸들러 메서드에 적용해봅시다

```java
@RestController
@RequestMapping("/product")
@AllArgsConstructor
public class ProductController {
  public static final String TRACE = "trace";

  @Value("${reflectoring.trace:false}")
  private boolean printStackTrace;
  
  private final ProductService productService;

  @GetMapping("/{id}")
  public Product getProduct(@PathVariable String id){
    return productService.getProduct(id);
  }

  @PostMapping
  public Product addProduct(@RequestBody @Valid ProductInput input){
    return productService.addProduct(input);
  }

  @ExceptionHandler(NoSuchElementFoundException.class)
  @ResponseStatus(HttpStatus.NOT_FOUND)
  public ResponseEntity<ErrorResponse> handleItemNotFoundException(
      NoSuchElementFoundException exception, 
      WebRequest request
  ){
    log.error("Failed to find the requested element", exception);
    return buildErrorResponse(exception, HttpStatus.NOT_FOUND, request);
  }

  @ExceptionHandler(MethodArgumentNotValidException.class)
  @ResponseStatus(HttpStatus.UNPROCESSABLE_ENTITY)
  public ResponseEntity<ErrorResponse> handleMethodArgumentNotValid(
      MethodArgumentNotValidException ex,
      WebRequest request
  ) {
    ErrorResponse errorResponse = new ErrorResponse(
        HttpStatus.UNPROCESSABLE_ENTITY.value(), 
        "Validation error. Check 'errors' field for details."
    );
    
    for (FieldError fieldError : ex.getBindingResult().getFieldErrors()) {
      errorResponse.addValidationError(fieldError.getField(), 
          fieldError.getDefaultMessage());
    }
    return ResponseEntity.unprocessableEntity().body(errorResponse);
  }

  @ExceptionHandler(Exception.class)
  @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
  public ResponseEntity<ErrorResponse> handleAllUncaughtException(
      Exception exception, 
      WebRequest request){
    log.error("Unknown error occurred", exception);
    return buildErrorResponse(
        exception,
        "Unknown error occurred", 
        HttpStatus.INTERNAL_SERVER_ERROR, 
        request
    );
  }

  private ResponseEntity<ErrorResponse> buildErrorResponse(
      Exception exception,
      HttpStatus httpStatus,
      WebRequest request
  ) {
    return buildErrorResponse(
        exception, 
        exception.getMessage(), 
        httpStatus, 
        request);
  }

  private ResponseEntity<ErrorResponse> buildErrorResponse(
      Exception exception,
      String message,
      HttpStatus httpStatus,
      WebRequest request
  ) {
    ErrorResponse errorResponse = new ErrorResponse(
        httpStatus.value(), 
        exception.getMessage()
    );
    
    if(printStackTrace && isTraceOn(request)){
      errorResponse.setStackTrace(ExceptionUtils.getStackTrace(exception));
    }
    return ResponseEntity.status(httpStatus).body(errorResponse);
  }

  private boolean isTraceOn(WebRequest request) {
    String [] value = request.getParameterValues(TRACE);
    return Objects.nonNull(value)
        && value.length > 0
        && value[0].contentEquals("true");
  }
}
```

고려해야할 사항들을 여기에 말해보겠습니다.

## 스택 트레이스 제공
스택 트레이스를 제공하는 것은 개발자와 QA 엔지니어가 로그파일을 살펴보는 시간을 구할 수 있습니다. 스프링 기본 예외 처리 메커니즘에서 본 것처럼 스프링은 이미 우리에게 기능을 제공합니다. 하지만 이제 에러 응답을 직접 다룬 것 처럼 이것도 직접 다뤄지길 필요가 있습니다.

그러기 위해, `refactoring.trace` 서버-사이드 속성을 소개할 것입니다. 이것은 `true`로 설정되면 응답에서 `stackTrace`를 필드를 제공합니다. 실제로 API에서 `stackTrace`를 얻기 위해서는 클라이언트는 추가적으로 `trace` 파라미터를 `true`로 보내야 합니다.

```curl
curl --location --request GET 'http://localhost:8080/product/1?trace=true
```

이제는 `stackTrace`의 기능은 우리 속성 파일의 기능 플래그에 의해 제어되므로 우리는 그것을 운영 환경에서는 삭제하거나 `false`로 설정할 수 있습니다.

## 모든 에러 잡기

Gotta catch em all:
```java
try{
  performSomeOperation();
} catch(OperationSpecificException ex){
  //...
} catch(Exception catchAllExcetion){
  //...  
}
```
예방조치로 우리는 때때로 원하지 않는 부작용이나 기능을 피하기 위해, 상위 메서드를 모든 에러를 잡는 블럭으로 감쌉니다. 우리 컨트롤러에서 `handleAllUncaughtException()` 메서드는 비슷하게 동작합니다. 이것은 우리가 특정한 메서드가 연결되지 않은 모든 예외를 잡습니다.

여기서 말하고 싶은 한가지는 우리가 만약 이 메서드를 사용하지 않더라도 스프링이 모든 예외를 처리할 것입니다. 하지만 우리는 응답 형식이 스프링의 형식보다는 우리 형식을 따르기를 원하기 때문에 이 메서드를 정의해야 합니다.

이 메서드는 가능한 버그에 대해 통찰을 제공하는 예외를 로깅하는데 좋은 곳입니다. `MethodArgumentNotValidException`같은 검증 예외는 스킵할 수 있습니다. 왜냐하면 문법적으로 유효하지 않은 입력에 의해 생기기 때문입니다. 하지만 예상치 못한 예외는 이 catch-all 핸들러 에서 로깅해야합니다.

### 예외 처리의 순서

예외를 처리하는 순서는 중요하지 않습니다. **스프링은 첫번째로 해당 에러와 가장 일치하는 핸들러 메서드를 찾습니다.**

만약 찾지 못한다면 부모 클래스의 핸들러를 찾습니다. 우리 경우에는 `RuntimeException`이 되겠죠. 그리고 아무것도 찾지 못한다면, `hhandleAllUncaughtException()` 메서드가 최종적으로 예외를 처리할 것입니다.

이건 우리가 특정 컨트롤러 안에서 예외 처리를 다룰 때 도움이 될 것입니다. 그런데 똑같은 예외가 다른 컨트롤러에서 발생한다면 어떨까요? 이것을 어떻게 처리해야 할 까요? 우리는 같은 핸들러 메서드를 모든 컨트롤러에서 만들어야 할까요? 아니면 공통 핸들러 메서드를 가진 부모 클래스를 만든 다음 모든 클래스에서 이것을 상속해야할 까요?

운이 좋게도 이렇게 하지 않아도 됩니다. 스프링은 "controller advice"의 형태로 매우 우아한 해결책을 제공합니다.

## @ControllerAdvice
> ### 왜 ControllerAdvice 일까?
> 'Advice'라는 말은 AOP에서 나온 말입니다. 이 AOP는 Aspect-Oriented Programming이라고 메서드들에 횡단 코드로 주입(Advice)할 수 있는 기법이다. ControllerAdivce는 우리의 예외 처리 경우에서 중간에 끼어들어 컨트럴로 메서드들의 반환 값을 수정할 수 있게 해줍니다.

ControllerAdivce는 우리가 예외 핸들러를 우리 어플리케이션의 하나 이상 또는 모든 클래스에 적용할 수 있게 해줍니다.

```java
@ControllerAdvice
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {

  public static final String TRACE = "trace";

  @Value("${reflectoring.trace:false}")
  private boolean printStackTrace;

  @Override
  @ResponseStatus(HttpStatus.UNPROCESSABLE_ENTITY)
  protected ResponseEntity<Object> handleMethodArgumentNotValid(
      MethodArgumentNotValidException ex,
      HttpHeaders headers,
      HttpStatus status,
      WebRequest request
  ) {
      //Body omitted as it's similar to the method of same name
      // in ProductController example...  
      //.....
  }

  @ExceptionHandler(ItemNotFoundException.class)
  @ResponseStatus(HttpStatus.NOT_FOUND)
  public ResponseEntity<Object> handleItemNotFoundException(
      ItemNotFoundException itemNotFoundException, 
      WebRequest request
  ){
      //Body omitted as it's similar to the method of same name
      // in ProductController example...  
      //.....  
  }

  @ExceptionHandler(RuntimeException.class)
  @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
  public ResponseEntity<Object> handleAllUncaughtException(
      RuntimeException exception, 
      WebRequest request
  ){
      //Body omitted as it's similar to the method of same name
      // in ProductController example...  
      //.....
  }
  
  //....

  @Override
  public ResponseEntity<Object> handleExceptionInternal(
      Exception ex,
      Object body,
      HttpHeaders headers,
      HttpStatus status,
      WebRequest request) {

    return buildErrorResponse(ex,status,request);
  }

}
```
@ExceptionHanlder쳅터에서 보여준 코드와 거의 비슷하기 때문에 핸들러 메서드와 그 외 다른 메서드의 바디 부분은 생략되었습니다. 깃헙 레포지토리 [`GlobalExceptionHanlder` class](https://github.com/thombergs/code-examples/blob/master/spring-boot/exception-handling/src/main/java/io/reflectoring/exception/exception/GlobalExceptionHandler.java)에서 전체 코드를 보실 수 있습니다.

잠시후 우리가 얘기할 새로운 것들이 있습니다. 하나의 큰 차이점은 이 핸들러는 `ProductController`에 국한되지 않고 어플리케이션의 모든 컨트롤러에서 발생하는 에러를 처리합니다. 

만약 우리가 선택적으로 ControllerAdvice의 영역을 특정 컨트롤러나 패키지에 제한하고 적용하고 싶다면, 우리는 어노테이션에서 제공되는 프로퍼티를 사용할 수 있습니다.

- @ContollerAdvice("com.reflectoring.controller"): 우리는 패키지의 이름이나 패키지 이름 리스트를 어노테이션의 `value`나 `basePackages`에 전달 할 수 있습니다. ControllerAdvice는 해당 패키지 컨트롤러의 예외만 처리합니다.
- @ControllerAdvice(annotations = Advised.class): 오직 `@Advised` 어노테이션된 클래스의 예외만 ControllerAdvice에 의해 처리될 것 입니다.

[@ControllerAdvice 어노테이션 문서](https://www.javadoc.io/doc/org.springframework/spring-web/4.3.8.RELEASE/org/springframework/web/bind/annotation/ControllerAdvice.html)에서 다른 파라미터를 찾아보세요.

## ResponseEntityExceptionHanlder

`ResponseEntityExceptionHanlder`는 `ControllerAdvice`의 편리한 기본 클래스입니다. 이것은 내부 스프링 예외를 위해 예외처리를 제공합니다. 만약 우리가 이것을 상속하지 않는다면, 모든 예외는 `ModelAndView`를 반환하는 `DefaultHandlerExceptionResolver`로 리다이렉트 됩니다. 우리는 우리의 에러 응답을 만드는 것이 목표이기 때문에 이렇게 하지 않겠습니다.

보시다시피, 우리는 ResponseEntityExceptionHandler의 두가지 메서드를 오버라이드하고 있습니다.

- handleMethodArgumentNotValid(): `@ExceptionHandler` 쳅터에서는 이것을 바로 구현했습니다. 여기서는 단지 이것의 기능을 재정의했습니다.
- handleExceptionInternal(): ResponseEntityExceptionHandlerd의 모든 핸들러가 우리의 buildErrorReponse()처럼 ReponseEntity를 만들기 위해 이 함수를 사용합니다. 만약 이것을 오버라이드하지 않는다면 클라이언트는 응답 헤더에 Http 상태를 받을 수 있을 것입니다. 하지만 우리는 응답 바디에 Http 상태를 포함하기를 원하니까 이 메서드를 오버라이드했습니다.

> ## `NoHandlerFoundException`를 처리하는 것은 몇 가지 추가 단계가 더 필요합니다.
> 이 에외는 우리의 시스템에 존재하지 않는 API를 요청하는 경우 일어납니다. `ResponseEntityExceptionHandler` 클래스를 통해 이 핸들러를 정의해도 이 에외는 `DefaultHandlerExceptionResolver`로 리다이렉트 됩니다.
> 우리 advice로 리다이렉트 시키기 위해서는 우리 속성 파일에서 몇 가지 속성을 설정해줘야합니다: `spring.mvc.throw-exception-if-no-handler-found=true`과 `spring.web.resources.add-mappings=false`
> credit: [Stackoverflow user mengchengfeng](https://stackoverflow.com/questions/36733254/spring-boot-rest-how-to-configure-404-resource-not-found)

## `@ControllerAdvice`를 사용하면서 몇 가지 고려해야할 점
- 단순하게 유지하는 방법은 항상 프로젝트에 오직 하나의 advice 클래스를 가지는 것입니다. 어플리케이션에 모든 예외의 하나의 레포지토리를 갖는 것은 좋은 방법입니다. 많은 ControllerAdvice를 만드는 경우, 어떤 Controller에서 발생한 예외를 처리할지 명확하게 하기 위해 `basePackages`와 `value`를 사용하세요.
- 우리가 `@Order` 어노테이션으로 어노테이션을 추가하지 않는 한 **스프링은 어떤 순서로든 controller advice를 처리할 수 있습니다.** 그래서 만약 여러 개의 controller advice가 있다면 catch-all 핸들러를 작성할 때 주의하세요. 특히 어노테이션에 특정한 `basePackage`나 `value`를 지정하지 않았을 때 더욱 주의하세요

## 어떻게 스프링은 예외를 처리하나?

이제 우리는 스프링에서 우리가 예외를 처리할 수 있는 매커니즘에 대해 설명했습니다. 어떻게 스프링이 처리하는지와 언제 한 매커니즘이 다른 매커니즘보다 우선시 되는지 짧게 이해해봅시다. 우리가 우리 자신의 예외 처리 핸들러를 만들지 않은 경우 예외 처리의 과정을 추적한 아래에 플로우 차트를 봅시다.

![스프링 예외 처리 매커니즘](/assets/images/20220116_how-spring-process-exception.png)

## 결론
예외가 컨트롤러의 경계를 넘어설 때 이 예외는 JSON 포멧이나 HTML 웹 페이지 형태로 클라이언트로 도달할 예정입니다.

이 게시글에서는, 어떻게 스프링이 이러한 예외를 우리 사용자들을 위해 사용자 친화적인 결과로 바꿀 수 있는지와 우리가 원하는 형식으로 더 성형할 수 있는 어노테이션에 대해 보았습니다.

읽어주셔서 감사합니다!