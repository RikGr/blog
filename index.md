<head>
{% if site.google_analytics and jekyll.environment == 'production' %}
{% include analytics.html %}
{% endif %}
</head>

## Blogs
<ul>
  {% for post in site.posts %}
    <li>
      {{post.date}}<a href="{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>

## Find me here
- [Twitter](https://twitter.com/RikGroenewoud)
- [LinkedIn](https://www.linkedin.com/in/rikgroenewoud/)
- [Xpirit](https://xpirit.com/team/rik-groenewoud/)
- [Github](https://github.com/RikGr)