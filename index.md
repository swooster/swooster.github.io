Posts
=====

{% for post in site.posts %}
  {% if post.wip == nil  %}
* [{{ post.title }}]({{ post.url }})
  {% endif %}
{% endfor %}

{% for post in site.posts %}
  {% if post.wip != nil  %}
<!-- {{ post.url }} -->
  {% endif %}
{% endfor %}
