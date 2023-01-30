---
layout: page
title: Blog
permalink: /blog/
---


<div class="home">
  {%- if site.posts.size > 0 -%}
    <ul class="list-none p-0">
      {%- for post in site.posts -%}
      <li class="p-0">
        {%- assign date_format = site.minima.date_format | default: "%b %-d, %Y" -%}
        <article class="">
          <header class="">
                <p class="font-light text-sm text-gray-400 pb-0 mb-0"><time>{{ post.date | date: date_format }}</time></p>
                <a class="text-xl no-underline font-extrabold lg:text-3xl hover:underline hover:decoration-amazon" href="{{ post.url | relative_url }}">{{ post.title | escape }}</a>
            </header>
          </article>
        {%- if site.show_excerpts -%}
          <p class="prose">{{ post.excerpt }}</p>
        {%- endif -%}
      </li>
      {%- endfor -%}
    </ul>

    <p class="rss-subscribe">subscribe <a href="{{ "/feed.xml" | relative_url }}">via RSS</a></p>
  {%- endif -%}

</div>
