---
layout: post
subheadline: Spring Boot Series
title: Stateless Sessions with Spring Boot
teaser: Example project for stateless session propagation.
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

In the [previous article](/blog/articles/custom-authentication-with-spring-boot.html), we discussed how to build a custom permissions system. In this article, we'll discuss how to use Zuul's reverse-proxy functionality to propagate session information in a stateless way.

## Spring Session

Spring is a great framework. It has lots of built-in features and optional libraries that can be added to enable new functionality. Security and session management are two great examples of this. If you are using Spring Boot, it's as easy as adding one of them (e.g. `spring-session-core`) to your classpath, and a default configuration kicks in. Further, if you add one of the extensions (like `spring-session-data-redis`) and an annotation (such as `@EnableRedisHttpSession`), you can pretty much transparently persist session state to the datastore of your choice without any other changes to your application.

Another great feature is propagating sessions. In a microservices architecture, you've got a lot of tiny services all doing their thing. What if you want them all to know about the currently logged-in user? I remember discussing this with a co-worker a year or so ago, when working on a new authentication system for a microservices architecture redesign of our system. It sounded theoretically possible, and solutions utilizing Spring Session or OAuth and JWTs seemed promising.

But what if there was a better way to share sessions within a microservices architecture?

Let's build such a solution.

## Set Up

Let's define a build for our project. In this example, we'll actually need two projects, which we'll get to in a minute. For our first project, here's a pom.xml skeleton to get us started:

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
    <description>Example project for stateless session propagation.</description>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.5.RELEASE</version>
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

## Security Configuration

Let's also add some security configuration to our project for example purposes. First, let's use pre-authentication similar to what we explored in the article [Stateless API Security with Spring Boot, Part 2]({% post_url /articles/2018-05-31-stateless-api-security-with-spring-boot-part-2 %}).

```java
import io.insource.spring.ws.examples.gateway.service.SimpleUserDetailsService;
import io.insource.spring.ws.examples.gateway.support.Http401AuthenticationEntryPoint;

import org.springframework.context.annotation.Configuration;
import org.springframework.core.annotation.Order;
import org.springframework.http.HttpStatus;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.AuthenticationProvider;
import org.springframework.security.authentication.ProviderManager;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.core.userdetails.UserDetailsByNameServiceWrapper;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.web.AuthenticationEntryPoint;
import org.springframework.security.web.authentication.logout.HttpStatusReturningLogoutSuccessHandler;
import org.springframework.security.web.authentication.preauth.PreAuthenticatedAuthenticationProvider;
import org.springframework.security.web.authentication.preauth.PreAuthenticatedAuthenticationToken;
import org.springframework.security.web.authentication.preauth.RequestHeaderAuthenticationFilter;

import java.util.Collections;

@Configuration
@EnableWebSecurity
@Order(-1)
public class
WebSecurityConfiguration extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.antMatcher("/**")
            .addFilterAfter(preAuthenticationFilter(), RequestHeaderAuthenticationFilter.class)
            .authorizeRequests()
                .antMatchers("/api/v1/anonymous").permitAll()
                .anyRequest().authenticated()
            .and()
                .exceptionHandling().authenticationEntryPoint(authenticationEntryPoint())
            .and()
                .logout().logoutUrl("/api/v1/logout").logoutSuccessHandler(logoutSuccessHandler())
            .and()
                .csrf().disable();
    }

    public RequestHeaderAuthenticationFilter preAuthenticationFilter() {
        RequestHeaderAuthenticationFilter preAuthenticationFilter = new RequestHeaderAuthenticationFilter();
        preAuthenticationFilter.setPrincipalRequestHeader("X-USERNAME");
        preAuthenticationFilter.setAuthenticationManager(authenticationManager());
        preAuthenticationFilter.setExceptionIfHeaderMissing(false);

        return preAuthenticationFilter;
    }

    @Override
    protected AuthenticationManager authenticationManager() {
        return new ProviderManager(Collections.singletonList(authenticationProvider()));
    }

    public AuthenticationProvider authenticationProvider() {
        PreAuthenticatedAuthenticationProvider authenticationProvider = new PreAuthenticatedAuthenticationProvider();
        authenticationProvider.setPreAuthenticatedUserDetailsService(userDetailsServiceWrapper());
        authenticationProvider.setThrowExceptionWhenTokenRejected(false);

        return authenticationProvider;
    }

    public UserDetailsByNameServiceWrapper<PreAuthenticatedAuthenticationToken> userDetailsServiceWrapper() {
        return new UserDetailsByNameServiceWrapper<>(userDetailsService());
    }

    public UserDetailsService userDetailsService() {
        return new SimpleUserDetailsService();
    }

    public AuthenticationEntryPoint authenticationEntryPoint() {
        return new Http401AuthenticationEntryPoint("MyRealm");
    }

    public HttpStatusReturningLogoutSuccessHandler logoutSuccessHandler() {
        return new HttpStatusReturningLogoutSuccessHandler(HttpStatus.NO_CONTENT);
    }
}
```

