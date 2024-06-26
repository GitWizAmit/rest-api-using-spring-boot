# Refactor

## Try to run the test

Before moving further ahead, let's take a look at our tests. There are a few tests which are returning `2xxSuccessful` but the response is not tested. Let's switch to `InMemoryRepository` by updating the Bean configuration and add the response checks to our tests. We already have the `Serializers` utils ready to parse the response.

```java
    @Bean
    IUserRepository configureUserRepository(DataSource dataSource,
                                            @Value("${function.user.create}") String createUserSproc,
                                            @Value("${function.user.get}") String getUserSproc) {
//        return new PostgresRepository(dataSource, createUserSproc, getUserSproc);
        return new InMemoryRepository();
    }
```

If you'd run the test now, all of them would be passing.

### Modify test

```java
@Test
void testGetUserWhenRequesterIsFetchingOwnDetails() throws Exception {
    fMockMvc.perform(getUserRequest(USER.getId(), USER.getId()))
                .andExpect(status().is2xxSuccessful());
}
```

 to

```java
@Test
void testGetUserWhenRequesterIsFetchingOwnDetails() throws Exception {
    final var expected = USER;
    final var contentAsString = fMockMvc.perform(getUserRequest(expected.getId(), expected.getId()))
            .andExpect(status().is2xxSuccessful())
            .andReturn().getResponse().getContentAsString();
    final var actual = USER_SERIALIZER.deserialize(contentAsString);
    Assertions.assertEquals(expected.getId(), actual.getId());
    Assertions.assertEquals(expected.getName(), actual.getName());
    Assertions.assertEquals(expected.getPhone(), actual.getPhone());
    Assertions.assertEquals(expected.getEmail(), actual.getEmail());
}
```

### Modify test

```java
@Test
void testGetUserWhenRequesterIsAdministratorAndUserIsPresent() throws Exception {
    fMockMvc.perform(getUserRequest(ADMINISTRATOR.getId(), USER.getId()))
            .andExpect(status().is2xxSuccessful());
}
```

to

```java
@Test
void testGetUserWhenRequesterIsAdministratorAndUserIsPresent() throws Exception {
    final var expected = USER;
    final var contentAsString = fMockMvc.perform(getUserRequest(ADMINISTRATOR.getId(), expected.getId()))
            .andExpect(status().is2xxSuccessful())
            .andReturn().getResponse().getContentAsString();
    final var actual = USER_SERIALIZER.deserialize(contentAsString);
    Assertions.assertEquals(expected.getId(), actual.getId());
    Assertions.assertEquals(expected.getName(), actual.getName());
    Assertions.assertEquals(expected.getPhone(), actual.getPhone());
    Assertions.assertEquals(expected.getEmail(), actual.getEmail());
}
```

### Modify test

```java
@Test
void testUpdateUserWhenRequesterIsUpdatingOwnDetails() throws Exception {
    fMockMvc.perform(updateUserRequest(USER.getId(), USER.getId(), new UpdateUserDto("Mike Selby", "8765436548", "selby@mark.com")))
            .andExpect(status().is2xxSuccessful());
}
```

to

```java
@Test
void testUpdateUserWhenRequesterIsUpdatingOwnDetails() throws Exception {
    final var expected = new UpdateUserDto("Mike Selby", "8765436548", "selby@mark.com");
    final var contentAsString = fMockMvc.perform(updateUserRequest(USER.getId(), USER.getId(), expected))
            .andExpect(status().is2xxSuccessful())
            .andReturn().getResponse().getContentAsString();
    final var actual = USER_SERIALIZER.deserialize(contentAsString);
    Assertions.assertEquals(USER.getId(), actual.getId());
    Assertions.assertEquals(expected.getName(), actual.getName());
    Assertions.assertEquals(expected.getPhone(), actual.getPhone());
    Assertions.assertEquals(expected.getEmail(), actual.getEmail());
}
```

