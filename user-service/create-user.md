# Create User

### Initiation <a href="#initiation" id="initiation"></a>

We should start by putting together the user model and define the CRUD service methods for the users. This chapter will deal with service method for creation of users.

### Model definition <a href="#model-definition" id="model-definition"></a>

Here's how the User model could look for now. Humans can usually be identified based on Name, Email and Phone and hence I added just those in the model definition. We can add other details if requirement arises.

Copy

```
//User information
{
    "id" : "UUID",
    "name" : "Steven Hendry",
    "email" : "thatsadevil@gmail.com",
    "phone" : "+919876654134"
}
```

### User Service <a href="#user-service" id="user-service"></a>

The first thing we need to get anything started is to put an operation to create the users. This could be a method which takes in an object encapsulating name, email and phone and returns the created user with an id. For the purposes of our first exercise, we can use an in-memory list to store the users. We can scale it to use a database later. The request object would be called as DTO (Data Transfer Object).

We will only start the creation operation once we have validated if the data present in the DTO is conforming with the service guidelines. These guidelines can be dynamic and we'll be beginning with the minimum possibility to demonstrate the process of enforcing validation. We will have to write tests for the validations before moving forward.

### Bean configuration <a href="#bean-configuration" id="bean-configuration"></a>

The services in spring are declared as beans and are autowired as dependencies when they are required.

Let's declare the file `AppConfig` where all the beans would be defined and would be imported in the tests.

Copy

```
@Configuration
public class AppConfig {

    @Bean
    IUserService configureUserService() {
        return new UserService();
    }
}
```

The tests would be required to import configuration annotations and can be done as following.

Copy

```
@SpringBootTest(
        properties = "spring.main.allow-bean-definition-overriding=true",
        classes = {AppConfig.class})
class UserServiceTest {

    @Autowired
    IUserService fUserService;

}
```

### Write a new test <a href="#write-a-new-test" id="write-a-new-test"></a>

Let's write a test in `UserServiceTest` when the DTO is passed as null. We would not want the request to be processed in such cases. Let's throw an `IIlegalArgumentException` in such cases, we can declare application specific exception later if we get the chance to scale the system with multiple micro-services.

Copy

```
    @Test
    void testCreateUserWhenDtoIsNull() {
        Assertions.assertThrows(IllegalArgumentException.class, () -> fUserService.createUser(null));
    }
```

#### Try to run the test <a href="#try-to-run-the-test" id="try-to-run-the-test"></a>

Copy

```
/Users/amitmahato/rest-api-using-spring-boot/user-service/v1/src/test/java/com/amit/gitbook/learn/users/UserServiceTest.java:20:83
java: cannot find symbol
symbol:   method createUser(<nulltype>)
location: variable fUserService of type com.amit.gitbook.learn.users.IUserService
```

This specifies that the `createUser` method is not yet declared. Let's define that in the `interface` and implement it in the `UserService` class.

#### Let's try to resolve the failure <a href="#lets-try-to-resolve-the-failure" id="lets-try-to-resolve-the-failure"></a>

We will declare the minimum items required to pass the test before thinking about the implementation details.

Copy

```
User createUser(UserDto userDto);
```

When you'd run the test, you'll see a compilation failure

Copy

```
/Users/amitmahato/rest-api-using-spring-boot/user-service/v1/src/main/java/com/amit/gitbook/learn/users/IUserService.java:4:21
java: cannot find symbol
symbol:   class UserDto
location: interface com.amit.gitbook.learn.users.IUserService
```

It's time to define the models now. Let's create a package `users/models/` and declare the classes and run the test again.

Copy

```
/Users/amitmahato/rest-api-using-spring-boot/user-service/v1/src/main/java/com/amit/gitbook/learn/users/impl/UserService.java:5:8
java: com.amit.gitbook.learn.users.impl.UserService is not abstract and does not override abstract method createUser(com.amit.gitbook.learn.users.models.UserDto) in com.amit.gitbook.learn.users.IUserService
```

These compilation failures may look terrible but they are the pathway towards a working software. We just need to read through the error and fix the things that it is demanding.

We will now implement the `createUser` method in `UserService`. The compilation errors should be fixed now. You'll notice that the test is failing with the following error.

