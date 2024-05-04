# Delete User

## Write a new script

`V5__delete_users.sql`

```sql
CREATE OR REPLACE FUNCTION fn_user_delete(entity_id UUID)
  RETURNS VOID AS
$$
BEGIN
  EXECUTE FORMAT(
      'DELETE FROM users WHERE id = %L', entity_id);
END;
$$
LANGUAGE plpgsql;
```

### Configure script invocation

`application.properties`

```text
function.user.delete=${FUNCTION_USER_DELETE:fn_user_delete}
```

`AppConfig.java`

```java
@Bean
IUserRepository configureUserRepository(UserRepositoryJdbcDaoSupport jdbcDaoSupport,
                                        @Value("${function.user.create}") String createUserSproc,
                                        @Value("${function.user.get}") String getUserSproc,
                                        @Value("${function.user.update}") String updateUserSproc,
                                        @Value("${function.user.delete}") String deleteUserSproc) {
    return new PostgresRepository(jdbcDaoSupport, createUserSproc, getUserSproc, updateUserSproc, deleteUserSproc);
}
```

```java
private final String fDeleteUserSproc;

public PostgresRepository(...
                          String deleteUserSproc) {
    ...
    fDeleteUserSproc = deleteUserSproc;
    
    ...
    
}
```

We can reuse the callable statement generator used in the get user method as the definition of stored procedure for get and delete are same.

```java
@Override
public void deleteUser(UUID id) {
    try {
        final var deleteUserCallableStatement = generateGetUserCallableStatement(id.toString(), fDeleteUserSproc, fJdbcDaoSupport.getConn());
        deleteUserCallableStatement.execute();
    } catch (SQLException e) {
        e.printStackTrace();
    }
}
```

### Try to run the test

If you'd run the tests, all of them should be passing now. The CRUD operations for user service are now scalable with a persistent database. 

