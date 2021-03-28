# Welcome

I hope you enjoy your time here!

## Articles

<ul>
    {% for post in site.posts %}
        <li>
            {{ post.date }} - <a href="{{ post.url }}">{{ post.title }}</a>
            {{ post.excerpt }}
        </li>
    {% endfor %}
</ul>

**Note: In case of any factual or grammatical error, please open an issue.**

## Contact

You can contact me via E-mail or via Discord (username: `Endless#5169`).
