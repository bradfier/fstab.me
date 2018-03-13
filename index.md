title: Richard Bradfield
layout: default.liquid
---
{% for post in collections.posts.pages %}
#### <span class=stubdate>{{ post.published_date | date: "%Y-%m-%d" }}</span> [{{ post.title }}]({{ post.permalink }})
{{ post.excerpt }}
{% endfor %}
