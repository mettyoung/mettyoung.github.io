---
layout: page
title: Projects
permalink: /projects/
---

Things I've built in my spare time, newest first.

{% assign featured = site.data.projects | where: "featured", true %}
{% assign others = site.data.projects | where_exp: "p", "p.featured != true" %}

<div class="projects">
{% for p in featured %}{% include project-card.html project=p featured=true %}{% endfor %}
{% for p in others %}{% include project-card.html project=p %}{% endfor %}
</div>
