# Update User

## Write a new script

`V4__update_user.sql`

```sql
CREATE OR REPLACE FUNCTION fn_user_update(entity_id UUID, body JSONB)
  RETURNS VOID AS
$$
BEGIN
  EXECUTE FORMAT(
      'UPDATE users SET name = %L, phone = %L, email = %L WHERE id = %L',
      body ->> 'name', body ->> 'phone', body ->> 'email', entity_id);
END;
$$
LANGUAGE plpgsql;
```

### Configure script invocation

`application.properties`

```yaml
function.user.update=${FUNCTION_USER_UPDATE:fn_user_update}
```

`AppConfig.java`

```java
@Bean
IUserRepository configureUserRepository(UserRepositoryJdbcDaoSupport jdbcDaoSupport,
                                        @Value("${function.user.create}") String createUserSproc,
                                        @Value("${function.user.get}") String getUserSproc,
                                        @Value("${function.user.update}") String updateUserSproc) {
    return new PostgresRepository(jdbcDaoSupport, createUserSproc, getUserSproc, updateUserSproc);
}
```

```java
private final String fUpdateUserSproc;

public PostgresRepository(...
                          String updateUserSproc) {
    ...
    fUpdateUserSproc = updateUserSproc;
    
    ...
    
}
```

We can reuse the callable statement generator used in the create user method as the definition of stored procedure for create and update are same.

```java
private final Serializer<UpdateUserDto> SERIALIZER_UPDATE_USER_DTO = Serializers.newJsonSerializer(UpdateUserDto.class);

@Override
public void updateUser(UUID userId, UpdateUserDto userDto) {
    try {
        final var updateUserCallableStatement = generateCreateUserCallableStatement(userId.toString(),
                userDto, fUpdateUserSproc, SERIALIZER_UPDATE_USER_DTO, fJdbcDaoSupport.getConn());
        updateUserCallableStatement.execute();
    } catch (SQLException e) {
        e.printStackTrace();
    }
}
```

### Try to run the test

The tests should all be passing now.

