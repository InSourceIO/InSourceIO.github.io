---
layout: post
subheadline: Google Cloud Series
title: Google Cloud Platform, Part 1
teaser: Deploy secure Spring Boot apps with Google Cloud SQL to App Engine Standard.
categories:
  - articles
tags:
  - google
  - mobile app
  - development
related:
  - /posts/latest-projects-and-other-news.html
image:
    title: stock/google-phone-1920.jpg
    thumb: stock/google-phone-1920-t.jpg
    homepage: finger/google-phone-1920.jpg
---

## ~&gt; ./start --debug --with-args -Xenv:dev

**The Hook:** I want to deploy a secure Spring Boot app with a Google Cloud SQL connection to the App Engine Standard environment.

**The Line:** Know what the @%$&!# you are doing.

**The Sinker:** Just use [spring-cloud-gcp-starter-sql-mysql][1].

## ~&gt; cat output.dump | grep INFO | less

This was difficult for several reasons. Here's the short list:

1. Local development does not (quite) work like a GCP deployed app
2. App Engine Standard and App Engine Flexible do not work the same way
3.  App Engine Standard is poorly documented (for certain use cases, like mine)
4.  App Engine Standard requires a WAR file deploy and runs on Jetty, not Tomcat

I assumed App Engine Flexible environment was the way to go, as it's newer, the docs are better, the examples are more relevant, and the setup seemed easier. It also uses Docker, so I figured this seems like the easy way to get to that utopia state of the same app deployed everywhere.

What I failed to realize is that Flexible environment is ridiculously expensive, has crazy slow deployments, and is not designed for the same use cases. In fact, most of the time, you don't want to use Flexible environment. That being said, it is vastly superior in almost every way. Apps work the Spring Boot way inside the Docker containers that Google builds. Configuration and setup are vastly simplified, and use yaml files instead of XML deployment descriptors (remember that archaic concept???). Most of Google's services just work out of the box. Etc.

But Flexible environment is a sledgehammer for the problem of driving a tiny stud into the sparse wasteland of a roof that is my budget-conscious project. I really need a @$%#!& nail gun. That's what App Engine Standard is. It just isn't my favorite brand. On account of it sucks.

## ~&gt; cat output.dump | grep ERROR | more

Let's cut to the chase. The failure cases I encountered are almost too numerous to list. What I started with was a functional app capable of being deployed to App Engine Flexible, built using the [very helpful tutorial on GCP's documentation site][2] and [Hello World GitHub project][3], among others.

While there were some very specific caveats to even getting this working (which took an embarassingly long time to get working, at least 2-3 evenings), it generally worked well and was only slightly ridiculous. Here's the setup for this piece to work:


`src/main/resources/application.yml`

```yaml
spring:
  application:
    name: hello-world
  jackson:
    serialization:
      write_dates_as_timestamps: false
  jpa:
    open-in-view: true
    show-sql: true
    hibernate:
      ddl-auto: none

security:
  oauth2:
    sso:
      loginPath: /login
    client:
      clientId: my-client-id
      clientSecret: my-client-secret
      accessTokenUri: https://www.googleapis.com/oauth2/v3/token
      userAuthorizationUri: https://accounts.google.com/o/oauth2/auth
      tokenName: oauth_token
      authenticationScheme: query
      clientAuthenticationScheme: form
      scope: profile
    resource:
      userInfoUri: https://www.googleapis.com/userinfo/v2/me
      preferTokenInfo: false

# Spoiler Alert: The following sits innocuously fooling 
# you into believing things that aren't true...
management:
  contextPath: /_ah
  security:
    enabled: false
```

Here, we've set up some basic security configuration using Google as an OAuth2 provider (there are good tutorials for how to do this on the web). The `management` section _seems_ to disable security for the `/_ah` requests Google makes, and in App Engine Flexible, things are fine.

Next, we have two profiles with separate configs, setting up a database for local development, and a Google Cloud SQL database for production.

`src/main/resources/application-dev.yml`

