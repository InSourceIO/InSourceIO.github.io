---
layout: post
subheadline: Tech Rant
title: Innocuous Code Lurks Around Every Corner
teaser: "!@#$%^&*"
categories:
  - posts
tags:
  - spring
  - reddit
  - api
related:
image:
    title: stock/fire-and-water-1920.jpg
    thumb: stock/fire-and-water-1920-t.jpg
    homepage: stock/fire-and-water-1920.jpg
---

After about three hours of debugging Spring Framework code to make a simple OAuth2 authenticated call to Reddit, I finally have some semblance of understanding around why my life sucks right now. I am not insane.

**TLDR;**

Perhaps in another post, we'll discuss how to actually connect to Reddit's API via Java (or Kotlin) code and do something useful. But I can at least explain why it doesn't work. Let's dig in.

## The Setup

In order to connect to any OAuth-based server from a Spring project, you need a bit of configuration. Plenty of examples on the web, so not treading new ground here. But for completeness, here's what we're dealing with. The following is the important snippet of my build:

_build.gradle_

```groovy
dependencies {
    compile('org.springframework:spring-web')
    compile('org.springframework.security.oauth:spring-security-oauth2')
}
```

I'm using the `client_credentials` grant type for simplicity, but will eventually switch to `authorization_code`.

<div class="alert alert-info" role="alert">
    There's a pretty decent tutorial on this topic over at <a href="https://www.baeldung.com/spring-security-oauth2-authentication-with-reddit">Baeldung</a>.
</div>

Here's a simple configuration using `@Value` properties from a `@PropertySource` pointing to `reddit.properties` in `src/main/resources`.

_RedditOAuthConfiguration.kt_

```kotlin
package com.reddit

import org.springframework.beans.factory.annotation.Value
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.context.annotation.PropertySource
import org.springframework.security.oauth2.client.OAuth2ClientContext
import org.springframework.security.oauth2.client.OAuth2RestTemplate
import org.springframework.security.oauth2.client.token.grant.client.ClientCredentialsAccessTokenProvider
import org.springframework.security.oauth2.client.token.grant.client.ClientCredentialsResourceDetails
import org.springframework.security.oauth2.config.annotation.web.configuration.EnableOAuth2Client

@Configuration
@EnableOAuth2Client
@PropertySource("classpath:reddit.properties")
class RedditOAuthConfiguration(
  @Value("\${clientId}")
  private val clientId: String,

  @Value("\${clientSecret}")
  private val clientSecret: String,

  @Value("\${accessTokenUri}")
  private val accessTokenUri: String
) {
  @Bean
  fun reddit() = ClientCredentialsResourceDetails().also {
    it.id = "reddit"
    it.clientId = clientId
    it.clientSecret = clientSecret
    it.scope = arrayListOf("identity")
    it.accessTokenUri = accessTokenUri
  }

  @Bean
  fun redditRestTemplate(clientContext: OAuth2ClientContext) = OAuth2RestTemplate(reddit(), clientContext).also {
    it.setAccessTokenProvider(ClientCredentialsAccessTokenProvider())
  }
}
``` 

With that configuration defining a `RestTemplate` and enabling a `Oauth2ClientContextFilter` (via `@EnableOAuth2Client`), we can use it. In my case, I'm stuffing it into a Swagger-generated API client.

_RedditApiConfiguration.kt_

```kotlin
package com.reddit

import com.reddit.api.RedditApi
import org.springframework.beans.factory.annotation.Value
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.context.annotation.PropertySource
import org.springframework.security.oauth2.client.OAuth2RestTemplate

@Configuration
@PropertySource("classpath:reddit.properties")
class RedditApiConfiguration(
  @Value("\${basePath}")
  private val basePath: String
) {
  @Bean
  fun redditApi(oAuth2RestTemplate: OAuth2RestTemplate) =
    RedditApi(ApiClient(oAuth2RestTemplate).also {
      it.basePath = basePath
    })
}
```