You can refactor out the assertion methods but I'll leave that upto you.

### Try to run the test

Let's switch to the `PostgresRepository` now and see how many of the tests are failing.

```java
[ERROR] Tests run: 20, Failures: 2, Errors: 0, Skipped: 0, Time elapsed: 7.973 s <<< FAILURE! - in com.atul.gitbook.learn.users.service.ControllerTest
[ERROR] testDeleteUserWhenRequestIsDeletingOwnProfile  Time elapsed: 0.068 s  <<< FAILURE!
java.lang.AssertionError: Status expected:<401> but was:<200>
        at com.atul.gitbook.learn.users.service.ControllerTest.testDeleteUserWhenRequestIsDeletingOwnProfile(ControllerTest.java:184)

[ERROR] testUpdateUserWhenRequesterIsUpdatingOwnDetails  Time elapsed: 0.011 s  <<< FAILURE!
org.opentest4j.AssertionFailedError: expected: <Mike Selby> but was: <David Marshal>
        at com.atul.gitbook.learn.users.service.ControllerTest.testUpdateUserWhenRequesterIsUpdatingOwnDetails(ControllerTest.java:145)
```

The responses of the create and get users are now working as they should. We're well on our way to migrating to a persistent database.

## Default user objects

Now that we've our `getUser` method working. Let's add the restriction to only create the `ADMINISTRATOR` and `USER` if they don't exist. We can also return the objects and have the reference stored in the repository.

```java
private User createUserIfNotPresent(UUID userId, UserDto userDto) {
    try {
        return getUser(userId);
    } catch (NoSuchElementException e) {
        return createUser(userId, userDto);
    }
}
```

```java
public final User ADMINISTRATOR;
public final User USER;
```

```java
ADMINISTRATOR = createUserIfNotPresent(UUID.fromString("f994c61d-ebd1-463c-a8d8-ebe5989aa501"),
        new UserDto("King Kong", "9999999999", "king@kong.com", true));
USER = createUserIfNotPresent(UUID.fromString("1109a8c8-49a3-4921-aa80-65e730d587fe"),
        new UserDto("David Marshal", "9999999999", "david@marshall.com", false)); 
```

The tests can now reference these objects as well.

### Refactor

As we can't place object definitions in interfaces, we can have the reference placed in the `IUserRepository` by making that an abstract class and have the other repositories extend it.

```java
public abstract class IUserRepository {

    /**
     * Creates and returns a new user.
     *
     * @param userDto contains the necessary information for the User.
     * @return the created user
     */
    public abstract User createUser(UserDto userDto);

    /**
     * Returns the user with the provided userId.
     *
     * @param id the userId of the user being queried.
     */
    public abstract User getUser(UUID id);

    /**
     * Updates the user.
     *
     * @param id userId of the user being queried.
     * @param userDto contains the new information for the User.
     */
    public abstract void updateUser(UUID id, UpdateUserDto userDto);

    /**
     * Deletes the user.
     *
     * @param id userId of the user to be deleted.
     */
    public abstract void deleteUser(UUID id);
}
```

```java
public class InMemoryRepository extends IUserRepository {
...
}
```

There should not be any problem with `InMemoryRepository` but there is an issue with the `PostgresRepository`. `PostgresRepository` is already extending `JdbcDaoSupport` and we all know Java is a single inheritance language. One solution is to add a new class which will extend `JdbcDaoSupport` and add it as dependency to the `PostgresRepository`. This will also help in separating some responsibilities.

```java
public class UserRepositoryJdbcDaoSupport extends JdbcDaoSupport {

    public UserRepositoryJdbcDaoSupport(DataSource dataSource) {
        setDataSource(dataSource);
    }
}
```

