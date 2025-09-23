---
layout: default
---
<div class="post-list">
  {% for post in site.posts %}
  <div class="post-card">
    <a href="{{ post.url }}">
      <img src="{{ post.image }}" alt="{{ post.title }}">
      <h2>{{ post.title }}</h2>
      <p>{{ post.description }}</p>
    </a>
  </div>
  {% endfor %}
</div>