---
title: Now
layout: archive
collection: now
permalink: /now/
author_profile: false
---

> To replace the social media updates I keep this page inspired by [Derek Sivers](http://sivers.org/) and the [/now page movement](https://nownownow.com/about).

{% assign now_pages = site.now |sort:'date' | reverse %} 
{%- for post in now_pages -%} 
  {% include archive-single.html %} 
{%- endfor -%}