---
layout: post
subheadline: Spring Boot Series
title: Stateless API Security with Spring Boot, Part 1
teaser: Example project for securing REST endpoints with an Authorization header for API security.
categories:
  - articles
tags:
  - spring boot
  - security
  - development
related:
  - /blog/articles/stateless-api-security-with-spring-boot-part-2.html
  - /blog/articles/custom-authentication-with-spring-boot.html
  - /blog/articles/custom-authorization-with-spring-boot.html
image:
    title: posts/spring-boot.png
    thumb: posts/spring-boot-t.png
    homepage: posts/spring-boot.png
---

## Introduction

In this series of articles, we'll discuss how to implement pure (stateless) API security for your REST application in Spring Boot using an Authorization header and a custom security scheme.

## Basic Authentication

First, let's discuss the original precursor in the web security space, Basic authentication. Basic uses an `Authorization` header (defined in [RFC 2617](http://www.ietf.org/rfc/rfc2617.txt)) of the form:

```
credentials        = "Basic" basic-credentials
basic-credentials  = base64-user-pass
base64-user-pass   = <base64 encoding of user-pass>
user-pass          = userid ":" password
userid             = *<TEXT excluding ":">
password           = *TEXT
```

However, the current acceptable format for this header is defined in the same RFC as:

```
credentials        = auth-scheme #auth-param
auth-scheme        = token
auth-param         = token "=" ( token | quoted-string )
```

Notice that these schemes are not compatible. The Basic form is supported for legacy purposes to enable the continued use of the Basic authentication scheme.

Let's build an application that supports basic authentication first, and then evolve it to meet our end goals for a custom authentication scheme that is compatible with industry standards.

### Set Up

Let's define a build for our project. Here's a pom.xml skeleton to get us started:

```xml
<?xml version="1.0" encoding="utf-8" ?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>io.insource</groupId>
    <artifactId>basicauth-security-example</artifactId>
    <version>0.1.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>basicauth-security-example</name>
    <description>Example project for securing REST endpoints with basic auth security.</description>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>1.5.13.RELEASE</version>
        <relativePath />
    </parent>

    <repositories>
        <repository>
            <id>spring-plugins-releases</id>
            <url>http://repo.spring.io/plugins-release</url>
        </repository>
    </repositories>

    <properties>
        <java.version>1.8</java.version>
        <spring-boot.version>1.5.13.RELEASE</spring-boot.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
            </plugin>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>

        <pluginManagement>
            <plugins>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-compiler-plugin</artifactId>
                    <configuration>
                        <source>${java.version}</source>
                        <target>${java.version}</target>
                    </configuration>
                </plugin>
            </plugins>
        </pluginManagement>
    </build>
</project>
```

Let's also define an entry point for our application:

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

Lastly, let's define an endpoint we want to be able to secure:

```java
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api/v1")
public class HelloController {
    @GetMapping("/hello")
    public String sayHello() {
        return "Hello, World";
    }
}
```

### Add Basic Authentication

Spring Boot makes it easy to create secure-by-default applications. Though they may not be useful beyond the prototyping stage. Let's add a dependency to our pom:

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
```

With that change in place, we can run this application and it will be secure out of the box. We must use Basic authentication to access the `/api/v1/hello` endpoint. The username is "user" by default, with a generated password printed out to the console.

```
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::       (v1.5.13.RELEASE)

...
2018-05-28 10:36:37.977  INFO 62012 --- [           main] b.a.s.AuthenticationManagerConfiguration : 

Using default security password: dc458486-180c-4653-a5d7-d4ca5e0eab3b

...
```

### Customize Basic Authentication

If we want to customize the username/password, we can do so by creating an `application.yml`, as follows:

```yml
security:
  basic:
    enabled: true
  user:
    name: user
    password: password