Remember that this security configuration is just an example. Whatever authentication scheme you are using should work fine here.

Next, let's add some authentication routes for our documentation tool to latch onto. These don't actually perform any authentication-related tasks, as that is all handled by the Spring Security filter chain.

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestHeader;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseStatus;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api/v1")
public class LoginController {
    private static final Logger log = LoggerFactory.getLogger(LoginController.class);

    @PostMapping("/login")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void login(@RequestHeader("X-USERNAME") String username) {
        log.info("User {} successfully logged in.", username);
    }

    @PostMapping("/logout")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void logout() {
        // Just for our chosen documentation tool, not actually invoked.
    }
}
```

## Microservice All the Things!

With the above set-up in place, let's create a microservice to represent one of many stateless services we'll deploy within our architecture.
Each microservice will be fully session-aware and yet require no setup or overhead.

The second project is just a normal web project, so head over to [Spring Initializr](https://start.spring.io) and select Web as a dependency to generate the project structure.
Once you've got the second project up and running, we'll add a controller that will eventually be able to utilize session state.

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

Very uninteresting, I know. But let's make it interesting by first discussing what we plan to do with this API endpoint.
The first thing to notice is that it takes no parameters. Why is this important, you ask?
To answer that question, let's examine a fundamental challenge in building APIs that are both re-usable and easy to consume.

## The Proverbial Hypothetical

<div class="alert alert-info" role="alert">
    TL;DR; The optimal solution is not your first thought; read on to learn more.
</div>

Imagine you are building an API that manages all sorts of different things for the currently logged in user.
You may need to return a list of students for a teacher and a list of tests that need to be graded, as well as submit attendance reports and manage parent-teacher topics and feedback, etc.
If you plan to eventually sell this platform or host it in the cloud, but your immediate need is to deploy it on-premise at a particular school or for a particular school district, your approach might be different.

For example, in the cloud you may have a completely different authentication scheme than in the on-premise solution, or your frontend may have very different needs in each case.
You may also want to eventually integrate this platform into other larger systems, or sell it as a <abbr title="Business to Business">B2B</abbr> service.
The list of possible problems that cause churn in your choice of how to architect session management is nearly endless.
But even if you have none of the types of problems described above or any others related to it, the challenge of managing session state across a stateless architecture is definitely a daunting one.

Your first thought may be to simply find a way to propagate the session among all the disparate microservices you'll be building.
This is certainly possible, and while there are some possible ways it can be accomplished, none of them are simple, and even the artistic ones aren't really elegant.

For example, perhaps you'll implement an <abbr title="Single Sign-on">SSO</abbr> using OAuth with <abbr title="JSON Web Token">JWT</abbr>s.
You'll need an authorization server to create the tokens.
Then, if you can somehow manage to have the JWTs sent to each microservice along with the request, you still have to manage shared secret keys to decode the payload, not to mention the possible size of the payload in JSON format.

Or perhaps you'll implement clustered sessions, and propagate a JSESSIONID cookie or header instead.
Each microservice will just act as if it owns the session, or at least a read-only copy of it, and load it into memory on each request, perhaps from a database or a key-value data store.
This could work, though I'd love to know if anyone has a link to a good tutorial on this exact topic.
Perhaps I'll work on one, just to fully analyze the benefits and drawbacks of this approach.

At any rate, my opinion is that all of these options kind of suck.
Let's build a better solution.

## The Journey

Let's list the goals of our solution:

1. Make session management completely transparent to the API consumer. Keep the API documentation tool (such as Swagger) in mind - if it's easy to use a "Try it out" button, it'll be easy for the real API consumer.
2. Make the on-boarding process for new microservices in the architecture stupid simple. We want as little setup as possible. Ideally, none, at least on the microservice deployment side. We may need a bit of setup on the architectural side.
3. All session state we consider essential for any microservice to do its job should be available anywhere in our architecture.

With these goals in mind, the first decision point is hopefully one you've already made in your journey toward microservices.
The question is: How do we tie all these separate deployment units together, without adding complexity on the client-side? Of course, the answer is the API Gateway pattern.

You may be tempted to say "Let's just delegate that job to some third-party system, vendor, or platform solution, such as the <abbr title="Pivotal Cloud Foundry">PCF</abbr> go-router."
While those may have a necessary place in your infrastructure, they are not going to help us here.
We need our own gateway.

## Add the Bits

Going back to our first project, let's add Zuul from the Spring Cloud Netflix suite to our stack. We'll need this to do reverse-proxy functions.
If you have another routing solution in your architecture, use it.
But make sure it has the additional capabilities we'll cover shortly.

First, add the following to your `pom.xml` to import the Spring Cloud Maven <abbr title="Bill of Materials">BOM</abbr> to manage Spring Cloud dependencies:

```xml
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>Greenwich.SR1</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
```

Note: Choose the latest version of Spring Cloud, as this article may be out of date.

Then add the following dependency:

```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
        </dependency>
