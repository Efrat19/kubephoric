---
layout: post
title: Categories
description:  List of all posts organized by categories.
permalink: /categories/
---

<div class="article">
{% for category in site.categories %}
    {% capture category_name %}{{ category | first }}{% endcapture %}
        <h3 class="category-head" id="{{ category_name }}">{{ category_name }}</h3>
        <ul>
        {% for post in site.categories[category_name] %}
        <!-- <div class="article"> -->
            <li>
                <a class="clearlink" href="{{ site.baseurl | prepend: site.url }}/{{ post.title | slugify }}">
                    <b>{{ post.title }}</b>
                </a>
                <div class="read-more clearfix">
                    <div class="more-left left">
                        <span class="date"><time>{{ post.date | date: '%B %d, %Y' }}</time></span>
                        <!-- <span>tags:&nbsp;</span> 
                        <span class="posted-in"><a href='numerical.html'>{{ post.tags }}</a>
                        </span> -->
                    </div>
                </div>
            </li>
        <!-- </div> -->
        {% endfor %}
        </ul>
{% endfor %}
</div>
