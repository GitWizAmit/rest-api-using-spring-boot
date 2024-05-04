# Get User

## Write a new script

The sproc being used in following is defined in the file `V2__get_users_by_id.sql`.

```sql
CREATE OR REPLACE FUNCTION public.fn_user_by_id(user_id UUID)
    RETURNS SETOF users AS
$$
DECLARE
    result users;
BEGIN
    EXECUTE FORMAT(
            'SELECT * from users where id = %L',
            user_id) INTO STRICT result;
    RETURN NEXT result;
EXCEPTION
    WHEN TOO_MANY_ROWS THEN
        RAISE EXCEPTION 'id must be unique in the table.';
    WHEN NO_DATA_FOUND THEN
        RETURN;
END;
$$ LANGUAGE plpgsql;
```

### Configure script invocation

We'd need a `RowMapper` to parse the db result into a User object.

```java
private RowMapper<User> getUserRowMapper() {
    return (final ResultSet rs, final int rowCount) -> {
        final var id = rs.getObject("id", UUID.class);
        final var name = rs.getString("name");
        final var phone = rs.getString("phone");
        final var email = rs.getString("email");
        final var administrator = rs.getBoolean("administrator");
        return new User(id, name, phone, email, administrator);
    };
}
```

We'd need the `CallableStatement` with which we'll be interacting with the stored procedure.

```java
private CallableStatement generateGetUserCallableStatement(
        final String id,
        final String sproc,
        final Connection connection) throws SQLException {
    final var sql = String.format("{call %s(?)}", sproc);
    final var cs = connection.prepareCall(sql);
    cs.setObject(1, id);
    return cs;
}
```

The getUser method in the `PostgresRepository` will call the statement and return the user if present and exception otherwise just like it did in the `InMemoryRepository`. This will take the sproc for getting user which we'll passing using the constructor and referenced in the `application.properties` file.

```yaml
function.user.get=${FUNCTION_USER_GET:fn_user_by_id}
```

```java
@Bean
IUserRepository configureUserRepository(DataSource dataSource,
                                        @Value("${function.user.create}") String createUserSproc,
                                        @Value("${function.user.get}") String getUserSproc) {
    return new PostgresRepository(dataSource, createUserSproc, getUserSproc);
}
```

```java
private final String fGetUserSproc;

public PostgresRepository(DataSource dataSource,
                          String createUserSproc,
                          String getUserSproc) {
    setDataSource(dataSource);
    this.fCreateUserSproc = createUserSproc;
    this.fGetUserSproc = getUserSproc;
    createUser(UUID.fromString("f994c61d-ebd1-463c-a8d8-ebe5989aa501"), new UserDto("King Kong", "9999999999", "king@kong.com", true));
    createUser(UUID.fromString("1109a8c8-49a3-4921-aa80-65e730d587fe"), new UserDto("David Marshal", "9999999999", "david@marshall.com", false));
}
```

```java
@Override
public User getUser(UUID id) {
    final PreparedStatementCreator getUserCallableStatement = (Connection connection) ->
            generateGetUserCallableStatement(id.toString(), fGetUserSproc, connection);
    final var userList = getJdbcTemplate().query(getUserCallableStatement, getUserRowMapper());
    final var user = userList.stream().findFirst();
    if (user.isPresent())
        return user.get();
    throw new NoSuchElementException();
}
```

The `createUser` method can be modified to use the `getUser` to return the created user.

```java
private User createUser(UUID userId, UserDto userDto) {
    try {
        final var createUserCallableStatement = generateCreateUserCallableStatement(userId.toString(), userDto, fCreateUserSproc, SERIALIZER_USER_DTO, getConnection());
        createUserCallableStatement.execute();
        return getUser(userId);
    } catch (SQLException e) {
        e.printStackTrace();
        return null;
    }
}
```

### Try to run the test

