# Create User Endpoint

## Project Recap

We have the User CRUD ready to be served externally. We have refactored the code in the service layer to be independent of data operations. Our tests can use a bit of refactoring but as it is having 100% coverage at the moment, we can leave that for now and work on adding functionality. 100% coverage is a bit much for any project. Let's see how long we can stay at this level.

## Web controllers

If you run the `LearnSpringApplication` now, you'll notice that it starts and exits correctly. But, what good is an application if it does not keep running. For the application to keep running, we must configure a delivery mechanism for the application. That's where web controllers come in. This configures the application to run on a port in the system. Let's begin by configuring the application and see if it keeps running.

Let's begin by adding the spring boot web dependency in our pom file.

```markup
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

### Maven commands

`mvn clean install` : Builds the jar for the project to be used to launch the application

`mvn spring-boot:run` : Launches the spring boot application.

From here on now, If I mention to launch the application, building the jar should be implicit as we should be using the latest code base, viz., it will have both the commands in the order.

After building the jar, run the application and you'll see that the application is not exiting. The console will have messages like this.

```markup
2020-12-28 20:33:04.083  INFO 77669 --- [           main] c.a.g.learn.LearnSpringApplication       : Starting LearnSpringApplication using Java 10 on Atuls-MacBook-Pro.local with PID 77669 (rest-using-spring-boot/user-service/v8/target/classes started by creations in /Users/creations/IdeaProjects/rest-using-spring-boot)
2020-12-28 20:33:04.085  INFO 77669 --- [           main] c.a.g.learn.LearnSpringApplication       : No active profile set, falling back to default profiles: default
2020-12-28 20:33:04.847  INFO 77669 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8080 (http)
2020-12-28 20:33:04.855  INFO 77669 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2020-12-28 20:33:04.855  INFO 77669 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet engine: [Apache Tomcat/9.0.41]
2020-12-28 20:33:04.913  INFO 77669 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2020-12-28 20:33:04.914  INFO 77669 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 784 ms
2020-12-28 20:33:05.041  INFO 77669 --- [           main] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 'applicationTaskExecutor'
2020-12-28 20:33:05.153  INFO 77669 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
2020-12-28 20:33:05.163  INFO 77669 --- [           main] c.a.g.learn.LearnSpringApplication       : Started LearnSpringApplication in 1.323 seconds (JVM running for 1.863)
```

You should see that the application is running on the port `8080`.

You can visit `localhost:8080` and see the following error.

```markup
Whitelabel Error Page
This application has no explicit mapping for /error, so you are seeing this as a fallback.

Mon Dec 28 20:36:05 IST 2020
There was an unexpected error (type=Not Found, status=404).
```

This is because there is no error handler present for the root. We can let this go for now. We want to interact with the service at endpoints and not root.

## Launching the application

You can execute the command `mvn spring-boot:run` in the project root to launch the application and check the response for consumption of the endpoint as we go along. You can consume the endpoint by executing a CURL request in the terminal or use a tool like Postman. I go with Postman as it gives the freedom to modify a request with the help of a beautiful User Interface. Under the hood, Postman also works with a CURL request. If you're comfortable with writing CURL request, you can use that directly.

## Development approach

1. Writing a test.
2. Writing production code to pass the test.
3. Consuming the endpoint.
4. Writing production code to ensure the response is as expected.

We won't be testing whether an exact error string is returned. Instead, we'll be checking just the status code for failure tests. If we started checking those, it would be hassle and the test & production code would get coupled extensively. It's better to have decoupling.

## Configuration for running tests

We can create a class `ControllerTest`.

Before we write any test,  we must inject a couple of dependencies.

```java
@EnableAutoConfiguration
@ComponentScan(
        excludeFilters = @ComponentScan.Filter(
                type = FilterType.ASSIGNABLE_TYPE,
                value = CommandLineRunner.class))
@SpringBootTest(
        webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT,
        properties = "spring.main.allow-bean-definition-overriding=true",
        classes = {AppConfig.class})
class ControllerTest {

}
```

I have taken the liberty of adding a few annotations required for running tests. I will explain their usage briefly here. You can read up on it if you prefer diving deep on each.

`@EnableAutoConfiguration`: This injects the bean `ServletWebServerFactory` which is required for running web services tests.

`@SpringBootTest` : This has been added to run the web server in the tests on random ports. This is also used to include spring configurations to be used by the tests.

`@ComponentScan`: This scans the source code and makes the `beans`, `configurations`, `components` available in this spring class. `CommandLineRunner` is usually required to run commands directly and is specific to the class where it is defined. It does not make sense to include it in the tests. Hence, the `CommandLineRunner` component has been filtered to be excluded.

Now we have two test class with quite a few annotations assigned. It makes sense to create a base class with the annotations and make the test classes extend them.

```java
@EnableAutoConfiguration
@ComponentScan(
        excludeFilters = @ComponentScan.Filter(
                type = FilterType.ASSIGNABLE_TYPE,
                value = CommandLineRunner.class))
@SpringBootTest(
        webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT,
        properties = "spring.main.allow-bean-definition-overriding=true",
        classes = {AppConfig.class})
@TestInstance(Lifecycle.PER_CLASS)
public class TestBase {

}
```

`@TestInstance` : This ensures that we have a different environment for every test class which subclasses the `TestBase`. This is added to ensure that data does not leak from one test class to another.

```java
class ControllerTest extends TestBase {
}
```

```java
class UserServiceTest extends TestBase {
...
}
```

A new test class can be defined without any boilerplate code now and looks much cleaner.

Let's configure web application in the `ControllerTest` now.

```java
class ControllerTest extends TestBase {

    @Autowired
    private WebApplicationContext fAppContext;