```yaml
spring:
  datasource:
    driverClassName: com.mysql.jdbc.Driver
    url: jdbc:mysql://localhost:3306/hello-world?useUnicode=true&characterEncoding=utf8&useSSL=false
    username: myuser
    password: mypassword
  jpa:
    database: mysql
```

`src/main/resources/application-prod.yml`

```yaml
spring:
  datasource:
    driverClassName: com.mysql.jdbc.Driver
    url: jdbc:mysql://google/hello-world?useSSL=false&cloudSqlInstance=my-project:us-central1:hello-world&socketFactory=com.google.cloud.sql.mysql.SocketFactory
    username: myuser
    password: mypassword
  jpa:
    database: mysql
```

In order to make this work for App Engine, I used the following configuration for Flexible environment.

`src/main/appengine/app.yaml`

```yaml
runtime: java
env: flex

service: processor

manual_scaling:
  instances: 1

resources:
  memory_gb: 0.768

handlers:
  - url: /.*
    script: this field is required, but ignored

env_variables:
  ENVIRONMENT: "prod"
```

Notice the environment variable. This was necessary to allow a local development setup that was different from the production deployment. A bit of trickery in the Spring Boot init uses this environment variable:

`src/main/java/com/example/java/gettingstarted/HelloworldApplication.java`

```java
@SpringBootApplication
public class HelloworldApplication {
  public static void main(String[] args) {
    SpringApplication springApplication = new SpringApplication(HelloworldApplication::class);

    String env = System.getenv("ENVIRONMENT");
    if (env == null) {
      env = "dev";
    }

    springApplication.setAdditionalProfiles(env);
    springApplication.run(args);
  }
}
```

Assuming the appengine plugin is applied to your build, you just need an extra dependency to add the socket factory for Google Cloud SQL to the classpath at runtime. I added the following to my Gradle build:

```groovy
dependencies {
    ...
    runtime 'com.google.cloud.sql:mysql-socket-factory'
}
```

While I won't delve into the details of why this special piece is needed for Cloud SQL, suffice it to say this handles connecting correctly in a Google Cloud environment.

With this basic app in place (and whatever code you want to do databas-ey things, which I trust you can figure out on your own), you're good to go in App Engine Flexible. The app is secure and can connect to a cloud database, and `/_ah` requests made by Google to monitor health work fine. So it would seem.

## ~&gt; cat output.dump | grep SEVERE > /dev/null

However, when you deploy this totally functional app to App Engine Standard (with a few changes, see below), it fails spectacularly. Here's the changes I thought I needed.

`src/main/webapp/WEB-INF/appengine-web.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<appengine-web-app xmlns="http://appengine.google.com/ns/1.0">
    <service>hello-world</service>
    <threadsafe>true</threadsafe>
    <runtime>java8</runtime>
    <sessions-enabled>true</sessions-enabled>
    <public-root>/static</public-root>
    <env-variables>
        <env-var name="ENVIRONMENT" value="prod" />
    </env-variables>
    <system-properties>
        <property name="java.util.logging.config.file" value="WEB-INF/classes/logging.properties"/>
    </system-properties>
</appengine-web-app>
```

The above replaces the `app.yaml` that Flexible required. You can add additional scaling parameters if you like, see [appengine-web.xml Reference][4].

Then, depending on your build tool (I use Gradle), you may need to make changes to deploy a WAR file as well. See [this totally unhelpful example][5]. I shouldn't say totally unhelpful. It definitely works. But it does not do anything Spring Boot-ish whatsoever, other than stare at you and wave... Creepy.

The problem is that deploying to Standard using the config that worked on Flexible, I can't connect to a Cloud SQL datasource. My app fails to deploy and start up successfully. Something is missing. And attempting to figure it out via Google was not working.

Somehow, at the end of it all (after nearly a month of procrastinating because this problem was so frustrating to work out), I found the [spring-cloud-gcp project][6]. After looking at the source code, I think I've worked out the missing piece, but it's still a theory. However, the simple fix was to add a starter from this project to my classpath, which effectively adds `spring-cloud-gcp-autoconfigure.jar` containing an implementation or two of `CloudSqlJdbcInfoProvider`, whatever that is.