Most of the tests would be passing now. There must be something missing in our tests as we've not yet worked on the update and delete operations.Write a new script

The sproc being used in following is defined in the file `V2__get_users_by_id.sql`.

```sql
CREATE OR REPLACE FUNCTION public.fn_user_by_id(user_id UUID)
    RETURNS SETOF users AS
$$
DECLARE
    result users;
BEGIN
    EXECUTE FORMAT(
            'SELECT * from users where id = %L',
            user_id) INTO STRICT result;
    RETURN NEXT result;
EXCEPTION
    WHEN TOO_MANY_ROWS THEN
        RAISE EXCEPTION 'id must be unique in the table.';
    WHEN NO_DATA_FOUND THEN
        RETURN;
END;
$$ LANGUAGE plpgsql;
```

### Configure script invocation

We'd need a `RowMapper` to parse the db result into a User object.

```java
private RowMapper<User> getUserRowMapper() {
    return (final ResultSet rs, final int rowCount) -> {
        final var id = rs.getObject("id", UUID.class);
        final var name = rs.getString("name");
        final var phone = rs.getString("phone");
        final var email = rs.getString("email");
        final var administrator = rs.getBoolean("administrator");
        return new User(id, name, phone, email, administrator);
    };
}
```

We'd need the `CallableStatement` with which we'll be interacting with the stored procedure.

```java
private CallableStatement generateGetUserCallableStatement(
        final String id,
        final String sproc,
        final Connection connection) throws SQLException {
    final var sql = String.format("{call %s(?)}", sproc);
    final var cs = connection.prepareCall(sql);
    cs.setObject(1, id);
    return cs;
}
```

The getUser method in the `PostgresRepository` will call the statement and return the user if present and exception otherwise just like it did in the `InMemoryRepository`. This will take the sproc for getting user which we'll passing using the constructor and referenced in the `application.properties` file.

```yaml
function.user.get=${FUNCTION_USER_GET:fn_user_by_id}
```

```java
@Bean
IUserRepository configureUserRepository(DataSource dataSource,
                                        @Value("${function.user.create}") String createUserSproc,
                                        @Value("${function.user.get}") String getUserSproc) {
    return new PostgresRepository(dataSource, createUserSproc, getUserSproc);
}
```

```java
private final String fGetUserSproc;

public PostgresRepository(DataSource dataSource,
                          String createUserSproc,
                          String getUserSproc) {
    setDataSource(dataSource);
    this.fCreateUserSproc = createUserSproc;
    this.fGetUserSproc = getUserSproc;
    createUser(UUID.fromString("f994c61d-ebd1-463c-a8d8-ebe5989aa501"), new UserDto("King Kong", "9999999999", "king@kong.com", true));
    createUser(UUID.fromString("1109a8c8-49a3-4921-aa80-65e730d587fe"), new UserDto("David Marshal", "9999999999", "david@marshall.com", false));
}
```

```java
@Override
public User getUser(UUID id) {
    final PreparedStatementCreator getUserCallableStatement = (Connection connection) ->
            generateGetUserCallableStatement(id.toString(), fGetUserSproc, connection);
    final var userList = getJdbcTemplate().query(getUserCallableStatement, getUserRowMapper());
    final var user = userList.stream().findFirst();
    if (user.isPresent())
        return user.get();
    throw new NoSuchElementException();
}
```

The `createUser` method can be modified to use the `getUser` to return the created user.

```java
private User createUser(UUID userId, UserDto userDto) {
    try {
        final var createUserCallableStatement = generateCreateUserCallableStatement(userId.toString(), userDto, fCreateUserSproc, SERIALIZER_USER_DTO, getConnection());
        createUserCallableStatement.execute();
        return getUser(userId);
    } catch (SQLException e) {
        e.printStackTrace();
        return null;
    }
}
```

### Try to run the test

Most of the tests would be passing now. There must be something missing in our tests as we've not yet worked on the update and delete operations.

