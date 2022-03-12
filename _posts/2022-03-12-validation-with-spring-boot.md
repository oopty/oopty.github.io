---
title: 스프링 부트 검증 완벽 가이드
date: 2022-03-12 16:54:00 +09:00
categories: [java, spring]
tags: [java, spirng, validation]
image:
    src: /assets/images/스프링부트검증완벽가이드_FrontImage.jpeg
    alt: Validation
---

[스프링 부트 예외처리 완벽 가이드](/posts/complete-guide-to-exception-handling-in-spring-boot/)에 이어 스프링 부트에서 지원하는 검증 방법을 배우고 예시를 들며 설명하려고 합니다. [Validation with Spring Boot - the Complete Guid](https://reflectoring.io/bean-validation-with-spring-boot/)를 번역했습니다.

[Bean Validation](https://beanvalidation.org/)은 자바 생태계에서 검증 로직 구현의 사실상 기준이다. 스프링과 스프링 부트와 잘 통합된다.

하지만, 몇가지 주의할 점이 있다. 이번 튜토리얼에선 모든 검증 사용 사례와 스포츠 코드 예제를 통해 각각 살펴본다.

## Example Code
이 기사는 [Github](https://github.com/thombergs/code-examples/tree/master/spring-boot/validation)에 예제코드를 포함하고 있다.

## Spring Boot Validation Starter
스프링 부트의 검증은 `validation starter`로 사용할 수 있고 이것은 Gradle으로 프로젝트에 포함할 수 있다.
```gradle
implementation('org.springframework.boot:spring-boot-starter-validation')
```
`Spring Dependency Management Gradle Plugin`덕분에 버전을 명시할 필요는 없다. 만약 플러그인을 사용하고 있지 않다면, 최신 버전을 [여기](https://search.maven.org/search?q=g:org.springframework.boot%20AND%20a:spring-boot-starter-validation&core=gav)에서 찾을 수 있다.

## Bean Validation Basics
기본적으로, 빈 검증은 [검증 어노테이션](https://docs.jboss.org/hibernate/beanvalidation/spec/2.0/api/javax/validation/constraints/package-summary.html)으로 클래스 필드에 제한을 정의하므로써 작동된다.

## 일반적인 검증 어노테이션

몇가지 가장 일반적인 어노테이션은 아래와 같다

- `@NotNull`: 필드는 `null`이면 안된다.
- `@NotEmpty`: 리스트 필드는 비어있으면 안된다.
- `@NotBlank`: string 필드는 빈 스트링이 아니어야 한다. (즉, 적어도 글자 하나가 있어야 한다.)
- `@Min` and `@Max`: numeric 필드는 특정 값 이상거나 이하여야 한다.
- `@Pattern`: string 필드는 특정 정규표현식을 만족해야 한다.
- `@Email`: sring 필드는 이메일 형식이어야 한다.

```java
class Customer {

  @Email
  private String email;

  @NotBlank
  private String name;
  
  // ...
}
```

## Validator
오브젝트가 유효한 지 검사하기 위해서는 [Validator](https://docs.jboss.org/hibernate/beanvalidation/spec/2.0/api/javax/validation/Validator.html)에 전달한다.

```java
Set<ConstraintViolation<Input>> violations = validator.validate(customer);
if (!violations.isEmpty()) {
  throw new ConstraintViolationException(violations);
}
```

## @Validated and @Valid 
그렇지만 많은 경우에는 스프링이 검증을 해준다. 우리는 validator 오브젝트를 만들 필요가 없고 대신 스프링에게 유효한 오브젝트가 필요한 것을 알려주면 된다. 이것은 `@Validated`와 `@Valid` 어노테이션을 통해 작동된다.

`@Validated` 어노테이션은 class-level 어노테이션이다. 스프링은 메서드로 넘겨지는 파라미터를 검증해준다. 후에 더 자세히 다뤄볼 것이다.

메서드 파라미터와 필드에 `@Valid` 어노테이션을 붙힐 수 있다. 스프링은 메서드 파라미터와 필드를 검증해준다. 이것 또한 후에 더 자세히 살펴볼 것이다.

## Spring MVC Controller의 입력 거증
우리가 REST Contoller를 구현했고 클라이언트에게 오는 입력들을 검증하기 원한다고 하자. HTTP request로 오는 세가지 검증 사항들이 있다.

- Request Body
- Path variables
- query paramters

### Request Body 검증
POST와 PUT request에서는 request body에 json 페이로드를 넣는 경우가 많다. 스프링은 자동으로 Java 오브젝트로 바꿔준다. 자 이제, 우리는 이 Java 오브젝트를 검증하기를 원한다.

아래가 입력 페이로드 클래스이다.
```java
class Input {
    @Min(1)
    @Max(10)
    private int numberBetweenOneAndTen;

    @Pattern(regexp = "%[0-9]{1,3}\\.[0-9]{1,3}\\.[0-9]{1,3}\\.[0-9]{1,3}$")
    private String ipAddress;

    // ...
}
```

`numberBetweenOneAndTen`은 1이상 10이하인 수를 가질 수 있고 `ipAddress`는 IP Address 형식에 맞는 문자열을 가질 수 있다.(IP Address는 각 클래스마다 255이하의 숫자만 가능하고 이에 대한 검증은 `custome validator`를 통해 뒤에서 구현할 것이다)

HTTP request body를 검증하기 위해 REST Controller의 request body에 `@Valid` 어노테이션을 붙여준다.

```java
@RestController
class ValidateRequestBodyController {

    @PostMapping("/validateBody")
    ResponseEntity<String> validateBody(@Valid @RequestBody Input input) {
        return ResponseEntity.ok("valid");
    }
}
```

### Path Variables과 Request Parameters 검증
path variables과 request paramters 검증은 약간 다르게 동작한다.

복잡한 자바 오브젝트를 검증하지 않는다. path variables과 request parameters는 primitive type이나 `Integer`, `String`같은 counterpart object이기 때문이다.

위 처럼 클래스 필드에 어노테이션을 적용하는 대신, 스프링 컨트롤러에서 메서드 파라미터에 직접적으로 어노테이션을 사용한다. (아래 경우에는 `@Min`)

```java
@RequestController
@Validated
class ValidateParametersController {
    @GetMappling("/validatePathVariable/{id}")
    ReponseEntity<String> validatePathVariable(
        @PathVariable("id") @Min(5) int id) {
            return ResponseEntity.ok("valid");
        }
    
    @GetMapping("/validateRequestParamter")
    ResponseEntity<String> validateRequestParamter(
        @RequestParam("param") @Min(5) int param) {
            return ResponseEntity.ok("valid");
        }
}
```

스프링이 메서드 파라미터에 검증 어노테이션을 평가하기 위해 class-level에서 컨틀롤러에 `@Validated` 어노테이션을 추가했다.

`@Validated`는 메서드에 사용할 수 있어도 이 경우에서는 오직 class-level에서만 평가된다. (왜 method-level에서 사용가능한지는 후에 논의하겠다.)

request body 검증과는 다르게 검증이 실패하면 `MethodArgumentNotValidException`대신 `ConstraintValidationException`을 던진다. 스프링은 이 예외에 대해 디폴트 핸들러를 등록하지 않는다. 따라서 기본적으로 HTTP status 500(Internal Server Error)을 날린다.

만약 HTTP status 400(Bad Request)을 날리고 싶다면, 우리는 커스텀 예외 핸들러를 컨트롤러에 추가할 수 있다.

```java
@RestController
@Validated
class ValidateParametersController{
    //request mapping method omitted

    @ExceptionHandler(ConstraintViolationException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    ResponseEntity<String> handleContraintViolationException(ConstraintViolationException e) {
        return new ResponseEntity<>("not valid due to validation error: " + e.getMessage(), HttpStatus.BAD_REQUEST);
    }
}
```

튜토리얼에서 어떻게 모든 실패 검증에 대한 설명을 포함한 구조화된 에러 응답을 보내는지 볼것이다.

통합 테스트로 검증 동작을 실증할 수 있다.
```java
@ExtendWith(SpringExtension.class)
@WebMvcTest(controllers = ValidateParamtersController.class)
class ValidateParamtersConrollerTest {

    @Autowired
    private MockMvc mvc;

    @Test
    void whenPathVariableIsInvalid_thenReturnStatus400() throws Exception {
        mvc.perform(get("/validatePathVariable/3"))
            .andExpect(status().isBadRequest());
    }

    @Test
    void whenRequestParameterIsInvalid_thenReturnsStatus400() throws Exception {
    mvc.perform(get("/validateRequestParameter")
            .param("param", "3"))
            .andExpect(status().isBadRequest());
}
```

## 스프링 Service 메서드의 입력 검증
컨트롤러 레벨에서 입력을 검증하는 대신, 어떤 스프링 컴포넌트에 입력을 검증할 수 있다. 그러기 위해, `@Validated`와 `@Valid` 어노테이션을 조합해서 사용한다.

```java
@Service
@Validated
class ValidatingService{

    void validateInput(@Valid Input input) {
        //do something
    }

}
```

다시 말하지만, `@Validated` 어노테이션은 class-level에서만 평가되기 때문에 메서드에 붙이지 않는다.

아래는 검증 동작 실증이다.

```java
@ExtendWith(SpringExtension.class)
@SpringBootTest
class ValidatingServiceTest {

    @Autowired
    private ValidatingService service;

    @Test
    void whenINputIsInvalid_thenThrowsException() {
        Input input = invalidInput();

        assertThrows(ConstraintViolationException.class, () -> {
            service.validateInput(input)
        });
    }
}
```

## JPA Entities 검증

검증을 위한 마지막 방어선은 persistence layer이다. Spring Data는 빈 검증을 바로 사용할 수 있는 Hibernate를 아래에 사용하고 있다.

> **Persistence layer에서 검증을 하는게 옳은 방법 일까?**  
> 
> 우리는 일반적으로 persisten layer처럼 늦게 검증을 하지 않는다. 왜냐하면 위에 비지니스 코드는 유효하지 않은 오브젝트로 동작할 수 있다는 말이다. 이것은 예기치 못한 에러를 만들 수 있다.
{: .prompt-tip }

`Input` 클래스를 데이터베이스에 저장해보자. 첫번째로 필수적인 JPA 어노테이션 `@Entity`와 ID 필드를 추가한다.
```java
@Entity
public class Input {

    @Id
    @GeneratedValue
    private Long id;

    @Min(1)
    @Max(10)
    private int numberBetweenOneAndTen;

    @Pattern(regexp = "^[0-9]{1,3}\\.[0-9]{1,3}\\.[0-9]{1,3}\\.[0-9]{1,3}$")
    private String ipAddress;

    // ...
}
```

그리고나서 Spring Data 레포지토리를 만든다.

```java
public interface ValidatingRepository extends CrudRepository<Input, Long> {}
```

기본적으로, 우리가 유효하지 않은 `Input` 오브젝트를 저장할 때마다 `ContraintViolationException`을 얻는다. 아래 통합 테스트를 보자.

```java
@ExtendWith(SpringExtension.class)
@DataJpaTest
class ValidatingRepositoryTest {

    @Autowired
    private ValidatingRepository repository;

    @Autowired
    private EntityManager entityManager;

    @Test
    void whenInputIsInvalid_thenThrowsException() {
    Input input = invalidInput();

    assertThrows(ConstraintViolationException.class, () -> {
        repository.save(input);
        entityManager.flush();
    });
    }

}
```
Spring Data 레포지토리 테스트에 대해서는 [여기](https://reflectoring.io/spring-boot-data-jpa-test/)에서 더 자세히 볼 수 있다.

빈 검증은 `EntityManager`가 플러쉬될때 마다 Hibernate에 의해 트리거된다. Hibernate는 특정 환경에서 `EntityManager`를 플러시하지만 이 통합 테스트에서는 직접 해줘야 한다.

만약 어떤 이유에서 Spring Data 레포지토리에서 빈 검증을 비활성하고 싶다면, Spring Boot 속성 중 `spring.jpa.properties.javax.persistence.validation.mode`을 `none`으로 설정한다.

## 커스텀 Validator

사용가능한 contrain 어노테이션이 우리 사용 사례에 충분치 않다면, 우리 스스로 만들어야 한다.

`Input` 클래스에서는 정규표현식을 통해 IP Address를 검증하지만 완벽하지 않다. 255 이상인 숫자도 허용하기 때문이다. (다시말해, "111.111.111.333"이 유효하다)

정규 표현식 대신 자바에서 검증하는 validator를 구현하므로 해결해보자. (더 복잡한 정규표현식을 사용해 해결할 수도 있다.)

첫번째로, 커스텀 contraint 어노테이션 `Ipaddress`를 생성한다.
```java
@Target({ FIELD })
@Retention(RUNTIME)
@Contraint(validateBy = IpAddressValidator.class)
@Documented
public @interface IpAddress {

    String message() default "{IpAddress.invalid}";

    Class<?>[] groups() default { };

    Class<? extends Payload>[] payload() default { };

}
```
커스텀 contraint 어노테이션은 아래와 같은 것들이 필요하다.

 - 파라미터 `message`, 검증 시패시 메시지를 해결하는데 사용되는 `ValidationMessage.properties`의 키를 가르킨다.
 - 파라미터 `groups`, 어느 환경에서 이 검증이 동작될 지 정의하게 해준다.
 - 파라미터 `payload`, 검증과 함께 넘겨지는 페이로드를 정의하게 해준다.
 - 그리고 `@Contraint` 어노테이션은 `ContrainValidator` 인터페이스의 구현체를 가르킨다.

Validator 구현체는 아래와 같다.
```java
class IpAddressValidator implements ConstraintValidator<IpAddress, String> {

  @Override
  public boolean isValid(String value, ConstraintValidatorContext context) {
    Pattern pattern = 
      Pattern.compile("^([0-9]{1,3})\\.([0-9]{1,3})\\.([0-9]{1,3})\\.([0-9]{1,3})$");
    Matcher matcher = pattern.matcher(value);
    try {
      if (!matcher.matches()) {
        return false;
      } else {
        for (int i = 1; i <= 4; i++) {
          int octet = Integer.valueOf(matcher.group(i));
          if (octet > 255) {
            return false;
          }
        }
        return true;
      }
    } catch (Exception e) {
      return false;
    }
  }
}
```

constraint 어노테이션과 같이 `@IpAddress` 어노테이션을 사용할 수 있다.
```java
class InputWithCustomValidator {

  @IpAddress
  private String ipAddress;
  
  // ...

}
```

## 프로그래밍적으로 검증
스프링 built-in 빈 검증 지원에 의존하기 보다 프로그래밍적으로 검증르 하고싶을 때가 있다. 이 경우 빈 검증 API를 직접적으로 사용할 수 있다.

`Validator`을 직접 만들고 검증하기 위해 실행한다.
```java
class ProgrammaticallyValidatingService {
  
    void validateInput(Input input) {
        ValidatorFactory factory = Validation.buildDefaultValidatorFactory();
        Validator validator = factory.getValidator();
        Set<ConstraintViolation<Input>> violations = validator.validate(input);
        if (!violations.isEmpty()) {
            throw new ConstraintViolationException(violations);
        }
    }
  
}
```

스프링 지원이 전혀 필요하지 않다.

그렇지만, 스프링은 미리 설정된 `Validator`를 지원한다. 우리는 직접 만드는 대신 우리 서비스에 이 객체를 주입하고 사용할 수 있다.

```java
@Service
class ProgrammaticallyValidatingService {

  private Validator validator;

  ProgrammaticallyValidatingService(Validator validator) {
    this.validator = validator;
  }

  void validateInputWithInjectedValidator(Input input) {
    Set<ConstraintViolation<Input>> violations = validator.validate(input);
    if (!violations.isEmpty()) {
      throw new ConstraintViolationException(violations);
    }
  }
}
```

이 서비스가 스프링에 의해 초기화될 때, 생성자에 의해 주입된 `Validator`를 가지게 될 것이다.

다음 코드는 두가지 메서드가 기대하는 대로 동작되나 확인한다.
```java
@ExtendWith(SpringExtension.class)
@SpringBootTest
class ProgrammaticallyValidatingServiceTest {

  @Autowired
  private ProgrammaticallyValidatingService service;

  @Test
  void whenInputIsInvalid_thenThrowsException(){
    Input input = invalidInput();

    assertThrows(ConstraintViolationException.class, () -> {
      service.validateInput(input);
    });
  }

  @Test
  void givenInjectedValidator_whenInputIsInvalid_thenThrowsException(){
    Input input = invalidInput();

    assertThrows(ConstraintViolationException.class, () -> {
      service.validateInputWithInjectedValidator(input);
    });
  }

}
```

## Validation groups를 이용해 다른 사용 유형에 따라 다르게 검증하기

종종, 어떤 객체는 여러 다른 사용사례들에서 공유한다.

CRUD 동작에 대해 얘기해보자, 예를 들어 Create과 Update는 대게 같은 객체를 입력으로 사용할 것이다. 그렇지만 다른 환경에서 다르게 작동되야 하는 검증이 있다.

 - 오직 "Create"일 때만
 - 오직 "Update"일 때만
 - 두 가지 경우 일 때

검증 룰을 구현하게 해주는 빈 검증 기능은 "Validation Groups"라고 불린다.

우리는 이미 커스텀 어노테이션을 구현하면서 contraint 어노테이션은 `groups`필드를 가지고 있어야 하는 것을 보았다. 이것은 트리거되야 하는 Validation Group을 정의한 클래스를 넘기는데 사용된다.

CRUD 예제에서는 `OnCreate`과 `OnUpdate` 마커 인터페이스를 정의한다.

```java
interface OnCreate {}

interface OnUpdate {}
```

마커 인터페이스는 contraint 어노테이션에 아래와 같이 사용할 수 있다.

```java
class InputWithGroups {

    @Null(groups = OnCreate.class)
    @NotNull(groups = OnUpdate.class)
    private Long id;

    // ...
  
}
```

이것은 ID가 "Create"으로 사용할 때는 비어있어야 하고 "Update"일 때는 비어있으면 안된다는 것을 말한다.

스프링은 Validation Groups를 `@Validated` 어노테이션으로 제공한다.

```java
@Service
@Validated
class ValidatingServiceWithGroups {

    @Validated(OnCreate.class)
    void validateForCreate(@Valid InputWithGroups input){
        // do something
    }

    @Validated(OnUpdate.class)
    void validateForUpdate(@Valid InputWithGroups input){
        // do something
    }

}
```

클래스에도 `@Validated` 어노테이션이 붙은 것을 확인해라. 어느 Validation Groups가 활성화 되었는지 정의하기 위해 method-level에도 붙여야 한다.

아래는 테스트다.
```java
@ExtendWith(SpringExtension.class)
@SpringBootTest
class ValidatingServiceWithGroupsTest {

    @Autowired
    private ValidatingServiceWithGroups service;

    @Test
    void whenInputIsInvalidForCreate_thenThrowsException() {
        InputWithGroups input = validInput();
        input.setId(42L);

        assertThrows(ConstraintViolationException.class, () -> {
            service.validateForCreate(input);
    });
    }

    @Test
    void whenInputIsInvalidForUpdate_thenThrowsException() {
        InputWithGroups input = validInput();
        input.setId(null);

        assertThrows(ConstraintViolationException.class, () -> {
            service.validateForUpdate(input);
        });
    }

}
```

> **Validation Groups 조심해서 사용**
>
> Validation Groups를 사용하는 것은 관점이 섞이기 때문에 쉽게 안티 패턴이 될 수 있다. Validation Groups와 함께 사용되는 엔티티는 모든 사용 케이스에 대해 검증 규칙을 알아야 한다. [Bean Validation anti-patterns](https://reflectoring.io/bean-validation-anti-patterns/#anti-pattern-3-using-validation-groups-for-use-case-validations) 토픽에서 더 자세히 설명한다.
{: .prompt-tip }

## 검증 에러 핸들링

검증이 실패하면 의미있는 에러메시지를 클라이언트에게 전달하고 싶다. 각각 검증 실패에 대해 에러메시지를 포함한 데이터를 전달해야한다.

첫째로, 우리는 데이터 구조체를 정의할 것이다. `ValidationErrorResponse`라고 명명할 것이고 이건 `Violation` 오브젝트 리스트를 포함한다.

```java
public class ValidationErrorResponse {

    private List<Violation> violations = new ArrayList<>();

    // ...
}

public class Violation {

    private final String fieldName;

    private final String message;

    // ...
}
```

그러고나서 controller-level까지 올라오는 모든 `ConstraintViolationExceptions`를 다룰 전역 `@ControllerAdvice`를 만든다. request body에서 일어나는 검증 에러도 잡기 위해 `MethodArgumentNotValidExceptions`에 대한 핸들러도 만든다.

```java
@ControllerAdvice
class ErrorHandlingControllerAdvice {

  @ExceptionHandler(ConstraintViolationException.class)
  @ResponseStatus(HttpStatus.BAD_REQUEST)
  @ResponseBody
  ValidationErrorResponse onConstraintValidationException(
      ConstraintViolationException e) {
    ValidationErrorResponse error = new ValidationErrorResponse();
    for (ConstraintViolation violation : e.getConstraintViolations()) {
      error.getViolations().add(
        new Violation(violation.getPropertyPath().toString(), violation.getMessage()));
    }
    return error;
  }

  @ExceptionHandler(MethodArgumentNotValidException.class)
  @ResponseStatus(HttpStatus.BAD_REQUEST)
  @ResponseBody
  ValidationErrorResponse onMethodArgumentNotValidException(
      MethodArgumentNotValidException e) {
    ValidationErrorResponse error = new ValidationErrorResponse();
    for (FieldError fieldError : e.getBindingResult().getFieldErrors()) {
      error.getViolations().add(
        new Violation(fieldError.getField(), fieldError.getDefaultMessage()));
    }
    return error;
  }

}
```

## 결론
이 튜토리얼에서 스프링 부트로 어플리케이션을 구축할 때 필요한 주요 검증 기능들을 훑어봤다. 예제 코드도 준비되어 있으니, 직접 타이핑해보자