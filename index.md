---
layout: default
---

<section class="posts-grid">
  {% assign posts_to_show = site.posts | where_exp:"p","p.published != false" %}
  {% if posts_to_show == empty %}
    <p>هنوز پستی منتشر نکرده‌اید — اولین پست خود را در پوشه <code>_posts/</code> اضافه کنید.</p>
  {% else %}
    {% for post in posts_to_show %}
      {% include post-card.html post=post %}
    {% endfor %}
  {% endif %}
</section>