Copy

```
org.opentest4j.AssertionFailedError: Expected java.lang.IllegalArgumentException to be thrown, but nothing was thrown.
at com.amit.gitbook.learn.users.UserServiceTest.testCreateUserWhenDtoIsNull(UserServiceTest.java:20)
```

We must add validations to ensure null entries are not processed. The files should look something like this now.

Copy

```
public interface IUserService {
    User createUser(UserDto userDto) throws IllegalArgumentException;
}
```

Copy

```
@Override
public User createUser(UserDto userDto) throws IllegalArgumentException {
    if (userDto == null) {
        throw new IllegalArgumentException();
    }
    return null;
}
```

When you'd run the test, it should be passing now. This concludes one cycle of TDD.

It's a good idea to add documentation to the interface method now. You can skip the documentation for the concrete class for now as it is performing no different action than the interface. This might look like a small project at the moment but imbibing good practices takes time and must be made into the habit of writing great software. Following is a sample of how you can go about it.

Copy

```
public interface IUserService {

    /**
     * Creates and returns a new user.
     *
     * @param userDto contains the necessary information for the User.
     * @return the created user
     * @throws IllegalArgumentException if fails validation.
     */
    User createUser(UserDto userDto) throws IllegalArgumentException;
}
```

### Write a new test <a href="#write-a-new-test-1" id="write-a-new-test-1"></a>

Let's extend the create method to accept all the inputs we decided to work with.

Copy

```
@Test
void testCreateUserWithAllTheFields() {
    final var userDto = new UserDto("Ramsay", "9876543456", "abc@def.com");
    Assertions.assertDoesNotThrow(() -> fUserService.createUser(userDto));
}
```

#### Try to run the test <a href="#try-to-run-the-test-1" id="try-to-run-the-test-1"></a>

Copy

```
[ERROR] Failed to execute goal org.apache.maven.plugins:maven-compiler-plugin:3.8.1:testCompile (default-testCompile) on project learn: Compilation failure
[ERROR] /Users/creations/IdeaProjects/rest-using-spring-boot/create-user/v2/learn/src/test/java/com/atul/gitbook/learn/users/UserServiceTest.java:[25,27] constructor UserDto in class com.atul.gitbook.learn.users.models.UserDto cannot be applied to given types;
[ERROR]   required: no arguments
[ERROR]   found: java.lang.String,java.lang.String,java.lang.String
[ERROR]   reason: actual and formal argument lists differ in length
```

#### Let's try to resolve the failure <a href="#lets-try-to-resolve-the-failure-1" id="lets-try-to-resolve-the-failure-1"></a>

We'd run the test after defining the constructor must be defined to take in all the fields. The test should be passing now. I've just defined the constructor at the moment and not declared any fields. We can do that when we get to successfully creating the user.

Copy

```
public class UserDto {

    public UserDto(String name, String phone, String email) {

    }
}
```

The tests should pass now.

### Input validations <a href="#input-validations" id="input-validations"></a>

Here is the list of input validations that I can think of at the moment.

* Phone number : Must be 10 characters in length and all numeric.
* Email : Must contain @ in the middle of the string.
* Any of the fields are null

You can add more validations later. I'm just giving an idea of how to organise validations.

### Where do you think these validations should go? <a href="#where-do-you-think-these-validations-should-go" id="where-do-you-think-these-validations-should-go"></a>

We have two distinct entities here `UserDto` and `UserService`. Here are some questions which can help in answering the question.

* Should `UserService` be responsible for validating contents of UserDto?
* Should `UserService` only get valid UserDto to perform an operation?
* Are all the validations of the same level?

Thoughts:

* `UserService` should handle validations which are specific to the business domain.
* `UserDto` should handle validations which are generic enough to be independent of business domain.
* For phone number, `UserService` can have validations which restrict the phone number to certain countries and that would require another field like CountryCode but the length of phone number as 10 can stay in the `UserDto` as we can assume that is the global requirement for the phone number.
* For email, the format decided by us is a global standard and we can have that in the UserDto.

The validations which would be placed in the `UserDto` are independent of `UserService` and should stay in the models package, away from the `UserServiceTest`. We would also not need the `Autowired` `IUserService` for those tests.

