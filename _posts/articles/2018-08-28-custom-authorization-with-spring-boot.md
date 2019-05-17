---
layout: post
subheadline: Spring Boot Series
title: Custom Authorization with Spring Boot
teaser: Example project for securing REST endpoints with a custom authorization scheme.
teaser: 
categories:
  - articles
tags:
  - spring boot
  - security
  - development
related:
  - /blog/articles/stateless-api-security-with-spring-boot-part-1.html
  - /blog/articles/stateless-api-security-with-spring-boot-part-2.html
  - /blog/articles/custom-authentication-with-spring-boot.html
image:
    title: posts/spring-boot.png
    thumb: posts/spring-boot-t.png
    homepage: posts/spring-boot.png
---

## Introduction

In the [previous article](/blog/articles/custom-authentication-with-spring-boot.html), we discussed how to enable Restful username/password authentication. In this article, we'll discuss how to build a custom permissions system.

## Spring Security Authorization

Authorization in Spring Security is a large topic. The most common form of authorization available, one which has the most coverage in tutorials on the web, is role-based access control (RBAC). This is where you log in as a user with a particular role, say User or Admin, and are authorized to perform certain actions based on that role.

In Spring Security, the central interface for this concept is `GrantedAuthority`, which represents an "authority", usually a role, such as `ROLE_USER` or `ROLE_ADMIN`. In fact, `ROLE_` is so special that there are numerous aspects of Spring Security that look for it, and perform logic only when that prefix is present in the authority name. Here's an example of a route that is protected in this way:

```java
@RestController
public class HelloController {
    @GetMapping("/hello")
    @PreAuthorize("hasRole('ROLE_USER')")
    public String sayHello() {
        return "Hello, World";
    }
}
```

But what if you want to perform authorization that is more specific than something the user is granted when they log in? You might refer to this as domain object instance security. While there are numerous ways you can go deeper than role-based, you will ultimately be led to Spring Security <abbr title="Access Control List">ACL</abbr>.

This extension of Spring Security forces you to adopt a specific data model for persisting your authorization data so Spring Security can perform lookups and caching of that data to enable seamless integration of ACLs into your service layer.

But what if your permissions are not traditional? I'm not sure very many existing enterprises would have their authorization concepts cleanly isolated to a few database tables that Spring Security can talk to out of the box. In those cases, you need a custom solution that's simple to start with, and easy to extend.

Let's build such a solution.

## Set Up

Let's define a build for our project. Here's a pom.xml skeleton to get us started:

```xml
<?xml version="1.0" encoding="utf-8" ?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>io.insource</groupId>
    <artifactId>custom-permissions-example</artifactId>
    <version>0.1.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>custom-permissions-example</name>
    <description>Example project for method security with a custom permission evaluator.</description>

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
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-security</artifactId>
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

## Custom Authorization

### Data Model

Let's start with a contrived data model. This is a terrible example, but since I am not great at contriving non-incriminating examples, this will have to do. Say you have Supervisor and Employee data. Here are the models in this example:

```java
@Entity
public class Supervisor {
    @Id
    private String name;
    @ManyToOne(mappedBy = "supervisor")
    private List<Employee> employees;
    // Constructors
    // Getters and Setters
}

@Entity
public class Employee {
    @Id
    private String name;
    private List<String> permissions = new ArrayList<>();
    @OneToMany
    private Supervisor supervisor;
    // Constructors
    // Getters and Setters
}
```

In this example, our permissions (the identifiers we want to use to secure our API in certain situations) are on the objects we want to secure.

<p class="alert alert-info">
<strong>Note:</strong> This may not seem like a normal example, if you're coming from the ACL model perspective, but in the real world, this is often what you get. Data coming from a system you have little/no control over (with the exception of data mapping) which has its own concept of permissions. In fact, you may normally have a situation which is even less desirable than this, where your permissions need to be derived from other data in the model, such as a status or other boolean flags. Did I mention data mapping? Hopefully, you can at least map the data coming into your application through a materialized view or a mapping layer so it looks similar to this.
</p>

Next, let's define some way to retrieve our models. We'll use fake Spring Data `@Repository` stubs for example purposes.

```java
@Repository
public class SupervisorRepository extends JpaRepository<Supervisor, String> {
}

@Repository
public interface EmployeeRepository extends JpaRepository<Employee, String> {
    List<Employee> findBySupervisorName(String name);
}
```

### PermissionEvaluator

The heart of Method Security (role and permissions-based authorization at the method level) in Spring Security is the `PermissionEvaluator` interface. For reference, it looks like this:

```java
import java.io.Serializable;

