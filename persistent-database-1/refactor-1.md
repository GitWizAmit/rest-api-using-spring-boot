# Refactor

## Refactor

You must have noticed the benefits of using stored procedures for executing the sql statements. Let's perform a bit of refactor in the `PostgresRepository` to reuse the methods for generating callable statements. We'll have the `UserRepositoryJdbcDaoSupport` to have the full capabilities of database CRUD operations.

### Method names

Let's change the `generateCreateUserCallableStatement` method name to `generateCallableStatementWithIdAndBody`.

Let's change the `generateGetUserCallableStatement` method name to `generateCallableStatementWithId`.

Let's move the above methods from `PostgresRepository` to `UserRepositoryJdbcDaoSupport`.

### Sproc configuration class

  Let's introduce a configuration class which will have the sprocs injected in `PostgresRepository`.

```java
public class RepoConfig {

    private final String fCreateSproc;
    private final String fGetSproc;
    private final String fUpdateSproc;
    private final String fDeleteSproc;

    public RepoConfig(String createSproc,
                      String getSproc,
                      String updateSproc,
                      String deleteSproc) {
        fCreateSproc = createSproc;
        fGetSproc = getSproc;
        fUpdateSproc = updateSproc;
        fDeleteSproc = deleteSproc;
    }

    public String getCreateSproc() {
        return fCreateSproc;
    }

    public String getGetSproc() {
        return fGetSproc;
    }

    public String getUpdateSproc() {
        return fUpdateSproc;
    }

    public String getDeleteSproc() {
        return fDeleteSproc;
    }
}
```

`AppConfig.java`

```java
@Bean
IUserRepository configureUserRepository(UserRepositoryJdbcDaoSupport jdbcDaoSupport,
                                        RepoConfig repoConfig) {
    return new PostgresRepository(jdbcDaoSupport, repoConfig);
}

@Bean
RepoConfig configureRepoConfig(@Value("${function.user.create}") String createUserSproc,
                               @Value("${function.user.get}") String getUserSproc,
                               @Value("${function.user.update}") String updateUserSproc,
                               @Value("${function.user.delete}") String deleteUserSproc) {
    return new RepoConfig(createUserSproc, getUserSproc, updateUserSproc, deleteUserSproc);
}
```

```java
private final RepoConfig fRepoConfig;
...
public PostgresRepository(UserRepositoryJdbcDaoSupport jdbcDaoSupport,
                          RepoConfig repoConfig) {
    fJdbcDaoSupport = jdbcDaoSupport;
    fRepoConfig = repoConfig;
    setDefaultAdministrator(createUserIfNotPresent(UUID.fromString("f994c61d-ebd1-463c-a8d8-ebe5989aa501"),
            new UserDto("King Kong", "9999999999", "king@kong.com", true)));
    setDefaultUser(createUserIfNotPresent(UUID.fromString("1109a8c8-49a3-4921-aa80-65e730d587fe"),
            new UserDto("David Marshal", "9999999999", "david@marshall.com", false)));
}
```

Use the getters from the `fRepoConfig` to inject the sprocs in the CRUD operation method.

### CRUD Operations

Now, we can move the CRUD operations to `UserRepositoryJdbcDaoSupport`

Add the `RepoConfig` to the `UserRepositoryJdbcDaoSupport`. 

```java
@Bean
UserRepositoryJdbcDaoSupport configureJdbc(DataSource dataSource,
                                           RepoConfig repoConfig) {
    return new UserRepositoryJdbcDaoSupport(dataSource, repoConfig);
}
```

#### Create operation

Let's refactor the create operation to `UserRepositoryJdbcDaoSupport`.

```java
public class UserRepositoryJdbcDaoSupport extends JdbcDaoSupport {

    private final RepoConfig fRepoConfig;

    public UserRepositoryJdbcDaoSupport(DataSource dataSource,
                                        RepoConfig repoConfig) {
        setDataSource(dataSource);
        fRepoConfig = repoConfig;
    }

    public <T> void create(
            final String id,
            final T body,
            final Serializer<T> serializer) throws SQLException {
        generateCallableStatementWithIdAndBody(id, body, fRepoConfig.getCreateSproc(), serializer, getConn()).execute();
    }
...
}
```

