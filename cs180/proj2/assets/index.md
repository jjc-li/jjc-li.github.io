---
layout: default
title: Project 2 Assets — Downloads
---

<section class="hero">
  <h1>Project 2 Assets</h1>
  <p class="muted">Browse and download the full-resolution figures generated for CS180 Project&nbsp;2. Links mirror the structure used in the writeup.</p>
</section>

<p>
  Individual files are listed by folder below. Click any filename to download it directly. The page updates automatically as new
  assets are added under <code>cs180/proj2/assets/</code> (no manual editing required).
</p>

{% assign proj2_files = site.static_files | where_exp: "f", "f.path contains '/cs180/proj2/assets/'" %}
{% assign grouped = proj2_files | group_by_exp: "f", "f.path | split: '/' | slice: 0, -1 | join: '/'" %}
{% assign grouped = grouped | sort: "name" %}

{% for dir in grouped %}
  {% assign display_path = dir.name | remove_first: '/cs180/proj2/assets/' %}
  {% assign clean_files = dir.items | where_exp: "f", "f.name != '.DS_Store' and f.name != '.gitkeep'" %}
  {% if clean_files == empty %}
    {% continue %}
  {% endif %}
  {% if display_path == '' %}
    {% assign display_path = 'root' %}
  {% endif %}
  <h2>{{ display_path | replace: '/', ' / ' | escape }}</h2>
  <ul>
  {% assign files = clean_files | sort: 'path' %}
  {% for file in files %}
      <li><a href="{{ file.path | relative_url }}" download>{{ file.name }}</a></li>
  {% endfor %}
  </ul>
{% endfor %}
