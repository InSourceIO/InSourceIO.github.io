---
layout: post
subheadline: Weekly Update Series
title: How To Create an Amazing Website
teaser: An inside look at creating an full-featured and amazing website.
categories:
  - posts
tags:
  - websites
  - development
image:
    title: stock/feet-1920.jpg
    thumb: stock/feet-1920-t.jpg
    homepage: stock/feet-1920.jpg
---
## Disclaimer

I should mention up front that making amazing websites is not necessarily a core business model of ours, but it certainly will be high on the list of things we do, and intend to do well. So you may be thinking: Why write a post giving away anything remotely secret sauce?

Well, if you have to ask, you should know that there is no such thing as secret sauce. Period. So that's settled. But further, I want to write about what it takes to be a great developer and do great things with code. In a large majority of cases (actually all cases), people reading this will fall into two camps.

1. Developers. You aren't going to pay me to build your site anyway. So I give this info away for free because it's helpful, and lets you know who I am and what I'm about, which certainly benefits me.
2. Non-developers. You will potentially pay me to build your site. So this information will serve to inform you of why my services are valuable and gives you an in-depth look at how it works, giving you a reason to do business with us, which definitely benefits me.

## The Business

Here we go. Let me boil it down into very digestible chunks. Easier to write, easier to read. 'Nuff said.

### Design

Design is a big part of building a great website. I won't belabor the point here, as I've talked about the main focus in previous posts.

First, always start with a purchased theme. This helps streamline the design process. Second, focus on your brand. This is why I advocate for developing a personal brand, which helps if you are developing a personal website. If you're developing a corporate website, it should go without saying that focusing on your brand is a formula for success.

With this website, I focused on the colors of the logo. In particular, I modified the primary color of my purchased theme to propagate my logo's aesthetic throughout the theme. This resulted in the following color palette:

<style type="text/css">
    .palette-square {
        display: inline-block;
        height: 96px;
        width: 96px;
        margin: 5px;
        color: #fff;
        text-align: center;
        line-height: 96px;
    }
    .color1 {
        background-color: #f68522;
    }
    .color2 {
        background-color: #071c33;
    }
    .color3 {
        background-color: #34495e;
    }
    .color4 {
        background-color: #546e7a;
    }
</style>
<div class="text-center">
    <div class="palette-square color1">#34495e</div>
    <div class="palette-square color2">#071c33</div>
    <div class="palette-square color3">#34495e</div>
    <div class="palette-square color4">#546e7a</div>
</div>

Notice that I specifically chose not to use the dark colors in my logo. This is because the theme had a great balance already with the non-primary colors, and because my logo would look too bland on top if its colors were used as background colors.

### Structure

A less obvious aspect of a great website is its structure. You may be wondering why this is important. Think of it this way. If you were to build a house, and someone said to you, "We need to focus on the foundation and structure first" you'd assuredly agree with them, knowing that these things are important. The same is true of a website. Getting the structure right improves the odds of long term maintainability.

When dealing with mostly static websites, structure amounts to the tools you use and the organization you apply to the reusable pieces (we'll call them includes) and page styles (we'll call them layouts) of your website.

In the case of this website, the purchased theme came with several distinct page styles. I designed several layouts so I could easily apply the various page styles on any page in the site.

* home - The default home page style, with corporate info banner, elaborate header, specially styled menu and corporate footer
* home-min - A minimal home page style, with simple header and simple footer, useful for alternate landing pages or for a sitemap, etc.
* jumbotron - A very visual layout containing only a header with a special background
* default - The base layout for other pages, containing a header, menu, and generic footer
* minimal - Extends the default layout with a breadcrumb
* page - Extends the minimal layout with optional sidebars, content section, and pagination
* post - An alternative to the page layout specialized for blog posts, containing numerous post-specific elements such as image, meta, social media sharing, related posts, comments, etc.
* video - Extends the post layout with an embedded video

As you start building out your layout options, you will quickly run into code duplication. This is an opportunity to define your includes. In this case, I took the time to define some unused includes as well, that give me options for redefining the look and feel at a moments notice. Things like the style and color scheme used in the header and footer of the default and home layouts. Other includes were things like copyright and social media icons, breadcrumb, meta tags in the head of the page, script includes after the footer, complex parts of the navbar, etc. Ultimately, I built several ready-made versions of the header and footer which made developing the above layouts a breeze.

* header-jumbotron
* header-min
* header-parallax
* header-transparent
* footer-dark
* footer-light
* footer-light-min
* footer-widget-dark
* footer-widget-light

Because the theme I purchased was so full featured, I had to pick and choose which pieces to put into includes and make available for later use. I actually left something like 30% of the theme on the cutting room floor, as there were some obscure options that I knew I would never use.

### Content

Once you have the structure of your site figured out, content should be the primary focus. It's easy to spend a ton of time on the structure and run out of steam at this point, especially if you're building your own website for yourself. What's fun about content is that as you build a website and add features to it (such as a blog, image gallery, post archives, etc.) you get to mix this step with another structure exercise. But focusing specifically on the content of the site and how it will impact visitors needs to stay in view, or else you can lose yourself in an endless loop.

With the structure step done (specifically layout w/ header, footer), you can begin to apply the layout(s) to different pages of the site. Again, having a purchased theme saves you from losing too much time here. For this site, I heavily used the demo gallery from the theme to pick and choose what page elements I wanted to use. It became a [styleguide-driven development](https://eligible.com/blog/styleguide-driven-development/) exercise.

I developed the following meta-content pages first:

* home - The landing page of the site, with some dangling carrots for other content across various other pages
* blog - The default blog index with pagination
* blog index - A 2-column layout of blog posts, used for categories and tags
* archive - A special version of a blog archive from the theme (this was tricky to develop, and still isn't perfect)

Then I developed various infrastucture pages:

* RSS feed
* Atom feed
* Robots file
* Sitemap file
* 404 page
* Coming soon page
* Maintenance page
* Login page
* Registration page
* Email confirmation page
* Unsubscribe page
* Terms of service page
* Contact Us page
* Search results page

Lastly, I spent time building out the website-specific content pages. These will be specific to the message you want to convey with your site, and how you want to engage visitors. It is critically important. But there's no how-to guide or checklist for this part. And it's sad that this has to come last. But if everything else has been done, the real meat-and-potatoes content can be done and re-done or tweaked as many times as necessary. And all the work put into the structure, meta-content, and infrastructure pieces of the site will shine in the final details added here.


## Final Words

I hope this post has shown you in brief just how much goes into making a website. And I glossed over most of it. It truly is a very large-scale and ambitious project. There are certainly shortcuts that will get you halfway there without all the work mentioned above. But the shortcuts and half-measures will speak volumes about your business or who you are to your site's visitors. Taking the time to build a tailored, well-designed, full-featured custom website will give you a compelling platform for all of the content, ideas, engagements, campaigns, interactions, advertising, commerce, and any other thing a website can be used for.

If you're interested in talking to us about designing your website (whether using a fully custom method as outlined above, or something simpler but still tailored to you), get in touch with us today: [contact@insource.io](mailto:contact@insource.io) or use the [Contact Us](/contact/) form.
