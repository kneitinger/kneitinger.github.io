---
layout: page
title: tidbits
permalink: /tidbits/
---

<div class="tidbit-subtitle">
a collection of pages that aren't quite tutorials and aren't quite
projects:  just little snippets of knowledge, or miscellaneous posts
</div>
---


<div class="home">
  <ul class="post-list">
    {% for post in site.posts %}
      {% if post.group == "tidbits" %}
        <li>
          <span class="post-meta">{{ post.date | date: "%b %-d, %Y" }}</span>
          <h3><a class="post-link" href="{{ post.url | prepend: site.baseurl }}">{{ post.title }}</a></h3>
          <p class="post-summary">{{ post.subtitle }}</p>
        </li>
      {% endif %}
    {% endfor %}
  </ul>
</div>
