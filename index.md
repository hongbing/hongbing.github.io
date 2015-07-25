---
layout: home
---

<div id="header_wrap" class="outer">
<header class="inner">
<h1 id="project_title">hongbing的部落格</h1>
<h2 id="project_tagline">Once a geek, forever a geek.</h2>
</header>
</div>

<div class="index-content blog">
    <div class="section">
        <div class="cate-bar"><span id="cateBar"></span></div>

        <ul class="artical-list">
        {% for post in site.categories.blog %}
            <li>
                <h2><a href="{{ post.url }}">{{ post.title }}</a></h2>
                <div class="title-desc">{{ post.description }}</div>
            </li>
        {% endfor %}
        </ul>
    </div>
</div>

<div id="footer_wrap" class="outer">
    <footer class="inner">
    </footer>
</div>
