---
layout: post
subheadline: Weekly Update Series
title: Weekly Update, August 22nd
teaser: How to host a site for free (or at least the cost of a domain).
categories:
  - posts
tags:
  - weekly update
  - github
image:
    title: stock/matrix-1920.jpg
    thumb: stock/matrix-1920-t.jpg
    homepage: stock/matrix-1920.jpg
---
For this week's update, I'd like to focus on sharing some additional details of the journey in making this website. This time up: How to host your site for free.

## Step 1: Switch to using a static site generator

If you've followed along with some of our recent posts, you know we're using Jekyll to generate this site. Jekyll is a static site generator. It's not the only one by any means, and if you have a reason to use another, I encourage you to do so.
 
But before we get into what static site generators are, here's a bit of history on the web.

### 1990s

The early days of the [world wide web][1] (not to be confused with the [internet][2]) was a very labor-intensive time for developers who maintained websites. There were few tools to do anything sophisticated, like generate dynamic content, or publish large complex websites. So websites remained simple out of necessity.

In a majority of cases, websites were built using completely static pages, and in some cases only a single page. Websites had simple headers and footers (if any), and barely any navigation at all. Duplication of content and <abbr title="Hyper-text Markup Language">HTML</abbr> markup was common. The `<table>` element was used heavily for layout, and `<font>` tags were the most common markup found in the page source.

However, it wasn't all bad. The majority of the best ideas for how websites and web pages should be organized, structured, and laid out were pioneered during this time. In fact, a lot of what we do today (nearly 25 years later) was already present in a basic form in some of the better websites of the time.

Additionally, early pioneers of web technology began to use <abbr title="Common Gateway Interface">CGI</abbr> scripts to dynamically generate some content. However, security issues were prevalent and sophistication was certainly lacking in the capabilities of brute force scripts written in bash-like interpreters.

### 2000s

Within half a decade, the web had progressed in leaps and bounds in its technology, tooling, and capabilities. Web standards had progressed significantly, and most of what we now think of as the modern web was beginning to come into existence. In particular, CGI scripts were replaced with interpreted scripting languages, allowing for much richer capabilities for generating dynamic content.

This led to the rapid rise in popularity of scripting languages such as Perl, PHP, and Ruby, among many others. These languages and scripting engines were quickly adopted by web developers, seeking to build websites faster to keep up with the expanding demand for content and web presence.

Websites during this time were typically generated on a server, which would use templates, includes, or partials to reduce code duplication, and send generated HTML markup down to the web browser on demand. Frameworks were created to make this style of development even easier and more sophisticated, allowing for rapid development and the invention of new paradigms of programming, such as the <abbr title="Model-View-Controller">MVC</abbr> pattern. During this time, relational database technology was used heavily to drive more and more dynamic content, which created the now ubiquitous <abbr title="Content Management System">CMS</abbr>.

However, something was lost in this transition, namely the simplicity of hosting static sites. Previously, only a web server capable of serving data (such as HTML pages, images, javascript, and stylesheets) was needed. With interpreted scripting languages, an interpreter or processing engine was needed, which had to be bundled with or baked into the web server itself. A popular choice was the Apache web server with a bundled PHP interpreter baked in, and a MySQL database thrown in for extra capability. This popular stack (one among many) became known as LAMP, or Linux-Apache-MySQL-PHP. Hosting your site on a non-trivial stack was now required, which cost money and quickly became a big business.

### 2010s

Ten years into a new millennium, not only had web development capabilities progressed, but enterprise software development had matured as well. This only meant that hosting websites and enterprise web applications was a booming business model, required specialized skills for development, and was more in demand than ever.

However, for the non-technical masses, building a website meant using the CMS platform to the fullest extent possible. Every single shred of content on a website was generated dynamically from data stored in a database, including the templates themselves in many cases. A new business model was born, and <abbr title="Do it yourself">DIY</abbr> or <abbr title="What you see is what you get">WYSIWYG</abbr> website builders were commonplace.

But a few particularly stable and established technical minds had not forgotten (how could they have, it was only like 11 years ago) the origins of the web, and their simplicity. By 2010, GitHub was already taking off in popularity, and founder Tom Preston-Werner was thinking ahead. He's not the only one of course, but for our purposes, GitHub is the major player in reigning in the collective insanity that had now gripped the rest of the planet.

### GitHub Pages

Instead of requiring databases, hosting providers, interpreters, scripting engines and dynamically generated content, the web was largely a collection of static (or slow changing) files that could still be hosted with little more than a regular web server. GitHub Pages was born from the idea that all (or at least all the good) open source projects in existence would need websites to host documentation, demos, and other related content, and that it would change infrequently. So why not host all of that as a static site? So they did.

### So what's a static site generator...?

A static site generator is basically a combination of the best aspects of the now traditional generated content capabilities of scripting languages like Ruby, and the simplicity of a fully static site from the late '90s. The idea is that you specify the content of your site using any data representation you like, such as a database, static files, a directory structure, etc. and build a framework or tool for combing through all that data and generating a full website at build time. Since content typically changes infrequently, this means we can run a script once to generate an entire site, then upload that as static files to be served up by a simple web server.

In most cases, a static site generator is a very simple affair. In fact, building your own is no more complex than authoring a dynamically generated website from scratch, which any more can be accomplished in a few hours to several days at the most.

But there are a host of choices for off-the-shelf generators which include advanced features ranging from generating blogs to full blown content management. This is simply a mirror image of the same thing you see in modern web platforms for mainline development.

Lately, even <abbr title="Single Page Application">SPA</abbr> javascript frameworks are now offering static site capabilities for developers looking to capitalize on their investment in learning a modern client-side framework. This is still a relatively new practice (at the time of this writing), and only a few of the most popular SPA frameworks have this feature mature enough to be of any real use.

## Step 2: Use GitHub Pages

Whether you're using Jekyll which is fully supported by GitHub Pages, any of the umpteen static site generators available, or you even built your own (a fun weekend project), you should host your site with GitHub Pages. There simply isn't a better choice, and it's completely free. It's one of the ways that GitHub encourages you to use their platform, and in my opinion, it is working out for them, and has for some time.

When you author your site using your chosen generator, you simply have to upload the resulting generated site to GitHub. There are modern GUI tools (again, have been for some time) that make this process about as simple as one click. The ironic part about this is that using a tool like Git is required, which for whatever reason, has remained a fairly advanced skill that hasn't been adopted by non-technical users. I have a feeling that will change soon.

## Conclusion

Other than the wall of text I threw at you off the top of my head about the history of the web, there's not much to this.

* Use a static site generator
* Use GitHub Pages

If you do these two things, you can host websites for free forever.

If you have detailed questions that you would like to see covered, please post a comment below. I would love to have some non-technical readers ask to learn more about the "superpowers" required to do this.

[1]: https://en.wikipedia.org/wiki/World_Wide_Web
[2]: https://en.wikipedia.org/wiki/Internet
