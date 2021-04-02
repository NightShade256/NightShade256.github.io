# Welcome

I hope you enjoy your time here!

## Articles

<ul>
    {% for post in site.posts %}
        <li>
            {{ post.date | date_to_string }} - <a href="{{ post.url }}">{{ post.title }}</a>
            {{ post.excerpt }}
        </li>
    {% endfor %}
</ul>

**Note: In case of any factual or grammatical error, please open an issue.**
