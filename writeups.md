---
layout: page
title: Writeups
permalink: /writeups/
---

{% for post in site.categories.ctf %}
### [{{ post.title }}]({{ post.url | relative_url }})
*{{ post.date | date: "%b %d, %Y" }}*
{% endfor %}
