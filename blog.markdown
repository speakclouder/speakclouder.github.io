---
layout: page
title: Blog
permalink: /blog/
---


<div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-3 mb-10">
  {%- if site.posts.size > 0 -%}
      {%- for post in site.posts -%}
        {%- assign date_format = site.minima.date_format | default: "%b %-d, %Y" -%}
        <div class="flex-1 p-6 border rounded-lg shadow bg-gray-800 border-gray-700">
          <p class="font-light text-sm text-gray-400 pb-0 mb-0"><time>{{ post.date | date: date_format }}</time></p>
          <a href="{{ post.url | relative_url }}" class="text-xl no-underline hover:underline hover:decoration-amazon">
              <h5 class="">{{ post.title | escape }}</h5>
          </a>
          <p class="mb-3 font-normal text-gray-400">{{ post.excerpt }}</p>
          <a class="text-white bg-blue-700 hover:bg-blue-800 focus:ring-4 focus:ring-blue-300 font-medium rounded-lg text-sm px-3 py-2 text-center inline-flex items-center no-underline" href="{{ post.url | relative_url }}">
                Read more
            </a>
      </div>
    {%- endfor -%}

  {%- endif -%}

</div>

