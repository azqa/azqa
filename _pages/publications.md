---
layout: page
permalink: /publications/
title: Publications
description: See my Google Scholar for an updated list.
years: [2019,2020,2021,2022,2023]
nav: true
---

<div class="publications">

{% for y in page.years %}
  <h2 class="year">{{y}}</h2>
  {% bibliography -f papers -q @*[year={{y}}]* %}
{% endfor %}

</div>
