---
layout: page
title: Archive
permalink: /archive/
---
<div class="archive">
  <div class="archive-list">
  {% if site.posts.size == 0 %}
    <h2>No post found</h2>
  {% else %}
  {% for post in site.posts %}
    {% unless post.next %}
      <h3>{{ post.date | date: '%Y' }}</h3>
    {% else %}
      {% capture year %}{{ post.date | date: '%Y' }}{% endcapture %}
      {% capture nyear %}{{ post.next.date | date: '%Y' }}{% endcapture %}
      {% if year != nyear %}
        <h3>{{ post.date | date: '%Y' }}</h3>
      {% endif %}
    {% endunless %}
    <li class="archive-list-post">
      <a href="{% if post.link %}{{ post.link }}{% else %}{{ post.url | prepend: site.baseurl }}{% endif %}" class="archive-list-post-title">{{ post.title }}</a>
      <span class="archive-list-post-date">
        <time>| {{ post.date | date:"%b %d" }}</time>
      </span>
    </li>
  {% endfor %}
  {% endif %}

  </div>
</div>