```java
public class PostgresRepository extends IUserRepository {
...
    private User createUser(UUID userId, UserDto userDto) {
        try {
            fJdbcDaoSupport.create(userId.toString(), userDto, SERIALIZER_USER_DTO);
            return getUser(userId);
        } catch (SQLException e) {
            e.printStackTrace();
            return null;
        }
    }
...
}
```

#### Get operation

Let's refactor the get operation to `UserRepositoryJdbcDaoSupport`.

```java
public class UserRepositoryJdbcDaoSupport extends JdbcDaoSupport {
...
    public <T> T get(
            final String id,
            final RowMapper<T> rowMapper) {
        final PreparedStatementCreator getCallableStatement = (Connection connection) ->
                generateCallableStatementWithId(id, fRepoConfig.getGetSproc(), connection);
        final var items = getJdbcTemplate().query(getCallableStatement, rowMapper);
        final var item = items.stream().findFirst();
        if (item.isPresent())
            return item.get();
        throw new NoSuchElementException();
    }
...
}
```

```java
public class PostgresRepository extends IUserRepository {
...

    @Override
    public User getUser(UUID id) {
        return fJdbcDaoSupport.get(id.toString(), getUserRowMapper());
    }
...
}
```

#### Update operation

 Let's refactor the update operation to `UserRepositoryJdbcDaoSupport`.

```java
public class UserRepositoryJdbcDaoSupport extends JdbcDaoSupport {
...
    public <T> void update(
            final String id,
            final T body,
            final Serializer<T> serializer) throws SQLException {
        final var updateCallableStatement = generateCallableStatementWithIdAndBody(id,
                body, fRepoConfig.getUpdateSproc(), serializer, getConn());
        updateCallableStatement.execute();
    }
...
}
```

```java
public class PostgresRepository extends IUserRepository {
...

    @Override
    public void updateUser(UUID userId, UpdateUserDto userDto) {
        try {
            fJdbcDaoSupport.update(userId.toString(), userDto, SERIALIZER_UPDATE_USER_DTO);
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
...
}
```

#### Delete operation

Let's refactor the delete operation to `UserRepositoryJdbcDaoSupport`.

```java
public class UserRepositoryJdbcDaoSupport extends JdbcDaoSupport {
...
    public void delete(final String id) throws SQLException {
        generateCallableStatementWithId(id, fRepoConfig.getDeleteSproc(), getConn()).execute();
    }
...
}
```

```java
public class PostgresRepository extends IUserRepository {
...

    @Override
    public void deleteUser(UUID id) {
        try {
            fJdbcDaoSupport.delete(id.toString());
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
...
}
```

### Clean up

We don't need the `getConn()` method now. Let's remove that and replace its usages with `getConnection()` which is available in the super class.

We also don't need the `fRepoConfig` in `PostgresRepository` anymore.

The `UserRepositoryJdbcDaoSupport` can be named as `RepositoryJdbcDaoSupport` to use it in other repositories.

## Validations

Let's add some restrictions in the `RepositoryJdbcDaoSupport`. When a sproc is not valid in the `RepoConfig`, let's not make the database call.

```java
public final class StoredProcedureValidator {
    private static final Pattern DEFAULT_SPROC_PATTERN = Pattern.compile("^[a-zA-Z][a-zA-Z0-9_]*$");

    private StoredProcedureValidator() {
    }

    public static void validateStoredProcedure(final String sproc) {
        Preconditions.validateNotNull(sproc);
        Preconditions.validateIsTrue(
                DEFAULT_SPROC_PATTERN.matcher(sproc).matches(),
                "StoredProcedure is not in correct format.");
    }
}
```

```java

public static void validateIsTrue(boolean expression, String message) {
    if (!expression) {
        throw new IllegalArgumentException(message);
    }
}
```

You can call the `validateStoredProcedure` as a precondition to the CRUD operation calls in the `RepositoryJdbcDaoSupport`.

## Package restructure

You can restructure the packages right now to easily use it in the notes service. I choose a package postgres for `RepoConfig`, `RepositoryJdbcDaoSupport` and `StoredProcedureValidator`.

## Project Status

The User Service is now fully refactored.

The controllers are extensively tested.

The repository is not scalable with a persistent database.

