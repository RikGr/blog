<head>
{% if site.google_analytics and jekyll.environment == 'production' %}
{% include analytics.html %}
{% endif %}
</head>




## Blog posts
<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>

## Find me here 
- [LinkedIn](https://www.linkedin.com/in/rikgroenewoud/)
- [Github](https://github.com/RikGr)
- [Xpirit](https://xpirit.com/team/rik-groenewoud/)