import org.springframework.security.core.Authentication;

public interface PermissionEvaluator {
	boolean hasPermission(Authentication authentication, Object targetDomainObject, Object permission);
	boolean hasPermission(Authentication authentication, Serializable targetId, String targetType, Object permission);
}
```

Out of the box, there isn't really an implementation of this interface, other than the `DenyAllPermissionEvaluator` which isn't that helpful but happens to be the default. This is why you'll usually be steered in the direction of ACLs, which has a holistic implementation of this and other decision points within the authorization portion of the framework.

However, this interface is very easy to implement, though it is a bit archaic. In our case, we need a multi-faceted implementation to allow us to extend it very easily in the future. One way would be to take the Spring approach, and add a "manager" class, which delegates to other implementations. Let's do that.

```java
import org.springframework.security.access.PermissionEvaluator;
import org.springframework.security.access.expression.DenyAllPermissionEvaluator;
import org.springframework.security.core.Authentication;

import java.io.Serializable;
import java.util.Map;

public class PermissionEvaluatorManager implements PermissionEvaluator {
    private static final PermissionEvaluator denyAll = new DenyAllPermissionEvaluator();
    private final Map<String, PermissionEvaluator> permissionEvaluators;

    public PermissionEvaluatorManager(Map<String, PermissionEvaluator> permissionEvaluators) {
        this.permissionEvaluators = permissionEvaluators;
    }

    @Override
    public boolean hasPermission(Authentication authentication, Object targetDomainObject, Object permission) {
        PermissionEvaluator permissionEvaluator = permissionEvaluators.get(targetDomainObject.getClass().getSimpleName());
        if (permissionEvaluator == null) {
            permissionEvaluator = denyAll;
        }

        return permissionEvaluator.hasPermission(authentication, targetDomainObject, permission);
    }

    @Override
    public boolean hasPermission(Authentication authentication, Serializable targetId, String targetType, Object permission) {
        PermissionEvaluator permissionEvaluator = permissionEvaluators.get(targetType);
        if (permissionEvaluator == null) {
            permissionEvaluator = denyAll;
        }

        return permissionEvaluator.hasPermission(authentication, targetId, targetType, permission);
    }
}
```

This manager class implements the `PermissionEvaluator` interface, and composes itself using two things:

1. A list of delegates, each matching a specific target type. This allows us to write one `PermissionEvaluator` for each domain object we want to support.
2. A default delegate. The default behavior will be to deny access if we're asked, as that's the framework's fallback. This will be the `DenyAllPermissionEvaluator`.

If the list of delegates can't find a match (by type name), we simply fall back `denyAll`.

### Targeted PermissionEvaluator

Next, we need a way to target a specific type. We'll use simple logic and only match on the type name, as mentioned above. A simple extension will suffice for this:

```java
import org.springframework.security.access.PermissionEvaluator;

public interface TargetedPermissionEvaluator extends PermissionEvaluator {
    String getTargetType();
}
```

Using this interface, we can determine what type we support for each evaluator. Here's an `EmployeePermissionEvaluator`:

```java
@Component
public class EmployeePermissionEvaluator implements TargetedPermissionEvaluator {
    @Override
    public String getTargetType() {
        return Employee.class.getSimpleName();
    }

    @Override
    public boolean hasPermission(Authentication authentication, Object targetDomainObject, Object permission) {
        if (targetDomainObject != null && Employee.class.isAssignableFrom(targetDomainObject.getClass())) {
            Employee domainObject = (Employee) targetDomainObject;
            return domainObject.getPermissions().contains(permission.toString());
        }

        return false;
    }

