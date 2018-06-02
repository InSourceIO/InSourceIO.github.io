---
layout: post
subheadline: Spring Boot Series
title: Stateless API Security with Spring Boot, Part 2
teaser: Example project for securing REST endpoints with an Authorization header for API security.
categories:
  - articles
tags:
  - spring boot
  - security
  - development
related:
  - /blog/articles/stateless-api-security-with-spring-boot-part-1.html
image:
    title: posts/spring-boot.png
    thumb: posts/spring-boot-t.png
    homepage: posts/spring-boot.png
---

In the [previous article](/blog/articles/stateless-api-security-with-spring-boot-part-1.html), we discussed adding Basic authentication to our project and turned off session management for a pure stateless API. In this article, we'll discuss how to extend that using an `Authorization` header and a custom security scheme.

## Custom Authentication

Let's now turn our attention to how to evolve beyond the Basic authentication scheme, which has limited uses for us in practice (though it can be a very suitable model for simple app-to-app security). Let's imagine that we want to provision API keys for clients to consume our API. With the previous concepts in place, we have the ability to do username/password authentication. But we need to move to a new scheme that only uses tokens as credentials.

### Add Pre-Authentication

Instead of basic auth, let's define a customized filter to enable pre-authentication. You may have seen tutorials on the web using the `SM_USER` header to do this. This is Siteminder security, which assumes an API Gateway is forcing users to authenticate prior to hitting our application, and providing a header with the pre-authenticated username.

Let's use this concept with the `Authorization` header. Replace the Java config we defined for Basic auth with the following:

```java
import org.springframework.boot.autoconfigure.security.Http401AuthenticationEntryPoint;
import org.springframework.cache.CacheManager;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.cache.concurrent.ConcurrentMapCache;
import org.springframework.cache.support.SimpleCacheManager;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.annotation.Order;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.AuthenticationProvider;
import org.springframework.security.authentication.ProviderManager;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.core.userdetails.AuthenticationUserDetailsService;
import org.springframework.security.web.AuthenticationEntryPoint;
import org.springframework.security.web.authentication.preauth.PreAuthenticatedAuthenticationProvider;
import org.springframework.security.web.authentication.preauth.PreAuthenticatedAuthenticationToken;
import org.springframework.security.web.authentication.preauth.RequestHeaderAuthenticationFilter;

import java.util.Collections;
import java.util.regex.Pattern;

@Configuration
@EnableWebSecurity
public class WebSecurityConfiguration extends WebSecurityConfigurerAdapter {
    public static final String REALM_NAME = "MyRealm";
    public static final String API_KEY_PARAM = "apikey";
    public static final Pattern AUTHORIZATION_HEADER_PATTERN = Pattern.compile(
        String.format("%s %s=\"(\\S+)\"", REALM_NAME, API_KEY_PARAM)
    );

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.antMatcher("/**")
            .addFilterAfter(preAuthenticationFilter(), RequestHeaderAuthenticationFilter.class)
            .authorizeRequests()
                .anyRequest().authenticated()
            .and()
                .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            .and()
                .exceptionHandling().authenticationEntryPoint(authenticationEntryPoint())
            .and()
                .csrf().disable();
    }

    @Bean
    public RequestHeaderAuthenticationFilter preAuthenticationFilter() {
        RequestHeaderAuthenticationFilter preAuthenticationFilter = new RequestHeaderAuthenticationFilter();
        preAuthenticationFilter.setPrincipalRequestHeader("Authorization");
        preAuthenticationFilter.setCredentialsRequestHeader("Authorization");
        preAuthenticationFilter.setAuthenticationManager(authenticationManager());
        preAuthenticationFilter.setExceptionIfHeaderMissing(false);

        return preAuthenticationFilter;
    }

    @Override
    protected AuthenticationManager authenticationManager() {
        return new ProviderManager(Collections.singletonList(authenticationProvider()));
    }

    @Bean
    public AuthenticationProvider authenticationProvider() {
        PreAuthenticatedAuthenticationProvider authenticationProvider = new PreAuthenticatedAuthenticationProvider();
        authenticationProvider.setPreAuthenticatedUserDetailsService(userDetailsServiceWrapper());
        authenticationProvider.setThrowExceptionWhenTokenRejected(false);

        return authenticationProvider;
    }

    @Bean
    public AuthenticationUserDetailsService<PreAuthenticatedAuthenticationToken> userDetailsServiceWrapper() {
        return new AuthorizationUserDetailsService();
    }

    @Bean
    public AuthenticationEntryPoint authenticationEntryPoint() {
        return new Http401AuthenticationEntryPoint(REALM_NAME);
    }
}
```

