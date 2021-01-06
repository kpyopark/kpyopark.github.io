---
layout: page
title: Blog
permalink: /blog/
---

# blog

도움이 되는 내용으로 가득 채우겠습니다.

*  {% for post in site.posts %} {% capture y %}{{post.date \| date:"%Y"}}{% endcapture %} {% if year != y %} {% assign year = y %}
* {{ y }}
*  {% endif %}
*  {{ post.date \| date:"%Y-%m-%d" }} [{{ post.title }}](https://github.com/kpyopark/kpyopark.github.io/tree/3b525eec0b24ef7a56e3facd68f5f92fe01212b6/%7B%7B%20post.url%20%7D%7D)
*  {% endfor %}

