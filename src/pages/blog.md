---
title: Blog
description: 'All blog posts can be found here'
layout: blog
pagination:
  data: collections.posts
  size: 6
permalink: 'blog/{% if pagination.pageNumber >=1  %}page-{{ pagination.pageNumber + 1 }}/{% endif %}index.html'
---

<p style="text-align: center;">
I yap here. Hopefully you find the yapping helpful.
Subscribe to get new posts via <a href="/feed.xml">RSS</a>.
</p>