This is a larger configuration. Let's break it down.

#### Constants

We've defined some constants at the top of the file. We could use a similar `@ConfigurationProperties` technique as before. But by the time we're defining our own security model, it's probably overkill to make it configurable. Using the RFC 2617 format, we've defined a custom scheme. It looks like this:

```
Authorization: MyRealm apikey="..."
```

#### PreAuthenticationFilter

We've defined a filter using a stock Spring class, `RequestHeaderAuthenticationFilter`. As with the Siteminder example, we can take any header we like and use it to perform "pre" authentication.

What this does is assume that the authentication was performed elsewhere, and our job is to validate or simply use the header to create an authenticated session. In this example, we've turned off session management (we'll see why that's important in a minute), so we're actually just creating a `SecurityContext` on each API call. That sounds expensive. We'll fix it later.

With this filter, we have to define a bunch of pieces to make it work.

1\. We've provided an instance of `RequestHeaderAuthenticationFilter`.

We've configured it with a principal (required) and credentials (optional) and told it not to puke on missing header (so our authentication entrypoint is used instead).

2\. We've provided an `AuthenticationManager` with a single `AuthenticationProvider`.

Our provider of type `PreAuthenticatedAuthenticationProvider` expects an `AuthenticationUserDetailsService` parameterized with `PreAuthenticatedAuthenticationToken`.

3\. We've provided a `AuthenticationUserDetailsService` to do the creation of a `UserDetails`.

This is where our custom authorization logic will go, based on the `PreAuthenticatedAuthenticationToken` given by our provider.

#### AuthenticationEntryPoint

Our entry point simply throws a `401 Unauthorized` with a `WWW-Authenticate` response header containing our custom auth scheme when things go wrong or are missing.

#### UserDetailsService

There's some magic in the `AuthenticationUserDetailsService`, so let's explore that. Here's the definition:

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;

import java.util.Collections;
import java.util.regex.Matcher;

/* Not marked @Component to simplify WebSecurityConfiguration. */
public class AuthorizationUserDetailsService implements AuthenticationUserDetailsService<PreAuthenticatedAuthenticationToken> {
    private static final Logger logger = LoggerFactory.getLogger(AuthorizationUserDetailsService.class);

    @Override
    public UserDetails loadUserDetails(PreAuthenticatedAuthenticationToken token) throws UsernameNotFoundException {
        String authorizationHeader = token.getCredentials().toString();
        logger.info("Loading user for Authorization header: " + authorizationHeader);

        // Check authentication scheme
        if (!authorizationHeader.startsWith(WebSecurityConfiguration.REALM_NAME)) {
            throw new AuthorizationHeaderException("Invalid authentication scheme found in Authorization header");
        }

        // Check for apikey parameter
        if (!authorizationHeader.contains(WebSecurityConfiguration.API_KEY_PARAM)) {
            throw new AuthorizationHeaderException("Unable to locate apikey parameter in Authorization header");
        }

        // Check that the Authorization header matches the pattern
        Matcher matcher = WebSecurityConfiguration.AUTHORIZATION_HEADER_PATTERN.matcher(authorizationHeader);
        if (!matcher.matches()) {
            throw new AuthorizationHeaderException("Unable to parse apikey from Authorization header");
        }

        return loadUserDetails(matcher.group());
    }

