---
layout: default
---
amin
<ul>
  {% for post in site.posts %}
    <li>{{ post.title }} — {{ post.date | date: "%Y-%m-%d" }}</li>
  {% endfor %}
</ul>