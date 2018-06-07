---
layout: post
subheadline: Spring Boot Series
title: Custom Authentication with Spring Boot
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
image:
    title: posts/spring-boot.png
    thumb: posts/spring-boot-t.png
    homepage: posts/spring-boot.png
---

In the [previous article](/blog/articles/stateless-api-security-with-spring-boot-part-2.html), we discussed adding an `Authorization` header and a custom security scheme to a Spring Boot application for stateless API security. In this article, we'll discuss how to enable Restful username/password authentication.

## Rest Authentication

In Spring Security, it's been fairly effortless to enable username/password authentication through Form Login, which is a vestige of a bygone era of simple login screens and stateful servers before single page applications were prevalent (or even existed). Admittedly, once an HTTP POST with URL encoded form data is no longer viable, it's likely that username/password authentication is not viable either. Most likely, we'll want a multi-factor authentication flow. We'll discuss this in a future post.

For now, let's look at how to bypass the traditional form login, but use the same concepts with a JSON-based API.

### Set Up

Let's define a build for our project. Here's a pom.xml skeleton to get us started:

```xml
<?xml version="1.0" encoding="utf-8" ?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>io.insource</groupId>
    <artifactId>customauth-security-example</artifactId>
    <version>0.1.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>customauth-security-example</name>
    <description>Example project for securing REST endpoints with custom authentication.</description>

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

### Custom Auth Filter

Instead of using the traditional `formLogin()` configurer, let's author our own simple filter. In fact, we can extend the existing form login filter, called `UsernamePasswordAuthenticationFilter`, and provide a tiny bit of code to get access to a request body.

Create the following class:

```java
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

import javax.servlet.FilterChain;
import javax.servlet.ServletException;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.util.HashMap;

public class CustomAuthenticationFilter extends UsernamePasswordAuthenticationFilter {
	private static final String BODY_ATTRIBUTE = CustomAuthenticationFilter.class.getSimpleName() + ".body";

	private final ObjectMapper objectMapper;

	public CustomAuthenticationFilter(ObjectMapper objectMapper) {
		this.objectMapper = objectMapper;
	}

	@Override
	public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) throws IOException, ServletException {
		HttpServletRequest request = (HttpServletRequest) req;
		HttpServletResponse response = (HttpServletResponse) res;

		// Parse the request body as a HashMap and populate a request attribute
		if (requiresAuthentication(request, response)) {
			UsernamePasswordRequest usernamePasswordRequest = objectMapper.readValue(request.getInputStream(), UsernamePasswordRequest.class);
			request.setAttribute(BODY_ATTRIBUTE, usernamePasswordRequest);
		}

		super.doFilter(req, res, chain);
	}

	protected String obtainUsername(HttpServletRequest request) {
		UsernamePasswordRequest usernamePasswordRequest = (UsernamePasswordRequest) request.getAttribute(BODY_ATTRIBUTE);
		return usernamePasswordRequest.get(getUsernameParameter());
	}

	protected String obtainPassword(HttpServletRequest request) {
		UsernamePasswordRequest usernamePasswordRequest = (UsernamePasswordRequest) request.getAttribute(BODY_ATTRIBUTE);
		return usernamePasswordRequest.get(getPasswordParameter());
	}

	private static class UsernamePasswordRequest extends HashMap<String, String> {
		// Nothing, just a type marker
	}
}
```

This class makes use of everything provided by `UsernamePasswordAuthenticationFilter` which in turn extends `AbstractAuthenticationProcessingFilter`. It will terminate processing of the request if it finds a request that matches, so no `@RestController` will be invoked (just as with Form Login).

In addition to the `UsernamePasswordAuthenticationToken` and other window dressing that is created by the parent class, we take over processing the request body. Using a simple `ObjectMapper`, we can convert an arbitrary key/value JSON structure into a `HashMap`.

Once the body is parsed, we can easily obtain an arbitrarily named username and password, just as with Form Login. We use a bit of request attribute trickery just to satisfy the method calls made by the parent class.

### Customize Authentication

Once we have a basic custom filter in place to do authentication (note we didn't have to code that part), let's turn our attention to configuring Spring Security.

```java
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.config.annotation.authentication.builders.AuthenticationManagerBuilder;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.web.authentication.AuthenticationSuccessHandler;
import org.springframework.security.web.authentication.SimpleUrlAuthenticationSuccessHandler;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;
import org.springframework.security.web.authentication.preauth.RequestHeaderAuthenticationFilter;
import org.springframework.security.web.authentication.session.ChangeSessionIdAuthenticationStrategy;
import org.springframework.security.web.authentication.session.CompositeSessionAuthenticationStrategy;
import org.springframework.security.web.authentication.session.SessionAuthenticationStrategy;
import org.springframework.security.web.csrf.CsrfAuthenticationStrategy;
import org.springframework.security.web.csrf.CsrfTokenRepository;
import org.springframework.security.web.csrf.HttpSessionCsrfTokenRepository;
import org.springframework.security.web.csrf.LazyCsrfTokenRepository;
import org.springframework.security.web.util.matcher.AntPathRequestMatcher;

