# Postgres

## Configuration

### Required tools

The tests should be able to interact with Postgres and that is possible using Docker containers. We'd required to install Postgres and Docker and have those services running to get the job done. You can choose to install any version of those but I'm choosing Postgres 13.1 at the time of this documentation. The installations seemed pretty straight-forward. You can keep everything as defaults. During the installation, it will ask you to install a few other tools, install all those. It'll also ask you to set a password, do that as you'd like it to be. pgAdmin is the tool to visualise our database using a url. You can launch that and notice that PostgreSQL 13 is available there.

I downloaded Docker Desktop as well. It'll ask you to use a DockerHub account to sign in. For the purposes of this exercise, we don't need to push anything to the DockerHub, we just need docker to be running and provide us test containers for our integration tests.

### Import dependencies

Now, our project needs to have all the dependencies imported before we can start using containers.

Let's add the dependencies for the testcontainers to our pom file.

We'd need the postgresql application dependency along with the postgres testcontainers to actually have the service interact with postgres.

As we're using junit for our tests, the testcontainers dependency for junit is also required.

I also extracted out the version numbers in the maven `properties` tag to easily alter them without looking for the dependency in the soon to be huge dependencies list.

```markup
<properties>
		<java.version>10</java.version>
		<jackson.version>2.10.0</jackson.version>
		<postgresql.version>42.2.18</postgresql.version>
		<testcontainer.version>1.15.1</testcontainer.version>
</properties>

<dependency>
   <groupId>org.postgresql</groupId>
   <artifactId>postgresql</artifactId>
</dependency>

<dependency>
   <groupId>org.testcontainers</groupId>
   <artifactId>postgresql </artifactId>
   <version>${testcontainer.version}</version>
   <scope>test</scope>
</dependency>

<dependency>
   <groupId>org.testcontainers</groupId>
   <artifactId>junit-jupiter</artifactId>
   <version>${testcontainer.version}</version>
   <scope>test</scope>
</dependency>
```

### Database credentials

We would need to specify the application properties to use the database in our application and the tests. In the project's main resources folder, update the file `application.properties`. If you don't have that file, create a new one.

```yaml
application.name = learn-spring-service

# Database configuration
postgres_host=${POSTGRES_HOST:localhost}
database.name=${DATABASE_NAME:learn_spring}
database.url=jdbc:postgresql://${postgres_host}:5432/${database.name}?stringtype=unspecified&ApplicationName=${application.name}
database.username=${LEARN_SPRING_SERVICE_DATABASE_USERNAME:user_crud}
database.password=${LEARN_SPRING_SERVICE_DATABASE_PASSWORD:user_crud_password}
spring.datasource.driver-class-name=org.postgresql.Driver

# Disable feature detection by this undocumented parameter.
# Check the org.hibernate.engine.jdbc.internal.JdbcServiceImpl.configure method for more details.
spring.jpa.properties.hibernate.temp.use_jdbc_metadata_defaults=false
```

If you have multiple versions of Postgres installed, the port would be different for all. You can query the following using pgAdmin to let the service know which one to interact with.

```sql
select inet_server_addr( ), inet_server_port( );
```

In the project's test resources folder, create a new file `application-test.properties`

```yaml
database.username=learn_crud
database.password=learn_crud@123
test.container.postgres=postgres:13.1
```

### Bean configuration

The suffix test creates a new spring profile which can be used by the tests. You'd need to specify this profile in the `TestBase` class using the following annotation

```java
@ActiveProfiles("test")
```

Now, the test classes will be using different credentials for interacting with Postgres.

Let's create a configuration class to separately initialize the Postgres container.

```java
@Testcontainers
@Configuration
public class TestContainerConfig {

    private static PostgreSQLContainer fSqlContainer;
    
}
```

We can import this class in our tests by modifying the `@SpringBootTest` annotation on `TestBase`. It is looking like the following for me right now. By any chance the imports are not getting detected, Invalidate and Restart your IDE.

```java
@SpringBootTest(
        webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT,
        properties = "spring.main.allow-bean-definition-overriding=true",
        classes = {AppConfig.class, TestContainerConfig.class})
```

We need to wire up the properties to have a new container instance ready to be used. We can add a Bean to the TestContainerConfig.