    private MockMvc fMockMvc;

    @BeforeAll
    void setUp() {
        fMockMvc = MockMvcBuilders.webAppContextSetup(fAppContext).build();
    }
}
```

The test class is now ready to accept tests. You can move the `fAppContext` and `fMockMvc` to `TestBase` as well. We'll do that when we write the test class for the Notes Service. It's been a while since we have written any actual domain code. We have been messing around with configuration for quite some time now.

We still need to decide on how our APIs will look.   
We can start off by defining the create user API.

{% api-method method="post" host="https://api.content.com" path="/v1/{requesterId}/user" %}
{% api-method-summary %}
Create a new user
{% endapi-method-summary %}

{% api-method-description %}
A user with administrative permissions can act as a requester to create a new user.
{% endapi-method-description %}

{% api-method-spec %}
{% api-method-request %}
{% api-method-path-parameters %}
{% api-method-parameter name="requesterId" type="string" required=false %}
Id of the requester
{% endapi-method-parameter %}
{% endapi-method-path-parameters %}

{% api-method-body-parameters %}
{% api-method-parameter name="administrator" type="boolean" required=false %}
true if the user will be an administrator
{% endapi-method-parameter %}

{% api-method-parameter name="email" type="string" required=false %}
email of the user
{% endapi-method-parameter %}

{% api-method-parameter name="phone" type="string" required=false %}
phone number of the user
{% endapi-method-parameter %}

{% api-method-parameter name="name" type="string" required=false %}
name of thThe
{% endapi-method-parameter %}
{% endapi-method-body-parameters %}
{% endapi-method-request %}

{% api-method-response %}
{% api-method-response-example httpCode=200 %}
{% api-method-response-example-description %}

{% endapi-method-response-example-description %}

```java
{
    "name": "James Moriarty",
    "id": "ca4b03ef-5354-4a49-8ee2-4fb65acd7bfb",
    "phone": "8474850235",
    "email": "adsa@adsad.com",
    "administrator": false
}
```
{% endapi-method-response-example %}

{% api-method-response-example httpCode=400 %}
{% api-method-response-example-description %}

{% endapi-method-response-example-description %}

```java
{
    "status": 400,
    "error": "Invalid UUID string: null",
    "timestamp": "2021-01-02T06:47:28Z",
    "path": "/v1/null/user",
    "message": "Bad Request"
}
```
{% endapi-method-response-example %}

{% api-method-response-example httpCode=401 %}
{% api-method-response-example-description %}

{% endapi-method-response-example-description %}

```java
{
    "status": 401,
    "error": "Requester is not present",
    "timestamp": "2021-01-02T08:27:08Z",
    "path": "/v1/6bfd12c0-49a6-11eb-b378-0242ac130002/user",
    "message": "Unauthorized"
}
```
{% endapi-method-response-example %}

{% api-method-response-example httpCode=403 %}
{% api-method-response-example-description %}

{% endapi-method-response-example-description %}

```java
{
    "status": 403,
    "error": "Requester is not an administrator and cannot request user creation.",
    "timestamp": "2021-01-02T09:35:13Z",
    "path": "/v1/1109a8c8-49a3-4921-aa80-65e730d587fe/user",
    "message": "Forbidden"
}
```
{% endapi-method-response-example %}
{% endapi-method-response %}
{% endapi-method-spec %}
{% endapi-method %}

## Write a new test

```java
private static final String CREATE_USER = "/v1/%s/user";

@Test
void testCreateUserWhenRequesterIsNull() throws Exception {
    fMockMvc.perform(MockMvcRequestBuilders
            .post(String.format(CREATE_USER, null))
            .contentType(APPLICATION_JSON))
            .andExpect(status().isBadRequest());
}
```

###  Try to run the test

```java
[ERROR] Tests run: 1, Failures: 1, Errors: 0, Skipped: 0, Time elapsed: 0.059 s <<< FAILURE! - in com.atul.gitbook.learn.users.service.ControllerTest
[ERROR] testCreateUserWhenRequesterIsNull  Time elapsed: 0.046 s  <<< FAILURE!
java.lang.AssertionError: Status expected:<400> but was:<404>
        at com.atul.gitbook.learn.users.service.ControllerTest.testCreateUserWhenRequesterIsNull(ControllerTest.java:36)
```

The test returned a `NotFound(404)`, because the endpoint is not present yet.

### Let's try to resolve the failure

We can add the controller and the create user method.

```java
@RestController
public class UserController {

    @PostMapping("/v1/{requesterId}/user")
    public User createUser(@PathVariable("requesterId") UUID requesterId) {
        return null;
    }
}
```

### Try to run the test

The test should be passing now.

### Consume the endpoint

```java
curl -X POST \
  http://localhost:8080/v1/null/user \
  -H 'cache-control: no-cache' \
  -H 'postman-token: ca7f7211-9278-a395-7966-33d27974a2fd'
```

```java
{
    "timestamp": "2021-01-02T06:40:43.351+00:00",
    "status": 400,
    "error": "Bad Request",
    "message": "",
    "path": "/v1/null/user"
}
```

 This request is giving correct status code but the error message does not convey why it is failing.

### Let's try to resolve the failure

We'd need an exception handler which will return an error object to specify the cause of failures.

Let's introduce the `UserError` model first.  This is the model object we'll be using to return the error response. This has a couple of new annotations being introduced. Their responsibilities is clearly evident, `@JsonProperty` sets the field's key in the json and `@JsonFormat` is used to specify the format of the date type fields.

You can experiment and see if a field is not annotated, it won't be returned in the json response.

```java
import com.fasterxml.jackson.annotation.JsonFormat;
import com.fasterxml.jackson.annotation.JsonProperty;