I added the following to my Gradle build:

```groovy
repositories {
    ...
    maven {
        url "https://repo.spring.io/milestone"
    }
}

dependencyManagement {
    imports {
        mavenBom 'org.springframework.cloud:spring-cloud-gcp-dependencies:1.0.0.M2'
    }
}

dependencies {
    ...
    providedRuntime 'org.springframework.boot:spring-boot-starter-tomcat'
    compileOnly 'org.slf4j:jul-to-slf4j'
    compileOnly 'javax.servlet:javax.servlet-api'
    runtime 'org.springframework.cloud:spring-cloud-gcp-starter-sql-mysql'
}
```

The first three dependencies are just changes going from Flexible to Standard, which runs in a Jetty container instead of embedded Tomcat. More on this next time. The last piece adds the aforementioned starter for mysql to the classpath at runtime. We do need a few config changes. Here's the prod config:

`src/main/resources/application-prod.yml`

```yaml
spring:
  datasource:
#    driverClassName: com.mysql.jdbc.Driver
#    url: jdbc:mysql://google/hello-world?useSSL=false&cloudSqlInstance=my-project:us-central1:hello-world&socketFactory=com.google.cloud.sql.mysql.SocketFactory
    username: myuser
    password: mypassword
  jpa:
    database: mysql
  cloud:
    gcp:
      project-id: my-project
      sql:
        database-type: mysql
        database-name: hello-world
        instance-connection-name: my-project:us-central1:hello-world
```

The theory is that the `CloudSqlJdbcInfoProvider` actually configures the two commented out settings. What looks to happen is that the `driverClassName` actually gets set to `com.mysql.jdbc.GoogleDriver`, which I don't even know where that comes from. It isn't on the classpath. It's just a mystery. I suspect this is actually the magic, which could make this starter dependency optional at best, though it could have other uses, so your mileage may vary.

Now, there's one last issue with App Engine Standard. Remember the config for Spring Boot from earlier, with the comment about innocuous settings fooling you into believing things that aren't true? Yeah... About that. So the `management` section does not seem to apply in the Standard environment. Possibly because of the war file deployment, additional Spring Boot features aren't available, feedback welcome on what causes it. But suffice it to say, the `/_ah` requests made by Google fail. Why? Because security.

So you'll need to add a couple of unsecured endpoints that can respond to the App Engine Standard health requests. These are `/_ah/start` and `/_ah/stop`.

```java
@RestController
public class HealthController {
  @GetMapping("/_ah/start")
  @PreAuthorize("hasRole('ANONYMOUS')")
  public String start() {
    return "OK";
  }

  @GetMapping("/_ah/stop")
  @PreAuthorize("hasRole('ANONYMOUS')")
  public String stop() {
    return "OK";
  }
}
```

You will want to add these to your unsecured routes as well.

```java
@Configuration
public class WebSecurityConfiguration extends WebSecurityConfigurerAdapter {
  @Override
  public void configure(HttpSecurity http) {
    http
      .authorizeRequests()
      .antMatchers("/_ah/*")
      .permitAll();
  }
}
```

This works in App Engine Standard. FINALLY!!!

Now, there are a few nasty problems left to deal with. I'll save those for next time.

Hint: Try running this pile of mess from your IDE _after_ converting to App Engine Standard. Double Hint: You can't!!!

[1]: https://github.com/spring-cloud/spring-cloud-gcp/tree/master/spring-cloud-gcp-starters/spring-cloud-gcp-starter-sql-mysql
[2]: https://cloud.google.com/community/tutorials/run-spring-petclinic-on-app-engine-cloudsql
[3]: https://github.com/GoogleCloudPlatform/getting-started-java/tree/master/helloworld-springboot
[4]: https://cloud.google.com/appengine/docs/standard/java/config/appref
[5]: https://github.com/GoogleCloudPlatform/getting-started-java/tree/master/appengine-standard-java8/springboot-appengine-standard
[6]: https://github.com/spring-cloud/spring-cloud-gcp
