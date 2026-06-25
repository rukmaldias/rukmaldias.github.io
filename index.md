---
layout: default
title: About
---

<div class="about-grid">
  <div class="about-img">
    <img src="{{ '/assets/images/profile.jpg' | relative_url }}" alt="Rukmal Dias" class="profile-img">
  </div>
  <div class="about-text">
    <h2>Rukmal Dias</h2>
    <p>
      Senior Android engineer based in Singapore, with 15+ years building mobile
      applications, security SDKs, and native systems. I work at the intersection
      of mobile and low-level security &mdash; cryptographic protocols, threat
      modelling, and writing Rust for native Android environments.
    </p>
    <p>
      Interested in AI-assisted development and distributed computing, both as
      tools that shape how software is built and as problems worth understanding
      deeply.
    </p>
    <div class="about-links">
      <a href="https://github.com/rukmaldias">github</a>
      <a href="mailto:rukmaldias@gmail.com">email</a>
      <a href="{{ '/resume' | relative_url }}">resume</a>
    </div>
  </div>
</div>

<hr>

<h2>Recent Posts</h2>

{% for post in site.posts limit:3 %}
<div class="post-entry">
  <h3><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h3>
  <p class="post-date">{{ post.date | date: "%Y-%m-%d" }}</p>
  {% if post.description %}<p class="post-desc">{{ post.description }}</p>{% endif %}
</div>
{% else %}
<p>No posts yet.</p>
{% endfor %}

{% if site.posts.size > 3 %}
<p><a href="{{ '/blog' | relative_url }}">all posts &rarr;</a></p>
{% endif %}