```

Add a quick configuration, or add an annotation to your entrypoint application.

```java
import org.springframework.cloud.netflix.zuul.EnableZuulProxy;
import org.springframework.context.annotation.Configuration;

@Configuration
@EnableZuulProxy
public class ZuulConfiguration {
}
```

Lastly, add some configuration to enable routing within Zuul:

```yaml
app:
  base-url: http://localhost:8081

zuul:
  ignored-patterns:
    - /api/v1/login
    - /api/v1/logout
  routes:
    my-service:
      strip-prefix: false
      path: /api/v1/**
      url: ${app.base-url}
```

Note: The `app.base-url` property is just a stand-in for whatever downstream routing you need to do, which is outside the scope of this article. The options are numerous, but will be very tailored to your particular use case.

In our example, we're assuming the microservice we built with an `/api/v1/hello` endpoint is hosted on port 8081. Here's the relevant part of the configuration for the second project:

```yaml
server:
  port: 8081
```

## Test It Out

Now, when we log in, we can route traffic to a downstream microservice through our gateway.
Using Postman ([here](https://github.io/InSourceSoftware/spring-ws-examples/Spring WebService Examples.postman_collection.json)), hit the Log In route with a `POST` at http://localhost:8080/api/v1/login, then issue a `GET` to http://localhost:8080/api/v1/hello.
We're using cookies with a `JSESSIONID` so make sure both requests take them into account.
All the session state is managed by the gateway (currently in memory, but we can easily use Spring Session to do better).

## But You Said...?

So, how do we propagate sessions? Basically all we've done so far is implement some authentication scheme, and a simple API gateway. Bear with me...

If we want to easily share state with these downstream microservices in a transparent way, without impacting our API contract, and without requiring any setup on the part of the downstream system, we're left with only one choice.
Let's just use the protocol we've already committed to. HTTP.

Let's use HTTP headers!

To do that, let's add a Zuul filter to our gateway that is session-aware.

```java
import com.netflix.zuul.ZuulFilter;
import com.netflix.zuul.context.RequestContext;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.core.userdetails.User;
import org.springframework.stereotype.Component;

import static org.springframework.cloud.netflix.zuul.filters.support.FilterConstants.PRE_DECORATION_FILTER_ORDER;
import static org.springframework.cloud.netflix.zuul.filters.support.FilterConstants.PRE_TYPE;

@Component
public class ZuulSessionFilter extends ZuulFilter {
    @Override
    public String filterType() {
        return PRE_TYPE;
    }

    @Override
    public int filterOrder() {
        return PRE_DECORATION_FILTER_ORDER + 1;
    }

    @Override
    public boolean shouldFilter() {
        return true;
    }

    @Override
    public Object run() {
        doFilter();

        return null;
    }

    private void doFilter() {
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        if (authentication == null) {
            // Should never happen, but could be null (see javadoc for SecurityContext#getAuthentication)
            return;
        }

        Object principal = authentication.getPrincipal();

        // Add header if user is not anonymous
        if (principal instanceof User) {
            User user = (User) principal;

            RequestContext requestContext = RequestContext.getCurrentContext();
            requestContext.addZuulRequestHeader("X-USERNAME", user.getUsername());
        }
    }
}
```

Again, if you're using another gateway or reverse-proxy technology, it needs to have the capability to plug code into it that is aware of our session. Using Zuul, this is incredibly easy. We contain all the "complexity" (all 50 lines of it) in once place.

In this example, we're not being very creative.
We'll simply be propagating the username.
We could (if we're ambitious) propagate the `JSESSIONID` itself.
Or we could add other session attributes that we've previously stored with our session.
The choice is up to you.

The key consideration here is to keep it light.
Only information that is considered "context" or "identity" should be passed in this way.
That allows each downstream microservice to have enough information to do its job, but no more.
If it needs to look up the user, and that has a cost, it should be solved with additional architecture (try caching!).
But passing 27KB of JSON data is not a good way to go here, nor is storing that in your session in the first place.

Ok, Ok, I know. Let's add the final touch and look at consuming that state down stream.

## The Really Tricky Bits...

Here it is. What we've all been waiting for.

```java
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import javax.servlet.http.HttpServletRequest;

@RestController
@RequestMapping("/api/v1")
public class HelloController {
    @GetMapping("/hello")
    public String sayHello(HttpServletRequest request) {
        String username = request.getHeader("X-USERNAME");

        return String.format("Hello, %s!", username);
    }

    @GetMapping("/anonymous")
    public String sayNothing() {
        return "I have nothing to say to you. Who are you?";
    }
}
```

Let's ask ourselves, have we accomplished our goals?

1. Make session management completely transparent to the API consumer?
    
    Yep. We didn't impact our API contract at all. As long as the API consumer is being routed through our gateway, they have no idea that username is a required parameter for `/hello`.
    
2. Make the on-boarding process for new microservices in the architecture stupid simple?
    
    Yep. There is literally nothing special about our microservice. It's just a regular Spring Boot app. No setup. No configuration. Nothing. Note: We did add a bit of routing to the gateway, but that's required regardless, isn't it?
    
3. All session state we consider essential for any microservice to do its job should be available anywhere in our architecture?
    
    Yep. We can add any as many headers from our session as needed. We just need to keep the request size in mind. We add a tiny bit of extra information to each proxied request, and gain a huge architectural advantage.

## Conclusion

In this article, we've learned how to build a stateful API gateway with routing using Zuul and secure it with any authentication scheme we want.
We also learned how to add a Zuul request filter that adds session state to our proxied requests as HTTP headers.
Finally, we learned how to set up microservices that are session-aware&mdash;in fact, we already knew how, because there's no special setup required!
