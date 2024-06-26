# Users Schema

## Project Recap

We have the application and tests ready to interact with Postgres database.

As there are several moving parts in our system, I chose to work with the `InMemoryRepository` first to get the system up and running before we get into the whole mess of databases. With our approach, we also established that database decisions can be postponed at the inception of a service and only introduced when it's time to scale the system. Once we'll have all the infrastructure set up with all the concepts in our grasp, we can create a service without the trouble of using the `InMemoryRepository`. The migration from `InMemory` to `Postgres` would not be straightforward but it is a task that demonstrates how to work with different types of repositories with the same domain service code.

## Schema creation

Go to pgAdmin and create a database `learn_spring` first.

Let's begin replacing the `InMemoryRepository` by creating the schema first. We've the following User Model.

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

As this schema is a flyway script, we must follow the file name convention required by flyway which is `V{number}__{name}.sql` The number tells flyway to run the scripts in a certain order. For the users schema creation, we can work with `V1__create_users_schema.sql` with the following definition.

```sql
CREATE TABLE IF NOT EXISTS users
(
  id            UUID         NOT NULL,
  name          VARCHAR(100) NOT NULL,
  phone         VARCHAR(15)  NOT NULL,
  email         VARCHAR(50)  NOT NULL,
  administrator BOOLEAN      NOT NULL,
  CONSTRAINT users_pkey PRIMARY KEY (id)
);
```

Always run the tests first to be sure of the syntax present in the schema.

You should see an entry like the following in the logs on success.

```sql
2021-01-09 21:26:41.194  INFO 30624 --- [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "public" to version "1 - create users schema"
2021-01-09 21:26:41.245  INFO 30624 --- [           main] o.f.core.internal.command.DbMigrate      : Successfully applied 1 migration to schema "public" (execution time 00:00.079s)
```

If there is something wrong, the entry would have something like the following.

```sql
2021-01-09 21:28:09.446  INFO 30697 --- [           main] o.f.core.internal.command.DbMigrate      : Migrating schema "public" to version "1 - create users schema"
2021-01-09 21:28:09.466 ERROR 30697 --- [           main] o.f.core.internal.command.DbMigrate      : Migration of schema "public" to version "1 - create users schema" failed! Changes successfully rolled back.
```

 If you run the tests now with the schema with correct syntax, they should all be passing now.

Let's launch the application and see if the schema got created correctly using pgAdmin.

You might see an error like this

```sql
***************************
APPLICATION FAILED TO START
***************************

Description:

Failed to configure a DataSource: 'url' attribute is not specified and no embedded datasource could be configured.

Reason: Failed to determine suitable jdbc url


Action:

Consider the following:
        If you want an embedded database (H2, HSQL or Derby), please put it on the classpath.
        If you have database settings to be loaded from a particular profile you may need to activate it (no profiles are currently active).
```

This is because the `datasource` is being configured automatically by spring. Modify the following annotation to exclude Auto configuration.

```java
@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})
```

The application should be running now.

The application requires the Bean for `dataSource` to interact with the database. Add a config class for the application and import this in the `AppConfig`.

```java
@Configuration
public class DataSourceConfig {

    @Bean("data_source")
    DataSource configureTestDataSource(
            @Value("${database.url}") final String jdbcUrl,
            @Value("${database.username}") final String username,
            @Value("${database.password}") final String password,
            @Value("${flyway.default.locations}") final Set<String> flywayLocations) {
        final var dataSource = DataSourceBuilder.create()
                .url(jdbcUrl).username(username).password(password).build();
        configureFlyway(dataSource, flywayLocations);
        return dataSource;
    }

    private static void configureFlyway(
            final DataSource dataSource,
            final Set<String> flywayLocations) {
        Flyway.configure()
                .dataSource(dataSource)
                .baselineOnMigrate(true)
                .baselineVersion("0")
                .locations(flywayLocations.toArray(new String[]{}))
                .target(MigrationVersion.LATEST)
                .validateOnMigrate(true)
                .load()
                .migrate();
    }
}
```

```java
@Configuration
@Import(DataSourceConfig.class)
public class AppConfig {
...
}
```

### Run the application

The logs will show a failure

```java
2021-01-09 22:12:59.070  WARN 32222 --- [           main] ConfigServletWebServerApplicationContext : Exception encountered during context initialization - cancelling refresh attempt: org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'data_source' defined in class path resource [com/atul/gitbook/learn/DataSourceConfig.class]: Bean instantiation via factory method failed; nested exception is org.springframework.beans.BeanInstantiationException: Failed to instantiate [javax.sql.DataSource]: Factory method 'configureTestDataSource' threw exception; nested exception is org.flywaydb.core.internal.exception.FlywaySqlException: 
Unable to obtain connection from database: FATAL: password authentication failed for user "user_crud"
```

### Let's try to resolve the failure

We need a user created in postgres to interact with the database. Similar to the test user run the following queries using pgAdmin's Query Tool. The option will come up after you right-click on any of the databases.

```sql
CREATE USER user_crud WITH PASSWORD 'user_crud_password';
ALTER USER user_crud WITH SUPERUSER;
```

If every thing worked out correctly, there will be 2 tables created in the public schema of `learn_spring` database, `flyway_schema_history` & `users`.  You can query the following and see that the table is created with 0 entries.

```sql
select * from users;
```

You can also query the `flyway_schema_history`  and see the applied migration script.

## Next on Agenda

In the coming chapters, we'll look into developing the database repository acting on top of stored procedures.

