---
title: Spring Boot와 @WebMVCTest로 MVC Controller를 테스트 하는 방법
date: 2022-07-02 20:14:00 +09:00
categories: [spring]
tags: [spring, relectoring, test, mvc, controller, spring boot, fluent API, java, interation test]
---
이 글은 [Testing MVC Web Controllers with Spring Boot and @WebMvcTest](https://reflectoring.io/spring-boot-web-controller-test/)을 해석하고 정리한 글입니다.

이번 글에서는 Controller에 대해 살펴볼 것이다. 먼저 책임 전체를 수용하는 테스트를 작성하기 위해 web Controller가 무엇을 하는지 탐색할 것이다. 그러고 각각 책임을 수용하는 테스트를 작성하는 방법을 찾을 것이다. 이러한 책임이 테스트 되어야지만 Controller가 운영 환경에서 예상대로 동작하는지 확인할 수 있다.

이 글은 [예제](https://github.com/thombergs/code-examples/tree/master/spring-boot/spring-boot-testing)를 제공한다.


## 의존성

Junit Jupiter(Junit5)를 사용할 것이고 모킹을 위해 Mockito, 어썰션을 위해 AssertJ와 보일러 플레이트 코드를 줄이기 위해 Lombok을 사용할 거다.

```gradle
dependencies {
  compile('org.springframework.boot:spring-boot-starter-web')
  compileOnly('org.projectlombok:lombok')
  testCompile('org.springframework.boot:spring-boot-starter-test')
  testCompile 'org.junit.jupiter:junit-jupiter-engine:5.2.0'
  testCompile('org.mockito:mockito-junit-jupiter:2.23.0')
}
```
AssertJ와 Mockito는 `spring-boot-starter-test` 의존성에서 자연스럽게 들어온다.

## Web Controller의 책임

아래의 전형적인 REST Controller를 보자
```java
@RestController
@RequiredArgsConstructor
class RegisterRestController {
  private final RegisterUseCase registerUseCase;

  @PostMapping("/forums/{forumId}/register")
  UserResource register(
          @PathVariable("forumId") Long forumId,
          @Valid @RequestBody UserResource userResource,
          @RequestParam("sendWelcomeMail") boolean sendWelcomeMail) {

    User user = new User(
            userResource.getName(),
            userResource.getEmail());
    Long userId = registerUseCase.registerUser(user, sendWelcomeMail);

    return new UserResource(
            userId,
            user.getName(),
            user.getEmail());
  }

}
```
URL을 정의하고 POST 매핑을 해주는 `PostMapping` HTTP request를 Deserialze해주는 `@PathVariable`, `@RequestBody`등 Spring이 지원해주는 많은 기능들이 있다. Controller를 테스트할 때 이런 것 까지 테스트를 해줘야 할까? 우리가 기대하는 Controller의 책임은 아래와 같다.

1. HTTP Request 요청 수신
2. Input Deserialization
3. 요청 검증
4. 비지니스 로직 호출
5. Output Serialization
6. 예외 변환

4번을 제외한 나머지 로직은 스프링이 지원해주는 부분이다. 하지만 우리가 Controller에 기대하는 책임이어서 이를 테스트하지 않는다면 운영환경에서 기대하는 대로 동작을 안할 수 있다. 아쉽게도 Unit Test로는 Spring이 지원해주는 부분을 테스트할 수 없다. 따라서 우리는 Spring의 HTTP 지원과 Controller 코드 사이에서 Integration Test를 해야한다. 

## @WebMvcTest로 Controller 책임을 검증하기
Spring Boot는 `@WebMvcTest` 어노세이션을 제공하고 web controller를 테스팅하는데 필요한 빈들만 포함한 application context을 뛰운다.
```java
@ExtendWith(SpringExtension.class)
@WebMvcTest(controllers = RegisterRestController.class)
class RegisterRestControllerTest {
  @Autowired
  private MockMvc mockMvc;

  @Autowired
  private ObjectMapper objectMapper;

  @MockBean
  private RegisterUseCase registerUserCase;

  @Test
  void whenValidInput_thenReturns200() throws Exception {
    mockMvc.perform(...);
  }

}
```
> ### @ExtendWith
> 
> Spring Boot 2.1부터는 `SpringExtensio`을 로드할 필요가 없다. 왜냐하면 `@DataJpaTest`, `@WebMvcTest`와 `SpringBootTest`와 같은 어노테이션에 이미 메타데이터로 포함되어 있기 때문이다.

이제 `@Autowired`로 모든 필요한 Beans를 가져올 수 있다.

또한 `@MockBean`을 통해 비지니스 로직을 모킹할 수 있다. 우리는 HTTP layer와 controller사이를 통합해서 테스트 하고 싶은거지 비지니스 로직까지 테스트하고 싶은 것은 아니다. 그래서 `@MockBean` 어노테이션을 통해 applicaiton context안에 있는 빈을 같은 타입의 mock bean으로 교체해 주었다.

`@MockBean`에 대해 더 자세한 글은 [이것](https://reflectoring.io/spring-boot-mock/)을 읽어라

> ### @WebMvcTest의 controllers 파라미터의 유무 차이
> 
> controller 파라미터를 세팅함으로서 Spring Boot에게 해당 컨트롤러와 필수적인 Spring Web MVC Bean들만 가져오게 한다. 필요할 수도 있는 다른 Bean들은 별도로 포함시키거나 `@MockBean`으로 모킹시켜야 한다.
> 만약 controller 파라미터를 사용하지 않는다면, 모든 controller들을 application context에 포함시키고 그러므로 모든 Bean들을 가져와야 하니까 테스트 셋업 과정이 복잡하다. 하지만 런타임시 모든 컨트롤러가 같은 application context를 사용하게 되니까 효과적일 수 있다.
> 나는 controller 테스트를 최대한 좁은 applicaiton context로 제한하는 편이다. 비록 Spring Boot가 테스트 마다 새로운 application context를 만들더라도 이렇게 한다.

controller의 책임들을 하나씩 보고 최고의 통합 테스트를 작성하기 위해 MockMVC를 사용해서 각각을 검증할 수 있는지 확인하자.

### 1. HTTP Request 매핑

controller가 특정 HTTP request에 응답하는지 확인하는 것은 매우 간단합니다. 간단히 `MockMVC`의 `perform()` 메서드를 호출하면 됩니다.

```java
mockMvc.perform(post("/forums/42/register")
          .contentType("application/json")
          .andExpect(status().isOk()))
```

우리는 특정 URL에 controller가 응답하는지, 정확한 HTTP method와 content type을 검증한다. 우리가 작성한 controller는 예제에서 설정한 것과 다른 HTTP request에 대해서는 거부한다.

이 테스트는 아직 우리가 input parameter를 넣어주지 않았기 때문에 실패한다.

HTTP request Matcher의 옵션에 대해 더 자세히 알고 싶으면 [Javadoc](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/test/web/servlet/request/MockHttpServletRequestBuilder.html)을 살펴보아라

### 2. Input Deserialization
Java 객체로 성공적으로 Deserialize되는지 확인하려면 Test 코드에서 Input을 제공해줘야 한다. Input은 Reqeust Body의 JSON 형태일 수 있고 URL path안에 있을 수 있고(`@PathVariable`) HTTP request의 파라미터일 수 있다.(`@RequestParam`)

```java
@Test
void whenValidInput_thenReturns200() throws Exception {
  UserResource user = new UserResource("Zaphod", "zaphod@galaxy.net");

  mockMvc.perform(post("/forums/{forumId}/register", 42L)
                .contetType("application/json")
                .param("sendWelComeMail", "true")
                .content(objectMapper.writeValueAsString(user)))
                .andExpect(status().isOk())
}
```
`forumId`는 path variable로, `sendWelcomeMail`은 request parameter로 user정보는 request body로 넘겨주었다. request body는 Spring Boot가 제공하는 objectMapper로 객체를 JSON 형식으로 변환해서 넘겨주었다.

만약 test가 성공했다면, 우리는 `register()` 메서드가 이러한 파라미터와 자바 객체를 받고 성공적으로 파싱을 수행했구나라고 생각할 수 있다.

### 3. 요청 검증

`UserResource` 변수에 `@NotNull` 어노테이션을 적용해 보자.
```java
@Value
public class UserResource {

  @NotNull
  private final String name;

  @NotNull
  private final String email;
  
}
```

Bean Validation은 메서드 파라미터에 `@Valid` 어노테이션을 붙혀주면 자동으로 트리거 된다. 따라서 검증이 성공한 경우에 대한 테스트는 우리가 위에 작성한 테스트 코드로 충분하다.

만약 실패하는 경우를 테스트하고 싶다면 검증을 실패하는 UserResource JSON 객체를 controller에 전달하면 된다. 그리고 나서 HTTP status 400을 검증하면 된다.

```java
@Test
void whenNullValue_thenReturns400() throws Exception {
  UserResource user = new UserResource(null, "zaphod@galaxy.net");
  
  mockMvc.perform(post("/forums/{forumId}/register", 42L)
      ...
      .content(objectMapper.writeValueAsString(user)))
      .andExpect(status().isBadRequest());
}
```
어플리케이션의 검증이 얼마나 중요한지에 따라, 검증에 실패하는 테스트를 모든 가능한 경우를 만들어야 할 수도 있다. 이것은 빠르게 테스트 케이스가 늘어날 수 있어서 검증 테스트 방법에 대해 팀과 상의를 해본후 결정해야한다.

### 4. 비지니스 로직 호출

다음으로, 비지니스 로직이 기대하는대로 호출하는지 검증해보자. 이번 경우에는 비지니스 로직이 `RegiesterUseCase` 인터페이스에 의해 제공되고 이 함수는 `User` 객체와 `boolean` 변수를 기대한다.

```java
interface RegisterUseCase {
  Long registerUser(User user, boolean sendWelcomeMail);
}
```
우리는 controller가 User 객체인 UserResource를 `regiesterUser()` 메서드로 전달하기를 기대한다.

이것을 검증하기 위해, application context에 `@MockBean`을 사용해 모킹된 `RegisterUseCase`를 사용할 수 있다.

```java
@Test
void whenValidInput_thenMapsToBusinessModel() throws Exception {
  UserResource user = new UserResource("Zaphod", "zaphod@galaxy.net");
  mockMvc.perform(...);

  ArgumentCaptor<User> userCaptor = ArgumentCaptor.forClass(User.class);
  verify(registerUseCase, times(1)).registerUser(userCaptor.capture(), eq(true));
  assertThat(userCaptor.getValue().getName()).isEqualTo("Zaphod");
  assertThat(userCaptor.getValue().getEmail()).isEqualTo("zaphod@galaxy.net");
}
```

controller의 함수를 호출한 뒤, `ArgumentCaptor`를 이용해 `RegisterUseCase.registerUser()`에 전달되는 `User`객체를 캡쳐한다. 그리고 기대하는 값을 가지고 있는지 확인한다.

`verify` 호출이 `regiesterUser()`가 정확히 한번 호출하는지 테스트한다.

우리는 User 객체를 검증하는데 많은 assertion을 사용했다. [커스텀 Mockito assertion methods](https://reflectoring.io/unit-testing-spring-boot/#creating-readable-assertions-with-assertj)를 사용해서 더 좋은 가독성을 만들 수 있다.


### 5. Output Serialization
비지니스 로직이 수행된 후에는 controller가 JSON 형식으로 포함된 결과를 HTTP response에 매핑하는 것을 기대한다. 예제의 경우 HTTP response body에 검증된 UserResource 객체를 JSON 형태로 포함하는지 테스트하면 된다.

```java
@Test
void whenValidInput_thenReturnsUserResource() throws Exception {
  MvcResult mvcResult = mockMvc.perform(...)
      ...
      .andReturn();

  UserResource expectedResponseBody = ...;
  String actualResponseBody = mvcResult.getResponse().getContentAsString();
  
  assertThat(actualResponseBody).isEqualToIgnoringWhitespace(
              objectMapper.writeValueAsString(expectedResponseBody));
}
```

response body를 확인하기 위해, `andReturn()` 메서드를 이용해 MvcResult 타입의 변수에 HTTP 결과를 저장했다.

JSON 결과를 response body에서 읽어 기대값과 `isEqualToIgnoringWhitespace()`를 사용해 비교해 주었다. 기대값은 Java 객체를 `ObjectMapper`를 사용해서 JSON 형태로 만들 수 있다.

또한 우리는 이것은 커스텀 `ResultMatcher`를 통해 좀 더 가동석을 높일 수 있다. 뒤에 섹션에서 살펴볼 것이다.

### 6. 예외 변환
보통, 예외가 일어난다면, controller는 특정 HTTP status를 반환한다. 예를 들어 Request에 문제가 있다면 400이 반환되고 서버 에러가 발생하면 500이 발생한다.

스프링이 대부분의 경우 기본으로 처리한다. 하지만 만약 우리가 커스텀 예외 처리를 하고 있다면 해당 부분을 테스트하기를 원할 것이다. 우리가 JSON 형태의 유효하지 않은 field 이름과 에러 메시지를 반환한다고 하자. 아래와 같이 `@ControllerAdvice`를 만들었다.

```java
@ControllerAdvice
class ControllerExceptionHandler {
  @ResponseStatus(HttpStatus.BAD_REQUEST)
  @ExceptionHandler(MethodArgumentNotValidException.class)
  @ResponseBody
  ErrorResult handleMethodArgumentNotValidException(MethodArgumentNotValidException e) {
    ErrorResult errorResult = new ErrorResult();
    for (FieldError fieldError : e.getBindingResult().getFieldErrors()) {
      errorResult.getFieldErrors()
              .add(new FieldValidationError(fieldError.getField(), 
                  fieldError.getDefaultMessage()));
    }
    return errorResult;
  }

  @Getter
  @NoArgsConstructor
  static class ErrorResult {
    private final List<FieldValidationError> fieldErrors = new ArrayList<>();
    ErrorResult(String field, String message){
      this.fieldErrors.add(new FieldValidationError(field, message));
    }
  }

  @Getter
  @AllArgsConstructor
  static class FieldValidationError {
    private String field;
    private String message;
  }
  
}
```

만약 Bean 검증이 실패되면 스프링은 `MethodArgumentNotValidException`을 던질 것이다. 우리는 `FieldError` 객체를 `ErrorResult` 객체로 매핑하면서 예외를 처리할 것이다. 이 예외 핸들러는 모든 Controller한테 적용된다.

이를 검증하기 위해, 객체 실패 검증 테스트를 확장한다.
```java
@Test
void whenNullValue_thenReturns400AndErrorResult() throws Exception {
  UserResource user = new UserResource(null, "zaphod@galaxy.net");

  MvcResult mvcResult = mockMvc.perform(...)
          .contentType("application/json")
          .param("sendWelcomeMail", "true")
          .content(objectMapper.writeValueAsString(user)))
          .andExpect(status().isBadRequest())
          .andReturn();

  ErrorResult expectedErrorResponse = new ErrorResult("name", "must not be null");
  String actualResponseBody = 
      mvcResult.getResponse().getContentAsString();
  String expectedResponseBody = 
      objectMapper.writeValueAsString(expectedErrorResponse);
  assertThat(actualResponseBody)
      .isEqualToIgnoringWhitespace(expectedResponseBody);
}
```
Reponse body에서 JSON 결과를 읽고 기대값과 비교한다. HTTP status가 400인지도 확인한다.

훨씬 더 가독성이 좋은 방식으로 작성될 수 있다. 아래 챕터을 봐보자

## 커스텀 ResultMatchers 만들기
특정 assertion은 쓰기 어려울 뿐더러 읽기도 어렵다. 지난 두 예제 코드에서 살펴봤듯이 HTTP response에서 JSON String을 가져와 기대값과 비교하는 코드는 매우 길다.

운이 좋게도, MockMvc의 fluent API를 이용해 커스텀 `ResultMatchers`을 만들 수 있다.
### JSON 결과 매칭하기
HTTP response body가 자바 객체의 JSON 형식을 포함하고 있는지 검증하기 위해 아래 코드를 사용하는게 좋아 보이지 않나?

```java
@Test
void whenValidInput_thenReturnsUserResource_withFluentApi() throws Exception {
  UserResource user = ...;
  UserResource expected = ...;

  mockMvc.perform(...)
      ...
      .andExpect(responseBody().containsObjectAsJson(expected, UserResource.class));
}
```
우리는 이제 수동으로 JSON 결과를 비교할 필요가 없다. 그리고 훨씬 가독성이 높아진다. 이 코드는 너무 자명해서 여기서 설명을 그만하겠다.

```java
public class ResponesBodyMatchers {
  private ObjectMapper objectMapper = new ObjectMapper();

  public <T> ResultMatcher containsObjectAsJson(
    Object expectedObject,
    Class<T> targetClass
  ) {
    return mvcResult -> {
      String json = mvcResult.getResponse().getContentAsString();
      T actualObject = objectMapper.readvalue(json, targetClass);
      assertThat(actualObject).isEqaulToComparingFieldByField(expectedObejct);
    };
  }

  static ResponseBodyMatchers responseBody() {
    return new ResponseBodyMatchers();
  }
}
```
정적 메서드 `responseBody()`는 fluent API의 진입점을 제공한다. 이것은 HTTP resonse body 에서 JSON을 파싱하는 `ResultMatcher`을 반환하고 넘겨받은 기대값과 필드 by 필드로 검증한다.

### 기대하는 검증 에러 매칭하기

우리는 예외 처리 테스트를 좀 더 간단하게 하기 위해 한 단계 더 나아갈 수 있다. 에러 메세지의 JSO 결과를 검증하기 위해 4 라인이 쓰였다. 아래와 같이 1 라인으로 대체할 수 있다.

```java
@Test
void whenNullValue_thenReturns400AndErrorResult_withFluentApi() throws Exception {
  UserResource user = new UserResource(null, "zaphod@galaxy.net");

  mockMvc.perform(...)
      ...
      .content(objectMapper.writeValueAsString(user)))
      .andExpect(status().isBadRequest())
      .andExpect(responseBody().containsError("name", "must not be null"));
}
```

이런 fluent API를 쓰기위해, `ResponseBodyMatchers`에 `containsErrorMessageForField()`를 추가해야 한다.

```java
public class ResponseBodyMatchers {
  private ObjectMapper objectMapper = new ObjectMapper();

  public ResultMatcher containsError(
        String expectedFieldName, 
        String expectedMessage) {
    return mvcResult -> {
      String json = mvcResult.getResponse().getContentAsString();
      ErrorResult errorResult = objectMapper.readValue(json, ErrorResult.class);
      List<FieldValidationError> fieldErrors = errorResult.getFieldErrors().stream()
              .filter(fieldError -> fieldError.getField().equals(expectedFieldName))
              .filter(fieldError -> fieldError.getMessage().equals(expectedMessage))
              .collect(Collectors.toList());

      assertThat(fieldErrors)
              .hasSize(1)
              .withFailMessage("expecting exactly 1 error message"
                         + "with field name '%s' and message '%s'",
                      expectedFieldName,
                      expectedMessage);
    };
  }

  static ResponseBodyMatchers responseBody() {
    return new ResponseBodyMatchers();
  }
}
```

모든 더티 코드는 이 helper class에 숨겨져 있고 우리는 즐겁게 integration test에서 클린 코드를 작성할 수 있다.