public class UserError {

    public static final String ISO8601PATTERN = "yyyy-MM-dd'T'HH:mm:ssXXX";

    @JsonProperty("status")
    private int fStatus;

    @JsonProperty("error")
    private String fError;

    @JsonProperty("timestamp")
    @JsonFormat(pattern = ISO8601PATTERN, timezone = "UTC")
    private Instant fTimestamp;

    @JsonProperty("path")
    private String fPath;

    @JsonProperty("message")
    private String fMessage;

    public UserError(int status, String error, Instant timestamp, String path, String message) {
        this.fStatus = status;
        this.fError = error;
        this.fTimestamp = timestamp;
        this.fPath = path;
        this.fMessage = message;
    }
}
```

We'll need to add this dependency to flesh out the root cause of the error. The class ExceptionUtils is present in this which does the job for us.

```markup
<dependency>
   <groupId>org.apache.commons</groupId>
   <artifactId>commons-lang3</artifactId>
</dependency>
```

```java
@ControllerAdvice
public class ServiceExceptionHandler extends ResponseEntityExceptionHandler {

    @ExceptionHandler({
            IllegalArgumentException.class,
            HttpMessageConversionException.class,
            MethodArgumentTypeMismatchException.class})
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    ResponseEntity<UserError> handleBadRequest(
            final RuntimeException ex,
            final WebRequest request) {
        final var rootCause = getExceptionRootCause(ex);
        final var message = rootCause != null ? rootCause.getMessage() : "";
        return createErrorResponse(message, request, HttpStatus.BAD_REQUEST);
    }
    
    private static Throwable getExceptionRootCause(@Nullable final Throwable cause) {
        if (cause == null)
            return null;
        final var rootCause = ExceptionUtils.getRootCause(cause);
        if (rootCause == null)
            return cause;
        return rootCause;
    }

    private ResponseEntity<UserError> createErrorResponse(String message, WebRequest request, HttpStatus statusCode) {
        return ResponseEntity.status(statusCode).body(createError(message, request, statusCode));
    }

    private UserError createError(String message, WebRequest request, HttpStatus status) {
        return new UserError(status.value(), message,
                Instant.now(), getUri(request), status.getReasonPhrase());
    }

    private static String getUri(final WebRequest request) {
        if (request == null)
            return "empty request";
        if (request instanceof ServletWebRequest)
            return ((ServletWebRequest) request).getRequest().getRequestURI();
        return "unknown path";
    }
}
```

I took the liberty of adding the `ExceptionHandler` in the refactored state. Let's get to its intricacies now. Whenever an exception is thrown, spring boot looks for a class annotated as `@ControllerAdvice` or `@RestControllerAdvice`. These classes have methods annotated `@ExceptionHandler` which are responsible for formatting the error responses.

Spring boot has a class `ResponseEntityExceptionHandler` which handles several errors for us.

In our case, this class catches all the exceptions and decides what to do with them based on its root cause. A `UserError` object gets created and returned as the response.

### Try to run the test

They all should pass. We have not broken anything with the big change we just added.

### Consume the endpoint

```java
curl -X POST \
  http://localhost:8080/v1/null/user \
  -H 'cache-control: no-cache' \
  -H 'postman-token: 1c5c828c-af16-f427-9a54-921242f7b24c'
```

```java
{
    "status": 400,
    "error": "Invalid UUID string: null",
    "timestamp": "2021-01-02T06:47:28Z",
    "path": "/v1/null/user",
    "message": "Bad Request"
}
```

The status code and error message is as expected now.

## Write a new test

```java
@Test
void testCreateUserWhenRequesterDoesNotExist() throws Exception {
    fMockMvc.perform(MockMvcRequestBuilders
            .post(String.format(CREATE_USER, UUID.randomUUID()))
            .content(new UserDto("Mike Selby", "8765436548", "selby@mark.com").toString())
            .contentType(APPLICATION_JSON))
            .andExpect(status().isUnauthorized());
}
```

### Try to run the test

```java
[ERROR] Failed to execute goal org.apache.maven.plugins:maven-compiler-plugin:3.8.1:compile (default-compile) on project learn: Compilation failure
[ERROR] /Users/creations/IdeaProjects/rest-using-spring-boot/user-service/v8/src/main/java/com/atul/gitbook/learn/users/service/UserController.java:[21,28] method createUser in interface com.atul.gitbook.learn.users.service.IUserService cannot be applied to given types;
[ERROR]   required: com.atul.gitbook.learn.users.models.UserDto
[ERROR]   found: no arguments
[ERROR]   reason: actual and formal argument lists differ in length
```

The controller method must be modified to accept a request body.

### Let's try to resolve the failure

```java
@PostMapping("/v1/{requesterId}/user")
public User createUser(@PathVariable("requesterId") UUID requesterId,
                       @RequestBody UserDto userDto) {
    return null;
}
```

### Try to run the test

```java
[ERROR] Tests run: 2, Failures: 1, Errors: 0, Skipped: 0, Time elapsed: 0.147 s <<< FAILURE! - in com.atul.gitbook.learn.users.service.ControllerTest
[ERROR] testCreateUserWhenRequesterDoesNotExist  Time elapsed: 0.108 s  <<< FAILURE!
java.lang.AssertionError: Status expected:<401> but was:<400>
        at com.atul.gitbook.learn.users.service.ControllerTest.testCreateUserWhenRequesterDoesNotExist(ControllerTest.java:47)
```

We must communicate with the `IUserService` and block the create request if the requester is not found.

### Let's try to resolve the failure

Let's add a constructor in `UserController` first followed by modifying the `UserService` to take in `requesterId` along with `userDto` in the `createUser` method.

```java
@RestController
public class UserController {

