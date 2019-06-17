---
layout: post
title:  "Displaying blog post excerpt at main page"
date:   2019-06-17
categories: blog life-saver
tags: html
---

* content
{:toc}

This post refers to [how this particular blog site is configured](https://largecats.github.io/2019/06/17/Build-blog/). 

In the file `index.html`, the code `<div class="excerpt">{{post.excerpt}}</div>` specifies that the main page displays an excerpt of each blog post. This excerpt is defaulted to the first paragraph of a blog post, separated from the second paragraph by at least three blank lines. If all paragraphs are separated by fewer than three blank lines, the entire blog post would be displayed at the main page.