Here's an example of `reddit.properties`. The `userAuthorizationUri` is there if you want to switch grant types.

_reddit.properties_

```properties
clientId=...
clientSecret=...
#userAuthorizationUri=https://www.reddit.com/api/v1/authorize
accessTokenUri=https://www.reddit.com/api/v1/access_token
basePath=https://oauth.reddit.com
```

## The Problem

So with this in place, any attempt to use the API client (and by association, the `RestTemplate`) to talk to Reddit's API results in this:

```
org.springframework.security.oauth2.common.exceptions.OAuth2Exception: 429
	at org.springframework.security.oauth2.common.exceptions.OAuth2ExceptionJackson2Deserializer.deserialize(OAuth2ExceptionJackson2Deserializer.java:120) ~[spring-security-oauth2-2.0.6.RELEASE.jar:na]
	at org.springframework.security.oauth2.common.exceptions.OAuth2ExceptionJackson2Deserializer.deserialize(OAuth2ExceptionJackson2Deserializer.java:1) ~[spring-security-oauth2-2.0.6.RELEASE.jar:na]
	at com.fasterxml.jackson.databind.ObjectMapper._readMapAndClose(ObjectMapper.java:4013) ~[jackson-databind-2.9.6.jar:2.9.6]
	at com.fasterxml.jackson.databind.ObjectMapper.readValue(ObjectMapper.java:3084) ~[jackson-databind-2.9.6.jar:2.9.6]
	at org.springframework.http.converter.json.AbstractJackson2HttpMessageConverter.readJavaType(AbstractJackson2HttpMessageConverter.java:237) ~[spring-web-5.0.8.RELEASE.jar:5.0.8.RELEASE]
	at org.springframework.http.converter.json.AbstractJackson2HttpMessageConverter.readInternal(AbstractJackson2HttpMessageConverter.java:217) ~[spring-web-5.0.8.RELEASE.jar:5.0.8.RELEASE]
	at org.springframework.http.converter.AbstractHttpMessageConverter.read(AbstractHttpMessageConverter.java:198) ~[spring-web-5.0.8.RELEASE.jar:5.0.8.RELEASE]
	at org.springframework.security.oauth2.client.token.OAuth2AccessTokenSupport$AccessTokenErrorHandler.handleError(OAuth2AccessTokenSupport.java:235) ~[spring-security-oauth2-2.0.6.RELEASE.jar:na]
	at org.springframework.web.client.ResponseErrorHandler.handleError(ResponseErrorHandler.java:63) ~[spring-web-5.0.8.RELEASE.jar:5.0.8.RELEASE]
	at org.springframework.web.client.RestTemplate.handleResponse(RestTemplate.java:766) ~[spring-web-5.0.8.RELEASE.jar:5.0.8.RELEASE]
	at org.springframework.web.client.RestTemplate.doExecute(RestTemplate.java:724) ~[spring-web-5.0.8.RELEASE.jar:5.0.8.RELEASE]
	at org.springframework.web.client.RestTemplate.execute(RestTemplate.java:690) ~[spring-web-5.0.8.RELEASE.jar:5.0.8.RELEASE]
	at org.springframework.security.oauth2.client.token.OAuth2AccessTokenSupport.retrieveToken(OAuth2AccessTokenSupport.java:137) ~[spring-security-oauth2-2.0.6.RELEASE.jar:na]
	at org.springframework.security.oauth2.client.token.grant.client.ClientCredentialsAccessTokenProvider.obtainAccessToken(ClientCredentialsAccessTokenProvider.java:44) ~[spring-security-oauth2-2.0.6.RELEASE.jar:na]
	at org.springframework.security.oauth2.client.OAuth2RestTemplate.acquireAccessToken(OAuth2RestTemplate.java:221) ~[spring-security-oauth2-2.0.6.RELEASE.jar:na]
	at org.springframework.security.oauth2.client.OAuth2RestTemplate.getAccessToken(OAuth2RestTemplate.java:173) ~[spring-security-oauth2-2.0.6.RELEASE.jar:na]
	at org.springframework.security.oauth2.client.OAuth2RestTemplate.createRequest(OAuth2RestTemplate.java:105) ~[spring-security-oauth2-2.0.6.RELEASE.jar:na]
	at org.springframework.web.client.RestTemplate.doExecute(RestTemplate.java:719) ~[spring-web-5.0.8.RELEASE.jar:5.0.8.RELEASE]
	at org.springframework.security.oauth2.client.OAuth2RestTemplate.doExecute(OAuth2RestTemplate.java:128) ~[spring-security-oauth2-2.0.6.RELEASE.jar:na]
	at org.springframework.web.client.RestTemplate.exchange(RestTemplate.java:668) ~[spring-web-5.0.8.RELEASE.jar:5.0.8.RELEASE]
	at com.reddit.ApiClient.invokeAPI(ApiClient.java:530) ~[classes/:na]
	at com.reddit.api.RedditApi.meUsingGET(RedditApi.java:78) ~[classes/:na]
    ...
```

