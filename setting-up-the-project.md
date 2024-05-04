# Setting up the project

## Maven initialisation using pom.

Spring boot uses Maven for dependency management and Maven defines them in a pom file. Let's begin by creating a spring module using Spring Initializr([https://start.spring.io\\](https://start.spring.io/)).

<figure><img src=".gitbook/assets/Screenshot 2024-05-03 at 1.54.29 AM.png" alt=""><figcaption><p>Project Metadata Entries</p></figcaption></figure>

After entering the project metadata, you should have a pom file like the following. Spring Initializr adds a couple of dependencies as the starter kit.



```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>3.2.5</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>
	<groupId>com.amit.gitbook</groupId>
	<artifactId>learn</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>learn</name>
	<description>Learn Spring Boot using Test Driven Description</description>
	<properties>
		<java.version>17</java.version>
	</properties>
	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
	</dependencies>

	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>

</project>
```

There are certain maven commands that could be helpful.

`mvn clean install` : Builds the jar for the project to be used to launch the application

`mvn spring-boot:run` : Launches the spring boot application.

`mvn test` : Run all the tests in the module.

You will notice that there is one test present in the `test/` package named `contextLoads()`. You can run all the tests and see that this one is passing. This passing tells you that the environment is all working and we are ready to start building the components.

### Project Structure <a href="#project-structure" id="project-structure"></a>

Let's add a new package `users` in the sources folder and add the same in the test folder as well. Having the same package in both ensures package access across source roots and it makes sense to have the tests in the same package as the code.

Let's add a `UserServiceTest` class in the `tests/users/` folder where the tests related to the UserService would reside.

You can remove the file containing `contextLoads()` now as we won't be testing the application class. We can assume that the application is working if the application starts up.
