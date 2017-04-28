---
layout: page
title: Stats
permalink: /stats/
---

### GitHub contributions chart:

<a href="https://github.com/{{ site.author.github }}">
  <img src="http://ghchart.rshah.org/{{ site.author.github }}"
    alt="{{ site.author.github }}'s Github chart" />
</a>

### Wakatime stats:

{% for chart in site.author.wakatime.charts %}

<figure>
  <embed src="https://wakatime.com/@{{ site.author.wakatime.username }}/{{ chart }}.svg" />
</figure>

{% endfor %}