Not very helpful. Spending lots of time tracing code yields a request reproduced via Postman that looks like this (using cURL):

```
curl -X POST \
  https://www.reddit.com/api/v1/access_token \
  -H 'Accept: application/json, application/x-www-form-urlencoded' \
  -H 'Authorization: Basic XDlvLWJJS0VTd1RzTEE6cm9mWG5Db1V4OHJTaj2W8UtGcx9QSUVmW2Lr' \
  -H 'Cache-Control: no-cache' \
  -H 'Content-Length: 44' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -d 'grant_type=client_credentials&scope=identity'
```

Amazingly, this works perfectly and produces the following:

```json
{
    "access_token": "-6808zn3KtBjkWJ7b7Z-lpmyKp_4",
    "token_type": "bearer",
    "expires_in": 3600,
    "scope": "identity"
}
```

So if I can hit this with Postman just fine, but not Java, what gives? So I traced the code some more, until I could fine every applicable line of code that results in making the actual API call to get the token. Here's the reproduced scenario in a unit test:

_HttpTest.kt_

```kotlin
package io.insource.api.demo

import org.hamcrest.Matchers.`is`
import org.junit.Assert.assertThat
import org.junit.Test
import org.springframework.http.HttpStatus
import org.springframework.http.MediaType
import org.springframework.util.FileCopyUtils
import java.net.HttpURLConnection
import java.net.URL
import java.nio.charset.StandardCharsets

class HttpTest {
  @Test fun testHttpURLConnection() {
    val charset = MediaType(MediaType.APPLICATION_FORM_URLENCODED, StandardCharsets.UTF_8).charset!!
    val bytes = "grant_type=client_credentials&scope=identity".toByteArray(charset)

    val url = URL("https://www.reddit.com/api/v1/access_token")
    val connection = url.openConnection() as HttpURLConnection
    connection.connectTimeout = 30000
    connection.readTimeout = 30000
    connection.doInput = true
    connection.doOutput = true
    connection.instanceFollowRedirects = false
    connection.requestMethod = "POST"
    connection.addRequestProperty("Authorization", "Basic XDlvLWJJS0VTd1RzTEE6cm9mWG5Db1V4OHJTaj2W8UtGcx9QSUVmW2Lr")
    connection.addRequestProperty("Content-Type", "application/x-www-form-urlencoded;charset=UTF-8")
    connection.addRequestProperty("Accept", "application/json, application/x-www-form-urlencoded")
    connection.addRequestProperty("Content-Length", bytes.size.toString())
    connection.connect()
    FileCopyUtils.copy(bytes, connection.outputStream)

    val httpStatus = HttpStatus.resolve(connection.responseCode)
    assertThat(httpStatus, `is`(HttpStatus.OK))

    val responseBody = FileCopyUtils.copyToByteArray(connection.inputStream).toString()
    println(responseBody)
  }
}
```