import java.util.Arrays;

@Configuration
@EnableWebSecurity
public class WebSecurityConfiguration extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.antMatcher("/**")
            .addFilterAfter(customAuthFilter(), RequestHeaderAuthenticationFilter.class)
            .authorizeRequests()
                .antMatchers("/", "/login").permitAll()
                .anyRequest().authenticated()
            .and()
                .csrf().csrfTokenRepository(csrfTokenRepository())
            .and()
                .formLogin().disable();
    }

    @Bean
    public UsernamePasswordAuthenticationFilter customAuthFilter() throws Exception {
        UsernamePasswordAuthenticationFilter authenticationFilter = new CustomAuthenticationFilter(objectMapper());
        authenticationFilter.setRequiresAuthenticationRequestMatcher(new AntPathRequestMatcher("/login", "POST"));
        authenticationFilter.setUsernameParameter("username");
        authenticationFilter.setPasswordParameter("password");
        authenticationFilter.setAuthenticationManager(authenticationManagerBean());
        authenticationFilter.setAuthenticationSuccessHandler(authenticationSuccessHandler());
        authenticationFilter.setSessionAuthenticationStrategy(sessionAuthenticationStrategy());

        return authenticationFilter;
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.inMemoryAuthentication()
            .withUser("user").password("password").roles("USER").and()
            .withUser("admin").password("admin").roles("ADMIN");
    }

    @Bean
    @Override
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }

    @Bean
    public AuthenticationSuccessHandler authenticationSuccessHandler() {
        return new SimpleUrlAuthenticationSuccessHandler("/");
    }

    @Bean
    public SessionAuthenticationStrategy sessionAuthenticationStrategy() {
        return new CompositeSessionAuthenticationStrategy(Arrays.asList(
            new ChangeSessionIdAuthenticationStrategy(),
            new CsrfAuthenticationStrategy(csrfTokenRepository())
        ));
    }

    @Bean
    public CsrfTokenRepository csrfTokenRepository() {
        return new LazyCsrfTokenRepository(new HttpSessionCsrfTokenRepository());
    }

    @Bean
    public ObjectMapper objectMapper() {
        return new ObjectMapper();
    }
}
```

There's a few things going on here, so let's break it down.

#### Configuration

First, we wire in our custom extension of `UsernamePasswordAuthenticationFilter`. It's best to define an order for the filter to fit into the filter chain. In this case, it doesn't clash with anything in the defaults, so we could skip this step, but in case we add pre-auth (see previous tutorials), the `addFilterAfter()` ensures it will be *after* that filter if present. In this case, it fires pretty early in the chain.

Next, we manually open up the `/` and `/login` routes and lock down everything else.

We also need to make sure our CSRF protection is consistent between the default filter chain and our custom filter, so we need to define the glue piece manually, which is the `HttpSessionCsrfTokenRepository`. We use this later as well.

Then we disable the default form login, which would put another `UsernamePasswordAuthenticationFilter` into the filter chain and we definitely don't want that.

#### UsernamePasswordAuthenticationFilter

In order to configure our filter, we need several additional things.

First, we define an `ObjectMapper` to use with our custom JSON parsing inside the filter.

Then, we define the request matcher. We don't have helper methods for this custom filter but it's not hard to do it manually with an `AntPathRequestMatcher`. This makes it identical to the default form login configuration, but with JSON instead of form fields.

Also similar to the defaults, we set up the username and password fields that will hold our principal and credentials. I've explicitly set them to call out where to configure them for your needs.

We also define a `SessionAuthenticationStrategy`, since we don't get any defaults for free. I haven't ensured this is perfectly consistent with the defaults, so comments are welcome, but in this example, we're also adding session-fixation and CSRF protection to the filter chain with a `CompositeSessionAuthenticationStrategy`. The `CsrfAuthenticationStrategy` uses the same `CsrfTokenRepository` we defined above, which also gets used by our own custom controller (shown below) to expose the CSRF token.

Lastly, we define a simple `AuthenticationManager` and `AuthenticationSuccessHandler`. The important thing about the `AuthenticationManager` is we need to expose it as a bean so we can add it to our custom filter.

<p class="alert alert-info">
<strong>Note:</strong> This is also useful if we need to access it from somewhere within our application, as the default security configurer does not expose any of these objects as beans. You may need that, for example, if you want to build a password management screen where you need to re-test the user's credentials prior to changing them.
</p>

## Expose the CSRF

We need to add one piece that's missing from the form generated by the `DefaultLoginPageGeneratingFilter`. I've not seen any tutorials for how to do this, though that doesn't mean they don't exist.

Let's add a `@RestController` to our application:

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContext;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.web.csrf.CsrfTokenRepository;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpSession;
import java.util.Optional;

@RestController
public class AuthController {
    private final CsrfTokenRepository csrfTokenRepository;

    @Autowired
    public AuthController(CsrfTokenRepository csrfTokenRepository) {
        this.csrfTokenRepository = csrfTokenRepository;
    }

    @GetMapping("/")
    public StatusResponse getStatus() {
        String username = Optional.of(SecurityContextHolder.getContext())
            .map(SecurityContext::getAuthentication)
            .map(Authentication::getPrincipal)
            .map(Object::toString)
            .orElse("anonymous");

        String statusText = "anonymous".equals(username)
            ? "You have been securely logged out."
            : "You have successfully logged in.";

        return StatusResponse.createSuccess(statusText);
    }

    @GetMapping("/login")
    public SessionResponse getSession(HttpServletRequest request, HttpSession httpSession) {
        String sessionId = httpSession.getId();
        String csrfToken = csrfTokenRepository.loadToken(request).getToken();

        return new SessionResponse(sessionId, csrfToken);
    }

    public static class StatusResponse {
        private final String status;
        private final String statusText;

        StatusResponse(String status, String statusText) {
            this.status = status;
            this.statusText = statusText;
        }

        public String getStatus() {
            return status;
        }

        public String getStatusText() {
            return statusText;
        }

        static StatusResponse createSuccess(String statusText) {
            return new StatusResponse("Success", statusText);
        }
    }

    public static class SessionResponse {
        private final String sessionToken;
        private final String csrfToken;

        SessionResponse(String sessionToken, String csrfToken) {
            this.sessionToken = sessionToken;
            this.csrfToken = csrfToken;
        }

        public String getSessionToken() {
            return sessionToken;
        }

        public String getCsrfToken() {
            return csrfToken;
        }
    }
}
```

