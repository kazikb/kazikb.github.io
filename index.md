---
layout: default
title: Home
---

<div class="container">
  <!-- POSTS -->
  <div class="main">
    <h1>Posts</h1>
    <ul id="posts">
      {% for post in site.posts %}
        <li data-tags="{{ post.tags | join: ' ' }}">
          <a href="{{ post.url }}">{{ post.title }}</a>
          <small>{{ post.date | date: "%Y-%m-%d" }}</small>
        </li>
      {% endfor %}
    </ul>
  </div>

  <!-- SIDEBAR -->
  <div class="sidebar">
    <h2>Tags</h2>
    <ul>
      <li>
        <a href="#" onclick="setTag('all'); return false;">All</a>
      </li>

      {% assign tags = site.tags | sort %}
      {% for tag in tags %}
        <li>
          <a href="#" onclick="setTag('{{ tag[0] }}'); return false;">
            {{ tag[0] }} ({{ tag[1].size }})
          </a>
        </li>
      {% endfor %}
    </ul>
  </div>
</div>

<script>
function filterTag(tag) {
  const posts = document.querySelectorAll("#posts li");

  posts.forEach(post => {
    const tags = post.dataset.tags.split(" ");

    if (tag === "all" || tags.includes(tag)) {
      post.style.display = "";
    } else {
      post.style.display = "none";
    }
  });
}

// update URL without reload
function setTag(tag) {
  const url = new URL(window.location);

  if (tag === "all") {
    url.searchParams.delete("tag");
  } else {
    url.searchParams.set("tag", tag);
  }

  window.history.pushState({}, "", url);
  filterTag(tag);
}

// read tag from URL
function getTagFromURL() {
  const params = new URLSearchParams(window.location.search);
  return params.get("tag") || "all";
}

// initial load
document.addEventListener("DOMContentLoaded", () => {
  const tag = getTagFromURL();
  filterTag(tag);
});

// handle back/forward navigation
window.addEventListener("popstate", () => {
  const tag = getTagFromURL();
  filterTag(tag);
});
</script>
