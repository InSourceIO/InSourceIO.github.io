---
#
# Use the widgets beneath and the content will be
# inserted automagically in the webpage. To make
# this work, you have to use › layout: frontpage
#
layout: frontpage
header:
  image_fullwidth: header_unsplash_21.jpg
widget1:
  title: "Our Blog"
  url: "/blog/"
  image: startup-thumb.jpg
  text: "<em>Read about</em> the latest from InSource Software on our blog. Here you'll find posts from engineers about the things we're working on, such as search, big data, microservices, distributed systems, mobile development, advanced UX, and more."
widget2:
  title: "Our Technology"
  url: "/about/"
  text: "<em>Get familiar</em> with the technologies and solutions we use for our customers. We use the latest technologies and bring core competencies from several disciplines of software engineering to the table to help modernize legacy platforms and optimize large-scale systems, in addition to building out the technology needed to compete in emerging markets."
  image: coffee-beans-thumb.jpg
widget3:
  title: "Our Code"
  url: "https://github.com/InSourceSoftware/"
  image: widget-github-303x182.jpg
  text: "<em>Visit our</em> GitHub page to see samples of our code and keep track of what our developers are up to. We're active in open source, and regularly contribute to projects whose code powers our solutions, in addition to a myriad of libraries contributed back to the community based on general problems solved internally by our engineers."
#
# Use the call for action to show a button on the frontpage
#
# To make internal links, just use a permalink like this
# url: /getting-started/
#
# To style the button in different colors, use no value
# to use the main color or success, alert or secondary.
# To change colors see sass/_01_settings_colors.scss
#
callforaction:
  url: /contact/
  text: "Contact us for a free consultation ›"
  style: alert
permalink: /home/
---
