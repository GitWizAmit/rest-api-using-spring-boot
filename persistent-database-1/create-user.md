# Create User

## Repository creation

We need a new repository class to handle Postgres operations. Let's add the following

```java
public class PostgresRepository implements IUserRepository {
    @Override
    public User createUser(UserDto userDto) {
        return null;
    }

    @Override
    public User getUser(UUID id) {
        return null;
    }

    @Override
    public void updateUser(UUID id, UpdateUserDto userDto) {

    }

    @Override
    public void deleteUser(UUID id) {

    }
}
```

### Try to run the test

```java
[INFO] 
[INFO] Results:
[INFO] 
[ERROR] Failures: 
[ERROR]   ControllerTest.testCreateUserWhenRequesterDoesNotExist:41 Status expected:<401> but was:<500>
[ERROR]   ControllerTest.testCreateUserWhenRequesterExistsButIsNotAdministrator:47 Status expected:<403> but was:<500>
[ERROR]   ControllerTest.testCreateUserWhenRequesterIsAdministrator:54 Range for response status value 500 expected:<SUCCESSFUL> but was:<SERVER_ERROR>
[ERROR]   ControllerTest.testDeleteUserWhenRequestIsDeletingOwnProfile:156 Range for response status value 500 expected:<SUCCESSFUL> but was:<SERVER_ERROR>
[ERROR]   ControllerTest.testDeleteUserWhenRequesterDoesNotExist:143 Status expected:<401> but was:<403>
[ERROR]   ControllerTest.testGetUserWhenRequesterDoesNotExist:77 Status expected:<401> but was:<500>
[ERROR]   ControllerTest.testGetUserWhenRequesterIsAdministratorAndUserIsPresent:95 Range for response status value 500 expected:<SUCCESSFUL> but was:<SERVER_ERROR>
[ERROR]   ControllerTest.testGetUserWhenRequesterIsDifferentFromUser:83 Status expected:<403> but was:<500>
[ERROR]   ControllerTest.testGetUserWhenRequesterIsFetchingOwnDetails:89 Range for response status value 500 expected:<SUCCESSFUL> but was:<SERVER_ERROR>
[ERROR]   ControllerTest.testUpdateUserWhenRequesterDoesNotExist:113 Status expected:<401> but was:<403>
[INFO] 
[ERROR] Tests run: 29, Failures: 10, Errors: 0, Skipped: 0
```

### Let's try to resolve the failure

We require a Create User script and have the `ADMINISTRATOR` and `USER` objects present in the database and the `PostgresRepository` updated to interact with the Stored Procedure.

## Write a new script

```sql
CREATE OR REPLACE FUNCTION fn_user_create(entity_id UUID, body JSONB)
  RETURNS VOID AS
$$
BEGIN
  INSERT INTO users (id, name, phone, email, administrator)
  VALUES (entity_id, body ->> 'name', body ->> 'phone', body ->> 'email', (body ->> 'administrator') :: BOOLEAN);
END;
$$
LANGUAGE plpgsql;
```

If you run the tests, the migration would be applied successfully.

Add the following to `application.properties`

```yaml
#Stored Procedures
function.user.create=${FUNCTION_USER_CREATE:fn_user_create}
```

### Configure script invocation

There are various ways of executing a Jdbc command in java. One of them is using the following.

First create the callable statement for `createUser`.

```java
private <T> CallableStatement generateCreateUserCallableStatement(
        final String id,
        final T body,
        final String sproc,
        final Serializer<T> serializer,
        final Connection connection) throws SQLException {
    final var pgObject = buildPgObject(serializer.serialize(body));
    final var sql = String.format("{call %s(?, ?)}", sproc);
    final var cs = connection.prepareCall(sql);
    var param = 1;
    cs.setObject(param++, id);
    cs.setObject(param, pgObject);
    return cs;
}

private static PGobject buildPgObject(String requestJson) throws SQLException {
    final var jsonObject = new PGobject();
    jsonObject.setType("jsonb");
    jsonObject.setValue(requestJson);
    return jsonObject;
}
```

Modify the constructor for `PostgresRepository` and the `createUser` method.

```java
private final String fCreateUserSproc;

private final Serializer<UserDto> SERIALIZER_USER_DTO = Serializers.newJsonSerializer(UserDto.class);

public PostgresRepository(DataSource dataSource, String createUserSproc) {
    setDataSource(dataSource);
    this.fCreateUserSproc = createUserSproc;
}

@Override
public User createUser(UserDto userDto) {
    return createUser(UUID.randomUUID(), userDto);
}

private User createUser(UUID userId, UserDto userDto) {
    try {
        final var createUserCallableStatement = generateCreateUserCallableStatement(userId.toString(), userDto, fCreateUserSproc, SERIALIZER_USER_DTO, getConnection());
        createUserCallableStatement.execute();
    } catch (SQLException e) {
        e.printStackTrace();
    }
    return null;
}
```

The create method is not returning the created user yet as we'd need the `getUser` stored procedure for that. We can still test if user creation succeeds by attempting to create the `ADMINISTRATOR` in the constructor which runs when the service starts.

### Test script using pgAdmin

Add the following to the `PostgresRepository`'s constructor and run the service. There will be failing tests at the moment but don't worry about those, we'll fix each and every one of them.

```java
createUser(UUID.fromString("f994c61d-ebd1-463c-a8d8-ebe5989aa501"), new UserDto("King Kong", "9999999999", "king@kong.com", true));
```

If you query the users table, you'll see that the user with the above details is added in the database.

If you run the service again, you'll see the following error.

```java
org.postgresql.util.PSQLException: ERROR: duplicate key value violates unique constraint "users_pkey" Detail: Key (id)=(f994c61d-ebd1-463c-a8d8-ebe5989aa501) already exists.
```

There won't be any issue in launching the service but this error will come up every time the service is launched. We can add checks to not attempt administrator creation when administrator is already present but we'd require the `getUser` call working for that. Let's get that going now.