    @Override
    public boolean hasPermission(Authentication authentication, Serializable targetId, String targetType, Object permission) {
        throw new UnsupportedOperationException("Not supported by this PermissionEvaluator: " + EmployeePermissionEvaluator.class);
    }
}
```

### Configure Method Security

With all that in place, we just need to configure the framework, and we can start securing APIs with Method Security and using other features of the authorization framework.

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.access.PermissionEvaluator;
import org.springframework.security.access.expression.method.DefaultMethodSecurityExpressionHandler;
import org.springframework.security.access.expression.method.MethodSecurityExpressionHandler;
import org.springframework.security.config.annotation.method.configuration.EnableGlobalMethodSecurity;
import org.springframework.security.config.annotation.method.configuration.GlobalMethodSecurityConfiguration;

import java.util.HashMap;
import java.util.List;
import java.util.Map;

@Configuration
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class MethodSecurityConfiguration extends GlobalMethodSecurityConfiguration {
    private final List<TargetedPermissionEvaluator> permissionEvaluators;

    @Autowired
    public MethodSecurityConfiguration(List<TargetedPermissionEvaluator> permissionEvaluators) {
        this.permissionEvaluators = permissionEvaluators;
    }

    @Override
    protected MethodSecurityExpressionHandler createExpressionHandler() {
        DefaultMethodSecurityExpressionHandler methodSecurityExpressionHandler = new DefaultMethodSecurityExpressionHandler();
        methodSecurityExpressionHandler.setPermissionEvaluator(permissionEvaluator());

        return methodSecurityExpressionHandler;
    }

    @Bean
    public PermissionEvaluator permissionEvaluator() {
        Map<String, PermissionEvaluator> map = new HashMap<>();

        // Build lookup table of PermissionEvaluator by supported target type
        for (TargetedPermissionEvaluator permissionEvaluator : permissionEvaluators) {
            map.put(permissionEvaluator.getTargetType(), permissionEvaluator);
        }

        return new PermissionEvaluatorManager(map);
    }
}
```

Let's break this down.

1. Enable Method Security. We also extend `GlobalMethodSecurityConfiguration` to override what we need to.
2. Autowire the list of `TargetedPermissionEvaluator`s found in the `ApplicationContext`.
3. Set up the `PermissionEvaluatorManager` by building a lookup table of `PermissionEvaluator` delegates.
4. Wire that up to the `DefaultMethodSecurityExpressionHandler` to be used by our `@Pre` and `@Post` method security.

<p class="alert alert-info">
<strong>Note:</strong> We can't simply component-scan the <code>PermissionEvaluatorManager</code> because we have numerous of <code>PermissionEvaluator</code>s on the classpath. The technique above ensures only the one we want is used. There are a few hacky ways to do this, but the above is the cleanest way to ensure our intended manager class is used.
</p>

### Secure your API

With the security layer configured, we can now use `@Pre` and `@Post` annotations to secure our API. Instead of the traditional placement of these annotations on the service layer, let's place them on our API directly. It's your choice, but putting them in controllers makes the authorization easier to document and understand, and makes your service layer more reusable (by choosing when to lock it down, and when not to).

```java
import org.springframework.http.HttpStatus;
import org.springframework.security.access.prepost.PostFilter;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.ResponseStatus;
import org.springframework.web.bind.annotation.RestController;

import javax.inject.Inject;
import java.util.List;
import java.util.Map;

@RestController
public class EmployeeController {
    @Inject
    private EmployeeRepository employeeRepository;

    @GetMapping("/supervisors/{name}/employees")
    public List<Employee> getEmployees(@PathVariable("name") String supervisorName) {
        return employeeRepository.findBySupervisorName(supervisorName);
    }

    @PostFilter("hasPermission(filterObject, #permission + '.' + #ext)")
    @GetMapping("/supervisors/{name}/employees/{permission}.{ext}")
    public List<Employee> getEmployeesWithPermission(@PathVariable("name") String name, @PathVariable("permission") String permission, @PathVariable("ext") String ext) {
        return employeeRepository.findBySupervisorName(name);
    }

    @PreAuthorize("hasPermission(#report['name'], 'Employee', 'expenseReport.allowed')")
    @PostMapping("/employees/expense-report")
    @ResponseStatus(HttpStatus.ACCEPTED)
    public void submitExpenseReport(@RequestBody Map<String, String> report) {
        // Do something with expense report data...
    }
}
```

In this example, we are using Method Security for two of our three routes.

1. `getEmployees()`: No authorization. Just list employees by supervisor.
2. `getEmployeesWithPermission()`: Uses a post-filter. This demonstrates reusing an existing query, but filtering the list based on permissions in our data model. Not applicable in all cases (for performance reasons), but useful for keeping LoC low and reusing queries with authorization logic for filtering.
3. `submitExpenseReport()`: Uses a pre-authorize check. Ensures the employee has expense-report permission and the current user (not necessarily _that_ employee) should be allowed to submit an expense report for that employee.

Again, these are contrived examples. But the important thing to note is how we've hooked into Spring Security to perform pre/post authorize or filtering logic with a very custom permissions scheme. You should note that with access to the `Authentication` in the `PermissionEvaluator`, you can make these checks specific to the currently logged in user, or not. It all depends on your requirements. I'll leave these custom implementations up to you. Happy coding!

## Conclusion

In this article, we've learned how to create an extensible permissions evaluation scheme with custom permission data in our model. We also learned how we can use that scheme to perform pre/post authorization logic, including filtering.