```java
@Bean("data_source")
DataSource configureTestDataSource(
        @Value("${database.url}") final String jdbcUrl,
        @Value("${database.username}") final String username,
        @Value("${database.password}") final String password,
        @Value("${test.container.postgres}") final String postgresContainer) {
    if (fSqlContainer == null) {
        fSqlContainer = new PostgreSQLContainer(postgresContainer);
        fSqlContainer.start();
    }
    // substring(5) strips the "jdbc:" from the front of the URI, otherwise .create() fails.
    final var queryParams = URI.create(jdbcUrl.substring(5));
    final var url = fSqlContainer.getJdbcUrl() + "&" + queryParams.getRawQuery();
    return DataSourceBuilder.create()
            .url(url).username(username).password(password).build();
}
```

We have the container ready and wired up with our properties. If you run `mvn clean install` now, you'll notice the logs interacting with Docker but it'll fail with the following error.

```java
2021-01-09 19:48:58.776  WARN 27905 --- [           main] ConfigServletWebServerApplicationContext : Exception encountered during context initialization - cancelling refresh attempt: org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'data_source' defined in com.atul.gitbook.learn.TestContainerConfig: Bean instantiation via factory method failed; nested exception is org.springframework.beans.BeanInstantiationException: Failed to instantiate [javax.sql.DataSource]: Factory method 'configureTestDataSource' threw exception; nested exception is java.lang.IllegalStateException: No supported DataSource type found
```

This is because there is no database available to interact. We need to import the `spring jdbc` dependency to enable that.

```markup
<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
```

The project will build correctly now.

### Initialization script

We have only just wired up the user credentials to our datasource, the user is currently not present in the database. We'd need to add sql script to handle database initialization for us. Also we're using UUID for the userIds, Postgres doesn't have in-built support to do that. We can add an extension to do that for us. The following sql script with the name `test-db-creation.sql` will do that for us.

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
SET timezone TO 'UTC';

CREATE USER learn_crud WITH PASSWORD 'learn_crud@123';
ALTER USER learn_crud WITH SUPERUSER;
```

The script should be wired up with our test container. Modify the container initialization to have the following entries.

```java
if (fSqlContainer == null) {
    fSqlContainer = new PostgreSQLContainer(postgresContainer);
    final var createScript = this.getClass().getClassLoader().getResource("test-db-creation.sql");
    if (createScript != null) {
        fSqlContainer.withInitScript("test-db-creation.sql");
    }
    fSqlContainer.start();
}
```

### Migration script configuration

Before we begin migrating from the in-memory database, it'd be nice to have to tool which can handle the sql migration effectively for us. One of the leading tools to do that is flyway. Let's import and configure that and then we can start writing the schema creation sql script.

```markup
<properties>
...    
    <flyway.version>7.4.0</flyway.version>
</properties>

...

<dependency>
	<groupId>org.flywaydb</groupId>
	<artifactId>flyway-core</artifactId>
</dependency>
```

The migration scripts stay in a folder as project resources. Let's use a folder `db/migration` which will contain all the scripts in a serialized fashion. Append the following to the application.properties

```yaml
# This disables flyway's autoconfig migration
spring.flyway.enabled=false
flyway.default.locations=db/migration
```

 Add the following to configure migration

```java
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
```

Flyway must be configured before the dataSource is returned from the bean. Modify the datasource bean creation to have a new parameter and call the configureFlyway method before the datasource is returned.

```java
@Bean("data_source")
DataSource configureTestDataSource(
        ....
        @Value("${flyway.default.locations}") final Set<String> flywayLocations) {
    ...
    final var url = fSqlContainer.getJdbcUrl() + "&" + queryParams.getRawQuery();
    final var dataSource = DataSourceBuilder.create()
            .url(url).username(username).password(password).build();
    configureFlyway(dataSource, flywayLocations);
    return dataSource;
}
```

## Project Status

We're done with the configuration now. Our tests verify the CRUD operations of the user service. When it comes to databases, we'd need to have tables before we can perform any operations. But, after creating the schema, how do we test whether the schema is created correctly. We'll be using the pgAdmin tool to visualise that using a web browser url.

The flyway migrations run when the service is launched and all the scripts run in the order mentioned by the file names.

Databases are a very critical piece of infrastructure and it's prudent to have safeguards associated with it. Flyway has some protection rules for the systems in production. Under which the service launch will fail if an already been migrated script is modified. We can add new script if we want to modify some behaviour.

For development purposes, it is recommended to have the script ready before migrating it using service launch. In the tests, this is not a problem as a new container is made available for every test run depending on the Test Lifecycle mentioned.

Its general practice to work on one script at a time and make it perfect before moving onto the next. Our approach of launching the application every time we make a test pass could prove to be an hinderance when we're working on the test which interact with the script. But not to worry, we'll look into an approach to delete the scripts from the list of migrated scripts using pgAdmin and get out of trouble.

