## Blogs
<ul class="no-bullets">
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a> {{ post.date | date: "%B %d, %Y" }}
    </li>
  {% endfor %}
</ul>
<br>

## Find me here

- [LinkedIn](https://www.linkedin.com/in/rikgroenewoud/)
- [Github](https://github.com/RikGr)