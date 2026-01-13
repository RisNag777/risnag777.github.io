---
layout: default
title: Home
---

This blog is to document my experiments with Agentic and Generative AI.  
I will describe my methodologies for different projects, my learnings and takeaways.  

You can find me on [LinkedIn](https://www.linkedin.com/in/rishab-nagaraj/)

## Blog Posts

<ul class="post-list">
  {% for post in site.posts %}
    <li>
      <span class="post-date">
        {{ post.date | date: "%Y-%m-%d" }}
      </span>
      —
      <a href="{{ post.url | relative_url }}">
        {{ post.title }}
      </a>
    </li>
  {% endfor %}
</ul>