So now we ask ourselves, "What do we know?"

"Well, we know that we keep getting errors. We also know that it works fine in a sane HTTP client, like Postman or cURL. So we know Java is being weird. But why..."

Then I looked closer. Let's ask ourselves again, "What do we know?"

"Well, we know we get a 429 error. Wait... 429? That's some !@#$ ain't it? Why is it giving us 429 Too Many Requests? Reddit's API is dropping us, isn't it? Hmm...."

Yep, it's confirmed. There's rate limiting going on. But if we're not DDoS'ing Reddit's API, who is? Well, as it turns out, everyone is. The rate limit is high enough for most API consumers to get by fine, but if you combine it with all the people making the same uneducated requests to Reddit's API that I am, you get quite a lot of noise. The answer, as it turns out, is here:

[Reddit Wiki - API](https://github.com/reddit-archive/reddit/wiki/API)

> Change your client's User-Agent string to something unique and descriptive, including the target platform, a unique application identifier, a version string, and your username as contact information, in the following format:
> * &lt;platform&gt;:&lt;app ID&gt;:&lt;version string&gt; (by /u/<reddit username>)
> * Example: User-Agent: android:com.example.myredditapp:v1.2.3 (by /u/kemitche)
> * Many default User-Agents (like "Python/urllib" or "Java") are drastically limited to encourage unique and descriptive user-agent strings.

If we add a User-Agent header (anything unique-ish basically), the minimal test case passes and prints our response.

```kotlin
package io.insource.api.demo

import org.hamcrest.Matchers.`is`
import org.junit.Assert.assertThat
import org.junit.Test
import org.springframework.http.HttpStatus
import org.springframework.http.MediaType
import org.springframework.util.FileCopyUtils
import java.net.HttpURLConnection
import java.net.URL
import java.nio.charset.StandardCharsets

class HttpTest {
  @Test fun testHttpURLConnection() {
    val charset = MediaType(MediaType.APPLICATION_FORM_URLENCODED, StandardCharsets.UTF_8).charset!!
    val bytes = "grant_type=client_credentials&scope=identity".toByteArray(charset)

    val url = URL("https://www.reddit.com/api/v1/access_token")
    val connection = url.openConnection() as HttpURLConnection
    connection.connectTimeout = 30000
    connection.readTimeout = 30000
    connection.doInput = true
    connection.doOutput = true
    connection.instanceFollowRedirects = false
    connection.requestMethod = "POST"
    connection.addRequestProperty("Authorization", "Basic XDlvLWJJS0VTd1RzTEE6cm9mWG5Db1V4OHJTaj2W8UtGcx9QSUVmW2Lr")
    connection.addRequestProperty("Content-Type", "application/x-www-form-urlencoded;charset=UTF-8")
    connection.addRequestProperty("Accept", "application/json, application/x-www-form-urlencoded")
    it.addRequestProperty("User-Agent", "This is a test")
    connection.addRequestProperty("Content-Length", bytes.size.toString())
    connection.connect()
    FileCopyUtils.copy(bytes, connection.outputStream)

    val httpStatus = HttpStatus.resolve(connection.responseCode)
    assertThat(httpStatus, `is`(HttpStatus.OK))

    val responseBody = FileCopyUtils.copyToByteArray(connection.inputStream).toString()
    println(responseBody)
  }
}
```

So why did cURL and Postman work? As it turns out, cURL works only sometimes. And postman must be silently adding a `User-Agent` header and not telling us. Either that, or it's not nearly as limited as the Java User-Agent (which is either `null` or another silent thing being added to requests that don't have a `User-Agent`, I'm unsure which).

So after all that, the answer is don't skimp reading the documentation. Especially if it's poorly organized and not cohesive or in one place like it should be.