### Write a new test <a href="#write-a-new-test-2" id="write-a-new-test-2"></a>

Let's create a new class `UserDtoTest` and write tests to handle more validations.

Copy

```
    @Test
    void testCreateUserWhenPhoneNumberContainsAlphabets() {
        Assertions.assertThrows(IllegalArgumentException.class, () -> new UserDto("Julie", "78978972ab", "abc@de.com"));
    }
```

#### Try to run the test <a href="#try-to-run-the-test-2" id="try-to-run-the-test-2"></a>

Copy

```
[ERROR] Failures: 
[ERROR]   UserDtoTest.testCreateUserWhenPhoneNumberContainsAlphabets:12 Expected java.lang.IllegalArgumentException to be thrown, but nothing was thrown.
```

#### Let's try to resolve the failure <a href="#lets-try-to-resolve-the-failure-2" id="lets-try-to-resolve-the-failure-2"></a>

We added an if check in the `UserDto` constructor to make the test pass.

Copy

```
public UserDto(String name, String phone, String email) throws IllegalArgumentException {
        if (!(phone.matches("[0-9]+"))) {
            throw new IllegalArgumentException();
        }
    }
}
```

The tests should pass now.

### Write a new test <a href="#write-a-new-test-3" id="write-a-new-test-3"></a>

Copy

```
@Test
void testCreateUserWhenPhoneNumberOfLengthNot10() {
    Assertions.assertThrows(IllegalArgumentException.class, () -> new UserDto("Julie", "789789728", "abc@de.com"));
}
```

#### Try to run the test <a href="#try-to-run-the-test-3" id="try-to-run-the-test-3"></a>

Copy

```
[ERROR] Failures: 
[ERROR]   UserDtoTest.testCreateUserWhenPhoneNumberOfLengthNot10:17 Expected java.lang.IllegalArgumentException to be thrown, but nothing was thrown.
```

#### Let's try to resolve the failure <a href="#lets-try-to-resolve-the-failure-3" id="lets-try-to-resolve-the-failure-3"></a>

We have modified the regex to only take in 10 characters to pass the test

Copy

```
public UserDto(String name, String phone, String email) throws IllegalArgumentException {
    if (!phone.matches("[0-9]{10}")) {
        throw new IllegalArgumentException();
    }
}
```

The tests should pass now.

### Write a new test <a href="#write-a-new-test-4" id="write-a-new-test-4"></a>

Copy

```
@Test
void testCreateUserWhenEmailIsNotOfCorrectFormat() {
    Assertions.assertThrows(IllegalArgumentException.class, () -> new UserDto("Julie", "7897897280", "abc@"));
}
```

#### Try to run the test <a href="#try-to-run-the-test-4" id="try-to-run-the-test-4"></a>

Copy

```
[ERROR] Failures: 
[ERROR]   UserDtoTest.testCreateUserWhenEmailIsNotOfCorrectFormat:22 Expected java.lang.IllegalArgumentException to be thrown, but nothing was thrown.
```

#### Let's try to resolve the failure <a href="#lets-try-to-resolve-the-failure-4" id="lets-try-to-resolve-the-failure-4"></a>

We can add another check for the email in the `UserDto` constructor.

Copy

```
public UserDto(String name, String phone, String email) throws IllegalArgumentException {
    if (!phone.matches("[0-9]{10}")) {
        throw new IllegalArgumentException();
    }
    if (!email.matches("^([\\w-\\.]+){1,64}@([\\w&&[^_]]+){2,255}.[a-z]{2,}$")) {
        throw new IllegalArgumentException();
    }
}
```

The tests should pass now.

Let's add one more test to finish off the object creation. We'll be passing valid input and expect that exception is not being thrown.

### Write a new test <a href="#write-a-new-test-5" id="write-a-new-test-5"></a>

Copy

```
@Test
void testCreateUserWhenValidInput() {
    Assertions.assertDoesNotThrow(() -> new UserDto("Julie", "7897897280", "abc@de.com"));
}
```

The tests should be passing without making any code changes.

### Refactor <a href="#refactor" id="refactor"></a>

Let's take a step back and look at how our code looks. Do we have repetitive code in system?