The `GET /` route is optional, but creates a simple landing page that tells you whether you're logged in or out. Since we opened up that route in Spring Security, it works whether or not you're logged in.

The `GET /login` route replaces the `_csrf` hidden attribute from the Form Login page by utilizing the aforementioned `CsrfTokenRepository`. API consumers will need to obtain the CSRF prior to invoking the `/login` route, as the entire application has CSRF protection enabled. Invoking it produces the following output:

```json
{
    "sessionToken": "E2676FC43DF7366FFC8F7B23FC689187",
    "csrfToken": "550fdde5-3786-4da1-b1f8-27d3fb08969f"
}
```

Here is a sample CURL request for using the CSRF token:

```jshelllanguage
curl -X POST \
  http://localhost:8080/login \
  -H 'Cache-Control: no-cache' \
  -H 'Content-Type: application/json' \
  -H 'X-CSRF-TOKEN: 550fdde5-3786-4da1-b1f8-27d3fb08969f' \
  -d '{
	"username": "user",
	"password": "password"
  }'
```

`X-CSRF-TOKEN` is the default name of the header required by the `CsrfFilter` that was enabled with `csrf()` in our `WebSecurityConfigurerAdapter`.

## Conclusion

In this article, we've learned how to create a custom username/password authentication filter, and manually configure Spring Security to use it. We also learned how to expose the CSRF token through our REST API with consistent CSRF protection throughout the application.