    private final IUserService fUserService;

    public UserController(IUserService userService) {
        this.fUserService = userService;
    }

    @PostMapping("/v1/{requesterId}/user")
    public User createUser(@PathVariable("requesterId") UUID requesterId,
                           @RequestBody UserDto userDto) {
        return fUserService.createUser(requesterId, userDto);
    }
}
```

```java
public interface IUserService {

    /**
     * Creates and returns a new user.
     *
     * @param requesterId id of the user making the request
     * @param userDto contains the necessary information for the User.
     * @return the created user
     */
    User createUser(UUID requesterId, UserDto userDto);
    
...

}
```

```java
public class UserService implements IUserService {

...

    @Override
    public User createUser(UUID requesterId, UserDto userDto) {
        validateNotNull(requesterId);
        validateNotNull(userDto);
        return fUserRepository.createUser(userDto);
    }
    
...

}
```

### Try to run the test

```java
[ERROR] Failed to execute goal org.apache.maven.plugins:maven-compiler-plugin:3.8.1:testCompile (default-testCompile) on project learn: Compilation failure: Compilation failure: 
[ERROR] /Users/creations/IdeaProjects/rest-using-spring-boot/user-service/v8/src/test/java/com/atul/gitbook/learn/users/service/UserServiceTest.java:[23,83] method createUser in interface com.atul.gitbook.learn.users.service.IUserService cannot be applied to given types;
[ERROR]   required: java.util.UUID,com.atul.gitbook.learn.users.models.UserDto
[ERROR]   found: <nulltype>
[ERROR]   reason: actual and formal argument lists differ in length
[ERROR] /Users/creations/IdeaProjects/rest-using-spring-boot/user-service/v8/src/test/java/com/atul/gitbook/learn/users/service/UserServiceTest.java:[29,57] method createUser in interface com.atul.gitbook.learn.users.service.IUserService cannot be applied to given types;
[ERROR]   required: java.util.UUID,com.atul.gitbook.learn.users.models.UserDto
[ERROR]   found: com.atul.gitbook.learn.users.models.UserDto
[ERROR]   reason: actual and formal argument lists differ in length
[ERROR] /Users/creations/IdeaProjects/rest-using-spring-boot/user-service/v8/src/test/java/com/atul/gitbook/learn/users/service/UserServiceTest.java:[35,40] method createUser in interface com.atul.gitbook.learn.users.service.IUserService cannot be applied to given types;
[ERROR]   required: java.util.UUID,com.atul.gitbook.learn.users.models.UserDto
[ERROR]   found: com.atul.gitbook.learn.users.models.UserDto
[ERROR]   reason: actual and formal argument lists differ in length
[ERROR] /Users/creations/IdeaProjects/rest-using-spring-boot/user-service/v8/src/test/java/com/atul/gitbook/learn/users/service/UserServiceTest.java:[54,42] method createUser in interface com.atul.gitbook.learn.users.service.IUserService cannot be applied to given types;
[ERROR]   required: java.util.UUID,com.atul.gitbook.learn.users.models.UserDto
[ERROR]   found: com.atul.gitbook.learn.users.models.UserDto
[ERROR]   reason: actual and formal argument lists differ in length
[ERROR] /Users/creations/IdeaProjects/rest-using-spring-boot/user-service/v8/src/test/java/com/atul/gitbook/learn/users/service/UserServiceTest.java:[84,38] method createUser in interface com.atul.gitbook.learn.users.service.IUserService cannot be applied to given types;
[ERROR]   required: java.util.UUID,com.atul.gitbook.learn.users.models.UserDto
[ERROR]   found: com.atul.gitbook.learn.users.models.UserDto
[ERROR]   reason: actual and formal argument lists differ in length
[ERROR] /Users/creations/IdeaProjects/rest-using-spring-boot/user-service/v8/src/test/java/com/atul/gitbook/learn/users/service/UserServiceTest.java:[110,38] method createUser in interface com.atul.gitbook.learn.users.service.IUserService cannot be applied to given types;
[ERROR]   required: java.util.UUID,com.atul.gitbook.learn.users.models.UserDto
[ERROR]   found: com.atul.gitbook.learn.users.models.UserDto
[ERROR]   reason: actual and formal argument lists differ in length
```

We modified the `createUser` method in `IUserService` to accept requesterId which broke some tests in the `UserServiceTest` which do not expect the `requesterId`. Let's pass a parameter to those tests and run the tests again.

```java
[ERROR] Tests run: 2, Failures: 1, Errors: 0, Skipped: 0, Time elapsed: 0.158 s <<< FAILURE! - in com.atul.gitbook.learn.users.service.ControllerTest
[ERROR] testCreateUserWhenRequesterDoesNotExist  Time elapsed: 0.113 s  <<< FAILURE!
java.lang.AssertionError: Status expected:<401> but was:<400>
        at com.atul.gitbook.learn.users.service.ControllerTest.testCreateUserWhenRequesterDoesNotExist(ControllerTest.java:47)
```

We're back to the error where we were at.

### Let's try to resolve the failure

We must add some code to block the user creation if requester is not found. Let's modify the `UserServiceTest`.

We can add service specific exceptions from `RuntimeException` at this point and subclass it for the `ForbiddenException`.

```java
public class ServiceException extends RuntimeException {

    private final HttpStatus fStatus;

    public ServiceException(final String msg, final HttpStatus status) {
        super(msg);
        fStatus = status;
    }

    public HttpStatus getStatus() {
        return fStatus;
    }
}
```

```java
public class UnauthorizedException extends ServiceException {

