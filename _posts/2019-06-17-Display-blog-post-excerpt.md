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

## Problem

The main page of my blog site displayed the entire length of each of my blog post. I wanted to display an excerpt of each blog post instead.



## Solution

I realized that this was because I separated all my paragraphs using one blank line, when I should use at least three. The reason is as follows.

In the file `index.html`, the code `<div class="excerpt">{{post.excerpt}}</div>` specifies that the main page displays an excerpt of each blog post. This excerpt is defaulted to the first paragraph of a blog post, which in turn is defined as a block of text separated from the second paragraph by at least three blank lines.