```java
@Bean
IUserRepository configureUserRepository(UserRepositoryJdbcDaoSupport jdbcDaoSupport,
                                        @Value("${function.user.create}") String createUserSproc,
                                        @Value("${function.user.get}") String getUserSproc) {
    return new PostgresRepository(jdbcDaoSupport, createUserSproc, getUserSproc);
}

@Bean
UserRepositoryJdbcDaoSupport configureJdbc(DataSource dataSource) {
    return new UserRepositoryJdbcDaoSupport(dataSource);
}
```

```java
private final UserRepositoryJdbcDaoSupport fJdbcDaoSupport;
...
public PostgresRepository(UserRepositoryJdbcDaoSupport jdbcDaoSupport,
                          String createUserSproc,
                          String getUserSproc) {
    fJdbcDaoSupport = jdbcDaoSupport;
    ...
}
```

This will introduce some errors in the methods where `getConnection()` and `getJdbcTemplate()` are being called. These methods can now be called using the `fJdbcDaoSupport` object.

The `getConnection()` in `JdbcDaoSupport` has protected access. We can define a new method in the `UserRepositoryJdbcDaoSupport` which can return the connection.

```java
public class UserRepositoryJdbcDaoSupport extends JdbcDaoSupport {

    ...

    public Connection getConn() {
        return getConnection();
    }
}
```

```java
private User createUser(UUID userId, UserDto userDto) {
    try {
        final var createUserCallableStatement = generateCreateUserCallableStatement(userId.toString(),
                userDto, fCreateUserSproc, SERIALIZER_USER_DTO, fJdbcDaoSupport.getConn());
        createUserCallableStatement.execute();
        return getUser(userId);
    } catch (SQLException e) {
        e.printStackTrace();
        return null;
    }
}
```

 The `getJdbcTemplate()` method should not have any issue of that sort.

```java
@Override
public User getUser(UUID id) {
    final PreparedStatementCreator getUserCallableStatement = (Connection connection) ->
            generateGetUserCallableStatement(id.toString(), fGetUserSproc, connection);
    final var userList = fJdbcDaoSupport.getJdbcTemplate().query(getUserCallableStatement, getUserRowMapper());
    final var user = userList.stream().findFirst();
    if (user.isPresent())
        return user.get();
    throw new NoSuchElementException();
}
```

Now, we can have the `PostgresRepository` extend our abstract class.

```java
public class PostgresRepository extends IUserRepository {

...
}
```

### Try to run the test

Run the tests to check if we didn't break anything. Fortunately, we didn't.

### Refactor

We can now work on having the user objects in one place. Though, there is another mismatch. The objects in `InMemoryRepository` are being accessed statically and we can't have the objects in `PostgresRepository` static as they get initialized in the constructor. We'd need getters to reference the objects now. Let's make the objects in `InMemoryRepository` non-static and add getters and modify the tests first to use getters.

```java
public class InMemoryRepository extends IUserRepository {

    private User fAdministrator = new User(UUID.fromString("f994c61d-ebd1-463c-a8d8-ebe5989aa501"), "King Kong", "9999999999", "king@kong.com", true);
    private User fUser = new User(UUID.fromString("1109a8c8-49a3-4921-aa80-65e730d587fe"), "David Marshal", "9999999999", "david@marshall.com", false);

    public User getAdministrator() {
        return fAdministrator;
    }

    public User getUser() {
        return fUser;
    }
    ...
}
```

The tests would require a bean of `InMemoryRepository`. But wait, the beans can't access the getters from `InMemoryRepository`. Let's move the constructor and field definition to `IUserRepository`. The initialization would have to be done in the constructor of the derived class.

```java
public abstract class IUserRepository {

    private User fAdministrator;
    private User fUser;

    public User getAdministrator() {
        return fAdministrator;
    }

    public User getUser() {
        return fUser;
    }

    public void setAdministrator(User administrator) {
        this.fAdministrator = administrator;
    }

    public void setUser(User user) {
        this.fUser = user;
    }
    
    ...
    
}        
```