    public UnauthorizedException(String msg) {
        super(msg, HttpStatus.UNAUTHORIZED);
    }
}
```

```java
@Override
public User createUser(UUID requesterId, UserDto userDto) {
    validateNotNull(requesterId);
    validateNotNull(userDto);
    try {
        fUserRepository.getUser(requesterId);
    } catch (NoSuchElementException e) {
        throw new UnauthorizedException("Requester is not present");
    }
    return fUserRepository.createUser(userDto);
}
```

### Try to run the test

```java
[ERROR] Tests run: 14, Failures: 1, Errors: 4, Skipped: 0, Time elapsed: 2.598 s <<< FAILURE! - in com.atul.gitbook.learn.users.service.UserServiceTest
[ERROR] testUpdateUser  Time elapsed: 0.01 s  <<< ERROR!
com.atul.gitbook.learn.exceptions.UnauthorizedException: Requester is not present
        at com.atul.gitbook.learn.users.service.UserServiceTest.testUpdateUser(UserServiceTest.java:84)

[ERROR] testCreateUserWhenDtoContainsAllTheFields  Time elapsed: 0.003 s  <<< FAILURE!
org.opentest4j.AssertionFailedError: Unexpected exception thrown: com.atul.gitbook.learn.exceptions.UnauthorizedException: Requester is not present
        at com.atul.gitbook.learn.users.service.UserServiceTest.testCreateUserWhenDtoContainsAllTheFields(UserServiceTest.java:29)
Caused by: com.atul.gitbook.learn.exceptions.UnauthorizedException: Requester is not present
        at com.atul.gitbook.learn.users.service.UserServiceTest.lambda$testCreateUserWhenDtoContainsAllTheFields$1(UserServiceTest.java:29)
        at com.atul.gitbook.learn.users.service.UserServiceTest.testCreateUserWhenDtoContainsAllTheFields(UserServiceTest.java:29)

[ERROR] testGetUser  Time elapsed: 0.002 s  <<< ERROR!
com.atul.gitbook.learn.exceptions.UnauthorizedException: Requester is not present
        at com.atul.gitbook.learn.users.service.UserServiceTest.testGetUser(UserServiceTest.java:54)

[ERROR] testCreateUser  Time elapsed: 0.003 s  <<< ERROR!
com.atul.gitbook.learn.exceptions.UnauthorizedException: Requester is not present
        at com.atul.gitbook.learn.users.service.UserServiceTest.testCreateUser(UserServiceTest.java:35)

[ERROR] testDeleteUser  Time elapsed: 0.002 s  <<< ERROR!
com.atul.gitbook.learn.exceptions.UnauthorizedException: Requester is not present
        at com.atul.gitbook.learn.users.service.UserServiceTest.testDeleteUser(UserServiceTest.java:110)

[ERROR] Tests run: 2, Failures: 1, Errors: 0, Skipped: 0, Time elapsed: 0.143 s <<< FAILURE! - in com.atul.gitbook.learn.users.service.ControllerTest
[ERROR] testCreateUserWhenRequesterDoesNotExist  Time elapsed: 0.099 s  <<< FAILURE!
java.lang.AssertionError: Status expected:<401> but was:<400>
        at com.atul.gitbook.learn.users.service.ControllerTest.testCreateUserWhenRequesterDoesNotExist(ControllerTest.java:47)
```

Woah! A bunch of tests failed again.

Cause of failure for the tests in `UserServiceTest`: As we have modified the `UserService`, these tests require an already present user to be used as a requester. We can't use the `createUser` method to create a requester as that would pretty much be the first user of the system. I believe we should add a new field to the `UserDto` to differentiate between a normal user and administrator and only allow the administrator to create new users. We can get away with the issue here by adding an administrator user directly to the repository and use that userId to create the users.

Ideally, these things are fixed using an Identity Management System but as we are working with Spring Boot to handle the users, we have to look for solutions here. Any software project is a continuously evolving system and changes are done in several iterations.

Cause of failure for the tests in `ControllerTest`: The newly introduced `UnauthorizedException` has not been handled in the `ServiceExceptionHandler`.

### Let's try to resolve the failure

Let's fix the privileges first.

We can modify the `UserDto` and `User` model to contain a new field. I have also added a new constructor in `UserDto` as it's gives us some flexibility in creating users. We can safely ignore the administrator parameter if a normal user is being created and use the other constructor to add administrators. I modified the User object to accommodate the administrator field. As it is being called from a single place, it wasn't required to add a new constructor.

```java
public class UserDto {

    private final String fName;
    private final String fPhone;
    private final String fEmail;
    private final boolean fAdministrator;

    /**
     * @param name  the name of the user.
     * @param phone the phone number of the user.
     * @param email the email of the user.
     * @throws IllegalArgumentException in any of the following fails to be conformant:
     *                                  1. Phone number : Must be 10 characters in length and all numeric.
     *                                  2. Email : Must contain @ in the middle of the string.
     */
    public UserDto(String name, String phone, String email) throws IllegalArgumentException {
        validateNotNull(name);
        validateNotNull(phone);
        validateNotNull(email);
        validatePhoneNumber(phone);
        validateEmail(email);
        fName = name;
        fPhone = phone;
        fEmail = email;
        fAdministrator = false;
    }

    /**
     * @param name          the name of the user.
     * @param phone         the phone number of the user.
     * @param email         the email of the user.
     * @param administrator true if the user is an administrator.
     * @throws IllegalArgumentException in any of the following fails to be conformant:
     *                                  1. Phone number : Must be 10 characters in length and all numeric.
     *                                  2. Email : Must contain @ in the middle of the string.
     */
    public UserDto(String name, String phone, String email, boolean administrator) throws IllegalArgumentException {
        validateNotNull(name);
        validateNotNull(phone);
        validateNotNull(email);
        validatePhoneNumber(phone);
        validateEmail(email);
        fName = name;
        fPhone = phone;
        fEmail = email;
        fAdministrator = administrator;
    }

