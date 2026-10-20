---
layout: page
title: Home
permalink: /
--- 

<div class="hero">
  <h1>Keerthan Singh</h1>
  <p class="hero-subtitle">Penetration tester documenting real exploit chains — CTF machines, bug bounty finds, and security research.</p>
  <div class="hero-cta">
    <a href="/home/writeups/" class="btn btn-primary">Browse Writeups</a>
    <a href="/home/blog/" class="btn btn-secondary">Read the Blog</a>
    <a href="/home/about/" class="btn btn-secondary">About Me</a>
  </div>
</div>

<div class="latest-section">
  <h2>Latest Writeups</h2>
  <ul class="post-list">
  {% for post in site.categories.ctf limit:3 %}
    <li>
      <span class="post-meta">{{ post.date | date: "%b %-d, %Y" }}</span>
      <h3><a class="post-link" href="{{ post.url | relative_url }}">{{ post.title }}</a></h3>
    </li>
  {% endfor %}
  </ul>
</div>