```

Note that it's also possible to encrypt the password in this file (and other techniques), which is outside the scope of this article.

We can also customize other aspects of the security model, and this is where we get into the delicate inner workings of Spring Security. Let's also add a Java configuration:

```java

import org.springframework.boot.autoconfigure.security.SecurityProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.authentication.builders.AuthenticationManagerBuilder;
import org.springframework.security.config.annotation.authentication.configurers.provisioning.InMemoryUserDetailsManagerConfigurer;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.config.http.SessionCreationPolicy;

@Configuration
@EnableWebSecurity
@EnableConfigurationProperties
public class WebSecurityConfiguration extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        BasicAuthProperties properties = basicAuthProperties();
        http.requestMatchers().antMatchers(properties.getPath())
            .and()
                .authorizeRequests()
                    .anyRequest().authenticated()
            .and()
                .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            .and()
                .httpBasic().realmName(properties.getRealm())
            .and()
                .csrf().disable();
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        BasicAuthProperties properties = basicAuthProperties();
        InMemoryUserDetailsManagerConfigurer<AuthenticationManagerBuilder> builder = auth.inMemoryAuthentication();
        for (SecurityProperties.User user : properties.getUsers()) {
            builder.withUser(user.getName())
                .password(user.getPassword())
                .roles(user.getRole().toArray(new String[0]));
        }
    }

    @Bean
    public BasicAuthProperties basicAuthProperties() {
        return new BasicAuthProperties();
    }
}
```

Here, we've defined some very elementary configuration to override the defaults. There are two notable aspects of this config.

First, we've set the `SessionCreationPolicy` to `STATELESS`. This effectively turns off session management and forces the client to authenticate on every API call. We've also disabled CSRF to make it clear that we won't need them. This step is optional as without sessions, we can't have CSRF tokens, but it just helps to be clear what our intention is with this security model.

Second, we've defined a new `@ConfigurationProperties` of `BasicAuthProperties` and enabled it with `@EnableConfigurationProperties`. We've also loaded a list of users in memory from external properties. Here's what that class looks like:

```java
import org.springframework.boot.autoconfigure.security.SecurityProperties;
import org.springframework.boot.context.properties.ConfigurationProperties;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

@ConfigurationProperties("app.security.basic")
public class BasicAuthProperties {
    /**
     * HTTP basic realm name.
     */
    private String realm = "Spring";

    /**
     * Comma-separated list of paths to secure.
     */
    private String[] path = new String[] { "/**" };

    /**
     * Basic authentication users.
     */
    private List<SecurityProperties.User> users = new ArrayList<>(Collections.singletonList(
        new SecurityProperties.User()
    ));

    public String getRealm() {
        return realm;
    }

    public void setRealm(String realm) {
        this.realm = realm;
    }

    public String[] getPath() {
        return path;
    }

    public void setPath(String[] path) {
        this.path = path;
    }

    public List<SecurityProperties.User> getUsers() {
        return users;
    }

    public void setUsers(List<SecurityProperties.User> users) {
        this.users = users;
    }
}
``` 

You may also want to add this dependency to generate metadata from your `@ConfigurationProperties`:

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-configuration-processor</artifactId>
            <optional>true</optional>
        </dependency>
```

This allows us to inject our own configuration into the `application.yml` file to control our custom Java config. We've borrowed just enough from the default config (including `SecurityProperties.User`) to be consistent. We can add this to `application.yml`:

```yml
app:
  security:
    basic:
      realm: MyRealm
      path: /**
      users:
        - name: user
          password: password
          roles: USER
        - name: admin
          password: admin
          roles: ADMIN
```

In practice, you'd probably want to get away from hard-coded credentials (though it can work in some instances using profiles and CI/CD pipelines). This is beyond the scope of this article, though I may provide an example of how to do this later.

## Conclusion

In this article, we've learned how to add Basic authentication to any Spring Boot application. In the next article, we'll learn how to evolve that into our own custom authentication scheme, in which we provision tokens for API consumers.
