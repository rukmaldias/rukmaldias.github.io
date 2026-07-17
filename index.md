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
      Senior software engineer based in Singapore, with 15+ years across mobile,
      backend, and web systems. Deep foundation in Android and native security
      engineering &mdash; cryptographic protocols, threat modelling, and shipping
      SDKs for regulated fintech and government platforms.
    </p>
    <p>
      Building on that foundation across the full stack: React.js and Node.js on
      the front-end, Java and Spring Boot on the back-end, and PostgreSQL for
      data &mdash; connected end-to-end through Git, Jenkins CI/CD, and Docker.
      Actively working with AI-assisted development tools (GitHub Copilot,
      Claude, ChatGPT) as part of daily engineering practice, and interested in
      distributed computing and Rust for systems-level problems worth
      understanding deeply.
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