```java
public InMemoryRepository() {
    setAdministrator(new User(UUID.fromString("f994c61d-ebd1-463c-a8d8-ebe5989aa501"), "King Kong", "9999999999", "king@kong.com", true));
    setUser(new User(UUID.fromString("1109a8c8-49a3-4921-aa80-65e730d587fe"), "David Marshal", "9999999999", "david@marshall.com", false));
    users.add(getAdministrator());
    users.add(getUser());
}
```

The test will now use `fUserRepository.getAdministrator()` and `fUserRepository.getUser()` to get the defaults. Let's refactor the setters and getters to reflect that these two objects are default. The setters can be `protected` as we only want them to be updated from the derived classes

```java
/**
 * @return the default administrator account. Used for creation of new users and is used in the tests.
 */
public User getDefaultAdministrator() {
    return fAdministrator;
}

/**
 * @return the default user account. Used in tests. 
 */
public User getDefaultUser() {
    return fUser;
}

/**
 * This must be set when a new repository is introduced, else the tests will fail.
 * @param administrator the default administrator account.
 */
protected void setDefaultAdministrator(User administrator) {
    this.fAdministrator = administrator;
}

/**
 * This must be set when a new repository is introduced, else the tests will fail.
 * @param user the default user account.
 */
protected void setDefaultUser(User user) {
    this.fUser = user;
}
```

You can update the `PostgresRepository` to set the default users.

```java
public PostgresRepository(UserRepositoryJdbcDaoSupport jdbcDaoSupport,
                          String createUserSproc,
                          String getUserSproc) {
    fJdbcDaoSupport = jdbcDaoSupport;
    fCreateUserSproc = createUserSproc;
    fGetUserSproc = getUserSproc;
    setDefaultAdministrator(createUserIfNotPresent(UUID.fromString("f994c61d-ebd1-463c-a8d8-ebe5989aa501"),
            new UserDto("King Kong", "9999999999", "king@kong.com", true)));
    setDefaultUser(createUserIfNotPresent(UUID.fromString("1109a8c8-49a3-4921-aa80-65e730d587fe"),
            new UserDto("David Marshal", "9999999999", "david@marshall.com", false)));
}
```

This was a long exercise but we have better code now.

## Consume the endpoint

Our controller endpoints should now be working for the create and get users. Let's launch the service and see how they are responding.

### Create user

```java
curl -X POST \
  http://localhost:8080/v1/f994c61d-ebd1-463c-a8d8-ebe5989aa501/user/ \
  -H 'cache-control: no-cache' \
  -H 'content-type: application/json' \
  -H 'postman-token: 2bb80a07-e752-9054-970d-31363bec4969' \
  -d '{
    "administrator": false,
    "phone": "9999999999",
    "email": "david@marshall.com",
    "name": "David Marshal"
}'
```

```java
{
    "administrator": false,
    "phone": "9999999999",
    "email": "david@marshall.com",
    "name": "David Marshal",
    "id": "a04bb531-3d3d-4e45-a47b-5e844f2c81a1"
}
```

Note that this id is different. You can also query in pgAdmin to see that there are multiple users present in the database.

### Get user

```java
curl -X GET \
  http://localhost:8080/v1/1109a8c8-49a3-4921-aa80-65e730d587fe/user/1109a8c8-49a3-4921-aa80-65e730d587fe \
  -H 'cache-control: no-cache' \
  -H 'content-type: application/json' \
  -H 'postman-token: 3df8f54f-1702-7e59-18be-4962f2123b8c'
```

```java
{
    "administrator": false,
    "phone": "9999999999",
    "email": "david@marshall.com",
    "name": "David Marshal",
    "id": "1109a8c8-49a3-4921-aa80-65e730d587fe"
}
```

We've now verified a couple of operations. I'm assuming the failures would be getting handled as they were. You can consume those cases and see for yourself.

Let's move on to implementing `updateUser()` and `deleteUser()` and then we can look for more refactor.

