---
title: "Categories"
layout: single
permalink: /categories/
author_profile: true
classes: wide
---

{% assign topic_keys = "" %}
{% for t in site.data.topics %}{% capture topic_keys %}{{ topic_keys }}|{{ t.category }}|{% endcapture %}{% endfor %}
{% for t in site.data.topics %}
{% assign posts = site.categories[t.category] %}
{% assign label = t.label | default: t.category %}
<section id="{{ t.category | slugify }}" class="taxonomy__section">
  <h2 class="archive__subtitle">{{ label }} <span class="taxonomy__count">{{ posts | size }}</span></h2>
  {% if posts %}
  <div class="entries-list">
  {% for post in posts %}{% include archive-single.html type="list" %}{% endfor %}
  </div>
  {% else %}
  <p class="page__meta"><i class="far fa-fw fa-file" aria-hidden="true"></i> 아직 글이 없습니다.</p>
  {% endif %}
</section>
{% endfor %}
{% for category in site.categories %}
{% capture key %}|{{ category[0] }}|{% endcapture %}
{% unless topic_keys contains key %}
<section id="{{ category[0] | slugify }}" class="taxonomy__section">
  <h2 class="archive__subtitle">{{ category[0] }} <span class="taxonomy__count">{{ category[1] | size }}</span></h2>
  <div class="entries-list">
  {% for post in category[1] %}{% include archive-single.html type="list" %}{% endfor %}
  </div>
</section>
{% endunless %}
{% endfor %}