    private UserDetails loadUserDetails(String apiKey) {
        // TODO: Implement with your own logic to resolve API key (from Authentication header value)
        return new User("user", apiKey, Collections.singletonList(new SimpleGrantedAuthority("ROLE_USER")));
    }
}
```

This is the easiest place I can think of to place our custom auth scheme logic. The `AuthenticationUserDetailsService` is usually used to convert a principal/credentials pair into a `UserDetails`. But we can easily use it to do some validation first. The `Authorization` header value is passed in the `PreAuthenticatedAuthenticationToken`, so we can use that to convert to a user however we wish to.

First, we validate that the header value is in the right format. Marginally expensive, but negligible in the scheme of things.

Then we parse the `apiKey` from the header. Once we have this, we're in the driver's seat as to how we implement the lookup/conversion. You can look up the user in an API key provisioning database or load them from a JWT. You can also place roles or other information in this `User` object, or define your own custom `UserDetails` implementation.

What's missing from this example is what happens if the `apiKey` is invalid. Just throw an exception similar to the above validation logic. By the way, here's that exception:

```java
import org.springframework.http.HttpStatus;
import org.springframework.security.core.AuthenticationException;
import org.springframework.web.bind.annotation.ResponseStatus;

@ResponseStatus(HttpStatus.UNAUTHORIZED)
public class AuthorizationHeaderException extends AuthenticationException {
    public AuthorizationHeaderException(String msg) {
        super(msg);
    }

    public AuthorizationHeaderException(String msg, Throwable t) {
        super(msg, t);
    }
}
```

You can also use the `UsernameNotFoundException` from the method signature if you prefer.

### Stateless???

The last problem with this scheme is that we're attempting to be stateless. Without sessions, we're kind of hosed from a performance perspective. Now, if you wish to enable sessions for your use case, you're welcome to. And there are many good reasons to do it. But in this case, we're developing a pure API that is consumed by another process or API consumer, not a web browser.

All "stateless" means in this context is that there shouldn't be any state stored in our JVM. Of course, for example purposes we're going to do that, but in production, you'll want to bind in sidecar services to handle it, such as Redis or MongoDB. Those would also be good choices if you want to re-enable sessions.

But in this case, we're going to practice treating stateless (part of any good 12 factor app) the right way, and make it totally transparent. Simply add this bean definition to the Java config above:

```java
import io.insource.spring.ws.examples.security.service.AuthorizationUserDetailsService;

import org.springframework.cache.CacheManager;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.cache.concurrent.ConcurrentMapCache;
import org.springframework.cache.support.SimpleCacheManager;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;

import java.util.Collections;

@Configuration
@EnableWebSecurity
@EnableCaching
public class WebSecurityConfiguration extends WebSecurityConfigurerAdapter {
    ...

    @Bean
    public CacheManager cacheManager() {
        SimpleCacheManager cacheManager = new SimpleCacheManager();
        cacheManager.setCaches(Collections.singletonList(new ConcurrentMapCache("users")));

        return cacheManager;
    }
}
```

With that in place (replace with your chosen production sidecar for caching, outside the scope of this article), we can start caching stuff. Very similar to adding session management, but handled down in our own `UserDetailsService`:

```java
import org.springframework.cache.annotation.Cacheable;
import org.springframework.security.core.userdetails.AuthenticationUserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.security.web.authentication.preauth.PreAuthenticatedAuthenticationToken;

public class AuthorizationUserDetailsService implements AuthenticationUserDetailsService<PreAuthenticatedAuthenticationToken> {
    @Override
    @Cacheable(value = "users", key = "#token")
    public UserDetails loadUserDetails(PreAuthenticatedAuthenticationToken token) throws UsernameNotFoundException {
        ...
    }
}
```

What we have here is a stateless application that uses no session management (no Cookies for you) and yet can cache user details (actually API consumers) as close to the application as we like. Using the caching layer, you can define when and how often your user details expire or are purged, etc.

With caching in place, our API calls to load the `Authentication` into the `SecurityContext` do not exhibit such a terrible performance penalty.

## Conclusion

In this article, we've learned how to implement a pure stateless API with tokens provided in an `Authorization` header, and defined our own custom scheme for accepting those tokens. We've also discussed how this example is very relevant even if you want to enable sessions and consume this API from a mobile app or web browser.