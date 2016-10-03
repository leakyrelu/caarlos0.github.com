---
layout: page
title: Stats
permalink: /stats/
---

### GitHub contributions chart:

<a href="https://github.com/{{ site.github_username }}">
  <img src="http://ghchart.rshah.org/{{ site.github_username }}"
    alt="{{ site.github_username }}'s Github chart" />
</a>

### Wakatime stats:

{% for chart in site.wakatime.charts %}

<figure>
  <embed src="https://wakatime.com/@{{ site.wakatime.username }}/{{ chart }}.svg" />
</figure>

{% endfor %}