    public String getName() {
        return fName;
    }

    public String getPhone() {
        return fPhone;
    }

    public String getEmail() {
        return fEmail;
    }

    public boolean isAdministrator() {
        return fAdministrator;
    }
}
```

```java
public class User {

    private final UUID fId;
    private final String fName;
    private final String fPhone;
    private final String fEmail;
    private final boolean fAdministrator;

    /**
     * @param id            the user id of the user.
     * @param name          the name of the user.
     * @param phone         the phone number of the user.
     * @param email         the email of the user.
     * @param administrator true if the user is an administrator.
     * @throws IllegalArgumentException in any of the following fails to be conformant:
     *                                  1. Phone number : Must be 10 characters in length and all numeric.
     *                                  2. Email : Must contain @ in the middle of the string.
     */
    public User(UUID id, String name, String phone, String email, boolean administrator) {
        fId = id;
        fName = name;
        fPhone = phone;
        fEmail = email;
        fAdministrator = administrator;
    }

    public static User with(UserDto userDto, UUID userId) {
        return new User(userId, userDto.getName(), userDto.getPhone(), userDto.getEmail(), userDto.isAdministrator());
    }...
}
```

```java
public class InMemoryRepository implements IUserRepository {
    
    public static User ADMINISTRATOR = new User(UUID.randomUUID(), "King Kong", "9999999999", "king@kong.com", true);
    private List<User> users = new ArrayList<>();

    public InMemoryRepository() {
        users.add(ADMINISTRATOR);
    }
    
... 
}
```

We can pass the id of this ADMINISTRATOR to the tests in UserServiceTest.

```java
class UserServiceTest extends TestBase{

    @Autowired
    IUserService fUserService;

    @Test
    void testCreateUserWhenDtoIsNull() {
        Assertions.assertThrows(IllegalArgumentException.class, () -> fUserService.createUser(null, null));
    }

    @Test
    void testCreateUserWhenDtoContainsAllTheFields() {
        final var userDto = new UserDto("Ramsay", "9876483456", "abc@gmail.com");
        Assertions.assertDoesNotThrow(() -> fUserService.createUser(ADMINISTRATOR.getId(), userDto));
    }

    @Test
    void testCreateUser() {
        final var expected = new UserDto("Ramsay", "9876483456", "abc@gmail.com");
        final var actual = fUserService.createUser(ADMINISTRATOR.getId(), expected);
        Assertions.assertEquals(expected.getName(), actual.getName());
        Assertions.assertEquals(expected.getPhone(), actual.getPhone());
        Assertions.assertEquals(expected.getEmail(), actual.getEmail());
    }

    @Test
    void testGetUser() {
        final var userDto = new UserDto("Ramsay", "9876483456", "abc@gmail.com");
        final var expected = fUserService.createUser(ADMINISTRATOR.getId(), userDto);
        final var actual = fUserService.getUser(expected.getId());
        Assertions.assertEquals(expected.getId(), actual.getId());
        Assertions.assertEquals(expected.getName(), actual.getName());
        Assertions.assertEquals(expected.getPhone(), actual.getPhone());
        Assertions.assertEquals(expected.getEmail(), actual.getEmail());
    }

    @Test
    void testUpdateUser() {
        final var userDto = new UserDto("Ramsay", "9876483456", "abc@gmail.com");
        final var user = fUserService.createUser(ADMINISTRATOR.getId(), userDto);
        final var newName = "Abc";
        final var newPhone = "7583929275";
        final var newEmail = "rewr@afsa.com";
        final var newUserDto = new UserDto(newName, newPhone, newEmail);
        fUserService.updateUser(user.getId(), newUserDto);
        final var actual = fUserService.getUser(user.getId());
        Assertions.assertEquals(user.getId(), actual.getId());
        Assertions.assertEquals(newName, actual.getName());
        Assertions.assertEquals(newPhone, actual.getPhone());
        Assertions.assertEquals(newEmail, actual.getEmail());
    }

    @Test
    void testDeleteUser() {
        final var userDto = new UserDto("Ramsay", "9876483456", "abc@gmail.com");
        final var user = fUserService.createUser(ADMINISTRATOR.getId(), userDto);
        Assertions.assertDoesNotThrow(() -> fUserService.deleteUser(user.getId()));
        Assertions.assertThrows(NoSuchElementException.class, () -> fUserService.getUser(user.getId()));
    }
}
```

### Try to run the test

The tests in `UserServiceTest` are now passing. But the `ControllerTest` gives the following error.

```java
[ERROR] Tests run: 2, Failures: 1, Errors: 0, Skipped: 0, Time elapsed: 0.157 s <<< FAILURE! - in com.atul.gitbook.learn.users.service.ControllerTest
[ERROR] testCreateUserWhenRequesterDoesNotExist  Time elapsed: 0.108 s  <<< FAILURE!
java.lang.AssertionError: Status expected:<401> but was:<400>
        at com.atul.gitbook.learn.users.service.ControllerTest.testCreateUserWhenRequesterDoesNotExist(ControllerTest.java:47)