If we look at the `UserDtoTest`, we can notice that the methods to validate invalid input performs the same operation but on different data inputs. These tests can be combined to what `JUnit` likes to call a `ParameterizedTest`. `ParameterizedTest` uses a stream of arguments and runs the test for all the set of arguments.

Let's refactor our tests to use them.

Copy

```
@ParameterizedTest
@MethodSource("streamForCreateUserInvalidInput")
void testCreateUserInvalidInput(String name, String phone, String email) {
    Assertions.assertThrows(IllegalArgumentException.class, () -> new UserDto(name, phone, email));
}

private static Stream<Arguments> streamForCreateUserInvalidInput() {
    return Stream.of(
            Arguments.of("Julie", "78978297a4", "abc@de.com"),
            Arguments.of("Julie", "789789728", "abc@de.com"),
            Arguments.of("Julie", "7897839780", "abc@")
    );
}
```

You can run the tests and see that all of them are passing. You can remove the individual tests as this will take care of them. You can easily add more set of arguments to test other invalid inputs. Let's pass null values and throw errors in those cases as well.

Copy

```
public UserDto(String name, String phone, String email) throws IllegalArgumentException {
    if (name == null) {
        throw new IllegalArgumentException();
    }
    if (phone == null) {
        throw new IllegalArgumentException();
    }
    if (email == null) {
        throw new IllegalArgumentException();
    }
    if (!phone.matches("[0-9]{10}")) {
        throw new IllegalArgumentException();
    }
    if (!email.matches("^([\\w-\\.]+){1,64}@([\\w&&[^_]]+){2,255}.[a-z]{2,}$")) {
        throw new IllegalArgumentException();
    }
}
```

There are several blocks throwing the same exception. We have not error message now because it could get difficult to make them generic later. We can refactor the code to handle specific error messages later. Now, let's refactor them into validation methods to make the constructor readable. You can mention these validations in the documentation for the `UserDto` for explaining its capabilities.

Copy

```
public UserDto(String name, String phone, String email) throws IllegalArgumentException {
    validateNotNull(name);
    validateNotNull(phone);
    validateNotNull(email);
    validatePhoneNumber(phone);
    validateEmail(email);
}

private void validateNotNull(Object field) {
    if (field == null) {
        throw new IllegalArgumentException();
    }
}

private void validatePhoneNumber(String phone) {
    if (!phone.matches("[0-9]{10}")) {
        throw new IllegalArgumentException();
    }
}

private void validateEmail(String email) {
    if (!email.matches("^([\\w-\\.]+){1,64}@([\\w&&[^_]]+){2,255}.[a-z]{2,}$")) {
        throw new IllegalArgumentException();
    }
}
```

`validateNotNull` These validate methods seems like they can be used at several places. Let's refactor it out to a class called `Preconditions`.

Copy

```
public class Preconditions {

    public static void validateNotNull(Object field) {
        if (field == null) {
            throw new IllegalArgumentException();
        }
    }

    public static void validatePhoneNumber(String phone) {
        if (!phone.matches("[0-9]{10}")) {
            throw new IllegalArgumentException();
        }
    }

    public static void validateEmail(String email) {
        if (!email.matches("^([\\w-\\.]+){1,64}@([\\w&&[^_]]+){2,255}.[a-z]{2,}$")) {
            throw new IllegalArgumentException();
        }
    }
}
```

UserDto should look cleaner now. Let's declare the fields and store the value in them. Add the getters for the fields. Repeat the same steps for the User model as well.

Copy

```
public class User {

    private final UUID fId;
    private final String fName;
    private final String fPhone;
    private final String fEmail;

    /**
     * @param id    the user id of the user.
     * @param name  the name of the user.
     * @param phone the phone number of the user.
     * @param email the email of the user.
     * @throws IllegalArgumentException in any of the following fails to be conformant:
     *                                  1. Phone number : Must be 10 characters in length and all numeric.
     *                                  2. Email : Must contain @ in the middle of the string.
     */
    public User(UUID id, String name, String phone, String email) throws IllegalArgumentException {
        validateNotNull(id);
        validateNotNull(name);
        validateNotNull(phone);
        validatePhoneNumber(phone);
        validateNotNull(email);
        validateEmail(email);
        fId = id;
        fName = name;
        fPhone = phone;
        fEmail = email;
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
}
```

