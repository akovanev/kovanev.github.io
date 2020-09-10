---
layout: default
title: Home
---


{% for tag in site.tags reversed %}
  <h3 >{{ tag[0] }}
  {% if tag[0] == "Data Generator" %}
    <a href="https://github.com/akovanev/DataGenerator"><img style="display: inline; margin:0" src="/public/GitHub-Mark-32px.png"></a>
    <a href="https://www.nuget.org/packages/Akov.DataGenerator/"><img style="display: inline; margin:0" src="https://img.shields.io/nuget/v/Akov.DataGenerator"></a>
    <a href="https://www.nuget.org/packages/Akov.DataGenerator/"><img style="display: inline; margin:0" src="https://img.shields.io/nuget/dt/akov.datagenerator"></a>
  {% endif %}
  {% if tag[0] == "Sanitizer" %}
    <a href="https://github.com/akovanev/Sanitizer"><img style="display: inline; margin:0" src="/public/GitHub-Mark-32px.png"></a>
    <a href="https://www.nuget.org/packages/Akov.Sanitizer/"><img style="display: inline; margin:0" src="https://img.shields.io/nuget/v/Akov.Sanitizer"></a>
    <a href="https://www.nuget.org/packages/Akov.Sanitizer/"><img style="display: inline; margin:0" src="https://img.shields.io/nuget/dt/akov.sanitizer"></a>
  {% endif %}
  </h3>
  <ul>
    {% for post in tag[1] %}
      <li><a href="{{ post.url }}">{{ post.title }}</a></li>
    {% endfor %}
  </ul>
{% endfor %}