```

Sometimes the test failure does not give enough clue about the error. It's better to launch the service and see how the endpoint is responding.

### Consume the endpoint

```java
curl -X POST \
  http://localhost:8080/v1/6bfd12c0-49a6-11eb-b378-0242ac130002/user \
  -H 'cache-control: no-cache' \
  -H 'content-type: application/json' \
  -H 'postman-token: 023a0876-1cc9-ed5f-7901-6a7423f2f9e6' \
  -d '{
	"name": "Msk",
	"phone": "8474850235",
	"email": "adsa@adsad.com",
	"administrator": false
}'
```

```java
{
    "status": 400,
    "error": "Cannot construct instance of `com.atul.gitbook.learn.users.models.UserDto` (no Creators, like default constructor, exist): cannot deserialize from Object value (no delegate- or property-based Creator)\n at [Source: (PushbackInputStream); line: 2, column: 2]",
    "timestamp": "2021-01-02T08:22:34Z",
    "path": "/v1/6bfd12c0-49a6-11eb-b378-0242ac130002/user",
    "message": "Bad Request"
}
```

### Let's try to resolve the failure

This is because there is no default constructor present in the `UserDto`. The fields can't be final now as the default constructor does not initialise them.

```java
public class UserDto {

    private String fName;
    private String fPhone;
    private String fEmail;
    private boolean fAdministrator;

    public UserDto() {
    }
    
...
}
```

### Consume the endpoint

```java
curl -X POST \
  http://localhost:8080/v1/6bfd12c0-49a6-11eb-b378-0242ac130002/user \
  -H 'cache-control: no-cache' \
  -H 'content-type: application/json' \
  -H 'postman-token: 6f3a40d7-af4c-f8a1-4910-26f59a15210b' \
  -d '{
	"name": "Msk",
	"phone": "8474850235",
	"email": "adsa@adsad.com",
	"administrator": false
}'
```

```java
{
    "status": 401,
    "error": "Requester is not present",
    "timestamp": "2021-01-02T08:27:08Z",
    "path": "/v1/6bfd12c0-49a6-11eb-b378-0242ac130002/user",
    "message": "Unauthorized"
}
```

The endpoint is responding as it should.

### Try to run the test

```java
[ERROR] Tests run: 2, Failures: 1, Errors: 0, Skipped: 0, Time elapsed: 0.165 s <<< FAILURE! - in com.atul.gitbook.learn.users.service.ControllerTest
[ERROR] testCreateUserWhenRequesterDoesNotExist  Time elapsed: 0.12 s  <<< FAILURE!
java.lang.AssertionError: Status expected:<401> but was:<400>
        at com.atul.gitbook.learn.users.service.ControllerTest.testCreateUserWhenRequesterDoesNotExist(ControllerTest.java:47)
```

This error does not explain the cause of the failure. We need more information before we can add anything.

### Let's debug

We need to figure out the cause of Exception failure. We have the following test.

```java
@Test
void testCreateUserWhenRequesterDoesNotExist() throws Exception {
    fMockMvc.perform(MockMvcRequestBuilders
            .post(String.format(CREATE_USER, UUID.randomUUID()))
            .content(new UserDto("Mike Selby", "8765436548", "selby@mark.com").toString())
            .contentType(APPLICATION_JSON))
            .andExpect(status().isUnauthorized());
}
```

If we put a debugger on the Line 7 of this snippet and Step Over after running the test in debug mode, it goes down a multitude of spring's internal files never ending anywhere. No use to us.

From the ServiceExceptionHandler, we can locate ResponseEntityExceptionHandler, you'll notice a method which handles a bunch of exceptions.

```java
public abstract class ResponseEntityExceptionHandler {
...
      @ExceptionHandler({
            HttpRequestMethodNotSupportedException.class,
            HttpMediaTypeNotSupportedException.class,
            HttpMediaTypeNotAcceptableException.class,
            MissingPathVariableException.class,
            MissingServletRequestParameterException.class,
            ServletRequestBindingException.class,
            ConversionNotSupportedException.class,
            TypeMismatchException.class,
            HttpMessageNotReadableException.class,
            HttpMessageNotWritableException.class,
            MethodArgumentNotValidException.class,
            MissingServletRequestPartException.class,
            BindException.class,
            NoHandlerFoundException.class,
            AsyncRequestTimeoutException.class
         })
      @Nullable
      public final ResponseEntity<Object> handleException(Exception ex, WebRequest request) throws Exception {
            HttpHeaders headers = new HttpHeaders();
      ...
      }
      
...
}
```

If we put a debugger on Line 20 of this snippet and run the test in debug mode, you'll notice that we have actually gotten the following.

```java
org.springframework.http.converter.HttpMessageNotReadableException: JSON parse error: Unrecognized token 'com': was expecting (JSON String, Number, Array, Object or token 'null', 'true' or 'false'); nested exception is com.fasterxml.jackson.core.JsonParseException: Unrecognized token 'com': was expecting (JSON String, Number, Array, Object or token 'null', 'true' or 'false')
 at [Source: (PushbackInputStream); line: 1, column: 5]
```

We've some information now. This is because our `requestBody` of the test is not serialized from the `UserDto` object correctly. We are using `toString()` method to generate the `requestBody` from userDto and we can add a customised `toString()` method which will convert to a `json`.

```java
public class UserDto {

...

    @Override
    public String toString() {
        return "{\n"
                + "\"name\" : \"" + fName + "\",\n"
                + "\"phone\" : \"" + fPhone + "\",\n"
                + "\"email\" : \"" + fEmail + "\",\n"
                + "\"administrator\" : " + fAdministrator + "\n"
                + "}";
    }
}
```

### Try to run the test

The tests should be passing now.

Adding `toString()` method to every model for serialization is a pain. Earlier, we noticed that the annotation `@JsonProperty` added in the fields of `UserError` converted the object to an error response. We can use the same for generating the `requestBody`. `Jackson` internally uses the annotations, and there are ways to use the serialization manually. In the system right now, there are very few models that we don't need to get into working on generic Serializers. Let's not get into those right now. We can always change it later.

## Write a new test

We should not allow creation of user if the requester is not an administrator. Let's add a normal user to the InMemoryRepository and use it in the test to pass the requester present criteria.

```java
public class InMemoryRepository implements IUserRepository {

