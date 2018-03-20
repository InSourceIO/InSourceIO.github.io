---
layout: post
subheadline: Weekly Update Series
title: Latest Projects and Other News
teaser: We're working on something big, which is why you haven't heard from us.
categories:
  - posts
tags:
  - google
  - mobile app
  - development
related:
  - /articles/google-cloud-platform-part-1.html
image:
    title: stock/finger-1920.jpg
    thumb: stock/finger-1920-t.jpg
    homepage: finger/finger-1920.jpg
---

## Latest Projects

Our latest projects have kept us pretty busy, so I wanted to give an update on where we're at and what we're up to. I can't share everything at this time, but much of it isn't too secret. Just time consuming.

### Business Plan

I spent the better part of last year (2017) developing a business plan. I know that sounds lame, but you'd be surprised what this can do for you. In this case, I wanted to make sure we had a solid footing for getting the company started. This meant exploring what the company was about, what kinds of clients we wanted to attract, and what the whole idea of being a software company was about. During this time, the website went through a major face-lift, but simultaneously I was exploring and developing a business plan.

While much of that work is relevant and should continue to be for some time, it is one of life's little ironies that spending lots of time "thinking" or "talking" about things is not where you find true inspiration. After this process, I felt pretty good about the company's foundation, and where I personally wanted to take it, and what we stood for overall. But shortly after, I stopped pursuing the goals I made for the company during that process. I haven't abandoned them, by any stretch, but full-fledged pursuit stopped in earnest.

### New Project

So what have we been up to since then? Thanks for asking. ;)

Not long after getting a business plan nailed down, I had an idea about a project I wanted to work on. It's nothing fancy, nothing earth-shattering, no trumpet sounds or velvet carpets were rolled out in the encountering of this idea. But that's what made this idea unique. It was very very simple. It offered a good place to start, and make a name for the company and myself as the Founder. And it had the potential to carry on in many forms, making it a very possible profit-center and viable idea for pitching to investors, recruiting new team members, etc.

Unfortunately, thinking of an idea and executing it are two different things. Thankfully though, I had quite a bit of motivation at that time to get through the initial push of developing the idea into a product that could function and be viable. I attempted to adopt an MVP model of development, but even so, this simple project quickly grew into a very ambitious endeavor. And it's taught me a lot about how this goes, and so I think it was a worthwhile distraction from launching the company full on.

I'll post a more detailed update on the project later, but suffice it to say this project has all the aspects of a good software development project in the modern age. It's basically a mobile app for Android and iOS, with a fairly sophisticated backend with multiple components including an API for the app frontend, and no fewer than two standalone processing applications.

The backend processing applications perform data processing of semi-structured data from various sources (a Node.js application written in modern Javascript), and loading of that data into a central data store (a Spring Boot application written in Kotlin). The latter also includes capabilities for real-time alerting of information that's changed (think mobile push notifications, email alerts and digests, web hooks, etc.) as well as scheduling and task status tracking. Of course this platform has to be hosted somewhere, so we went down the path of deploying to Google Cloud Platform (GCP), and ended up with a Cloud Function and two App Engine deployments. While it's somewhat analogous to microservices, there's not a lot of standalone services yet, just a few deployment units. GCP provides the backplane for adopting a microservices architecture should the need arise, so I'm hoping we don't need to spend much time on an in depth architecture journey here.

The frontend mobile app is currently being developed as a React Native app in JavaScript with JSX. We're still exploring this technology and getting a feel for it, but the general approach of applying web development skills to mobile app development appeals greatly.

## Next Steps

Since the project we're developing is for a seasonal sport, the current season of our target region (the contiguous United States) is winding down. We've unfortunately missed the window of doing a public beta this year in the US and having very relevant feedback. So we'll be looking to branch out and create a version of our app for a different region, to garner some international interest and get feedback. It will likely be a mostly English-speaking country in the southern hemisphere, like Australia or New Zealand. We may even find a reason to travel down there. Wouldn't that be a terrible tragedy...

If I find time, I'd like to tell you about some specific struggles and revelations encountered during this project. In particular, deploying Spring Boot apps with Google Cloud SQL connectivity to GCP's App Engine Standard environment is significantly under-documented. Look for a post of three about that.

For now, if you'd like to get ahold of us for a free consultation, just shoot us a line at our [Contact Us][1] page or contact us on Twitter [@InSourceOmaha][2].

[1]: https://insource.io/contact/
[2]: https://twitter.com/InSourceOmaha