Copy

```
public class UserDto {

    private final String fName;
    private final String fPhone;
    private final String fEmail;

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
}
```

Copy

```
class UserDtoTest {

    @ParameterizedTest
    @MethodSource("streamForCreateUserInvalidInput")
    void testCreateUserInvalidInput(String name, String phone, String email) {
        Assertions.assertThrows(IllegalArgumentException.class, () -> new UserDto(name, phone, email));
    }

    private static Stream<Arguments> streamForCreateUserInvalidInput() {
        return Stream.of(
                Arguments.of("Julie", "78978297a4", "abc@de.com"),
                Arguments.of("Julie", "789789728", "abc@de.com"),
                Arguments.of("Julie", "7897839780", "abc@"),
                Arguments.of(null, "7897897280", "abc@de.com"),
                Arguments.of("Julie", null, "abc@de.com"),
                Arguments.of("Julie", "7897897280", null)
        );
    }

    @Test
    void testCreateUserValidInput() {
        Assertions.assertDoesNotThrow(() -> new UserDto("Julie", "7897897280", "abc@de.com"));
    }
}
```

This concludes another cycle of TDD and you can see the beauty of the code which we have developed.

### Project Status <a href="#project-status" id="project-status"></a>

We have the `UserService` ready to only accept valid values in the input. The `UserDto` validates invalid input and throws `Exception` whenever invalid input is passed. We can now move on to creating a `User`. We won't be storing the object just yet as we have not written the tests for retrieving a `User`.

### Write a new test <a href="#write-a-new-test-6" id="write-a-new-test-6"></a>

Copy

```
@Test
void testCreateUser() {
    final var expected = new UserDto("Ramsay", "9876483456", "abc@gmail.com");
    final var actual = fUserService.createUser(expected);
    Assertions.assertEquals(expected.getName(), actual.getName());
    Assertions.assertEquals(expected.getPhone(), actual.getPhone());
    Assertions.assertEquals(expected.getEmail(), actual.getEmail());
}
```

#### Try to run the test <a href="#try-to-run-the-test-5" id="try-to-run-the-test-5"></a>

Copy

```
[ERROR] Failures: 
[ERROR]   UserServiceTest.testCreateUser:34 expected: <com.atul.gitbook.learn.users.models.UserDto@10afe71a> but was: <null>
```

#### Let's try to resolve the failure <a href="#lets-try-to-resolve-the-failure-5" id="lets-try-to-resolve-the-failure-5"></a>

We can use a random `UUID` as we just need to pass the test at the moment.

Copy

```
@Override
public User createUser(UserDto userDto) throws IllegalArgumentException {
    if (userDto == null) {
        throw new IllegalArgumentException();
    }
    return new User(UUID.randomUUID(), userDto.getName(), userDto.getPhone(), userDto.getEmail());
}
```

The tests should all be passing now.

### Refactor <a href="#refactor-1" id="refactor-1"></a>

The createUser method can use the validation method we just declared in `Preconditions`.

Copy

```
@Override
public User createUser(UserDto userDto) throws IllegalArgumentException {
    validateNotNull(userDto);
    return new User(UUID.randomUUID(), userDto.getName(), userDto.getPhone(), userDto.getEmail());
}
```

The return value has the creation of a `User` object and could be better suited in the User class. We can create a static method which takes in a `id` and `userDto` to flesh out the `User` object.

Copy

```
public static User with(UserDto userDto, UUID userId) {
    return new User(userId, userDto.getName(), userDto.getPhone(), userDto.getEmail());
}
```

Copy

```
@Override
public User createUser(UserDto userDto) throws IllegalArgumentException {
    validateNotNull(userDto);
    return User.with(userDto, UUID.randomUUID());
}
```

The service method now looks much cleaner than earlier and is easily comprehensible. Another TDD cycle concluded.

### Project Status <a href="#project-status-1" id="project-status-1"></a>

We are at the end of user creation now. We have just enough code to test that the service method is returning a `User` object with the passed in data. The `UserService` is the only place in the system which can create instances of the `User` model.