    public static User ADMINISTRATOR = new User(UUID.randomUUID(), "King Kong", "9999999999", "king@kong.com", true);
    public static User USER = new User(UUID.randomUUID(), "David Marshal", "9999999999", "david@marshall.com", false);
    private List<User> users = new ArrayList<>();

    public InMemoryRepository() {
        users.add(ADMINISTRATOR);
        users.add(USER);
    }
    
...
} 
```

```java
@Test
void testCreateUserWhenRequesterExistsButIsNotAdministrator() throws Exception {
    fMockMvc.perform(MockMvcRequestBuilders
            .post(String.format(CREATE_USER, USER.getId()))
            .content(new UserDto("Mike Selby", "8765436548", "selby@mark.com").toString())
            .contentType(APPLICATION_JSON))
            .andExpect(status().isForbidden());
}
```

### Try to run the test

The test should be passing.

### Consume the endpoint

We need to figure out the requesterIds in the InMemoryRepository before making the request. We are generating an UUID every time the InMemoryRepository is initialised. Let's hard-code these values in the repository and use them in the network requests.

```text
public class InMemoryRepository implements IUserRepository {

    public static User ADMINISTRATOR = new User(UUID.fromString("f994c61d-ebd1-463c-a8d8-ebe5989aa501"), "King Kong", "9999999999", "king@kong.com", true);
    public static User USER = new User(UUID.fromString("1109a8c8-49a3-4921-aa80-65e730d587fe"), "David Marshal", "9999999999", "david@marshall.com", false);
    
... 
}
```

```java
curl -X POST \
  http://localhost:8080/v1/1109a8c8-49a3-4921-aa80-65e730d587fe/user \
  -H 'cache-control: no-cache' \
  -H 'content-type: application/json' \
  -H 'postman-token: 6af6d8fe-73c6-4cb5-90bd-1d9700640481' \
  -d '{
	"name": "Msk",
	"phone": "8474850235",
	"email": "adsa@adsad.com",
	"administrator": false
}'
```

```java
{
    "status": 403,
    "error": "Requester is not an administrator and cannot request user creation.",
    "timestamp": "2021-01-02T09:35:13Z",
    "path": "/v1/1109a8c8-49a3-4921-aa80-65e730d587fe/user",
    "message": "Forbidden"
}
```

The endpoint is returned correct status and error message.

## Write a new test

We'll be using the id of administrator to successfully create users.

```java
@Test
void testCreateUserWhenRequesterIsAdministrator() throws Exception {
    final var userDto = new UserDto("Mike Selby", "8765436548", "selby@mark.com");
    fMockMvc.perform(MockMvcRequestBuilders
            .post(String.format(CREATE_USER, ADMINISTRATOR.getId()))
            .content(userDto.toString())
            .contentType(APPLICATION_JSON))
            .andExpect(status().is2xxSuccessful());
}
```

###  Try to run the test

```java
[ERROR] Tests run: 4, Failures: 1, Errors: 0, Skipped: 0, Time elapsed: 0.194 s <<< FAILURE! - in com.atul.gitbook.learn.users.service.ControllerTest
[ERROR] testCreateUserWhenRequesterIsAdministrator  Time elapsed: 0.163 s  <<< FAILURE!
java.lang.AssertionError: Range for response status value 400 expected:<SUCCESSFUL> but was:<CLIENT_ERROR>
        at com.atul.gitbook.learn.users.service.ControllerTest.testCreateUserWhenRequesterIsAdministrator(ControllerTest.java:69)
```

### Let's try to resolve the failure

This error is not straightforward to debug. Previously, we have seen issues with serialization. Let's add the `@JsonProperty` to fields in `UserDto` and run the test again.

```java
public class UserDto {

    @JsonProperty("name")
    private String fName;

    @JsonProperty("phone")
    private String fPhone;

    @JsonProperty("email")
    private String fEmail;

    @JsonProperty("administrator")
    private boolean fAdministrator;
    
...
}
```

```java
public class User {

    @JsonProperty("id")
    private UUID fId;

    @JsonProperty("name")
    private String fName;

    @JsonProperty("phone")
    private String fPhone;

    @JsonProperty("email")
    private String fEmail;

    @JsonProperty("administrator")
    private boolean fAdministrator;
    
...
}
```

### Try to run the test

The tests should all pass now.

### Consume the endpoint

```java
curl -X POST \
  http://localhost:8080/v1/f994c61d-ebd1-463c-a8d8-ebe5989aa501/user \
  -H 'cache-control: no-cache' \
  -H 'content-type: application/json' \
  -H 'postman-token: dc8d4d68-1446-095e-fe3c-1254ddd7e207' \
  -d '{
	"name": "James Moriarty",
	"phone": "8474850235",
	"email": "adsa@adsad.com",
	"administrator": false
}'
```

```java
{
    "name": "James Moriarty",
    "id": "ca4b03ef-5354-4a49-8ee2-4fb65acd7bfb",
    "phone": "8474850235",
    "email": "adsa@adsad.com",
    "administrator": false
}
```

A user has successfully been created using the controller endpoint now.

## Project Status

We handled a lot of cases with the create user request. We passed only 4 tests, but there was a lot of code involved in making that happen.

We added restriction on who can create the users and handled the errors gracefully by throwing custom exceptions.

We understood the importance of Jackson annotations for serialising and deserialising object.

We broke some tests along the way but they were not so difficult to fix and we handled them.

Let's move on to fetching the user details.

