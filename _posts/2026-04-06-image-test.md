---
layout: post
title: "Testing Images in Jekyll"
date: 2026-04-06
categories: [test, setup]
image:
  path: /assets/headers/test-pic.jpg
  alt: "My Test Image"
  width: 100
  height: 800
published: true
---

# Image Testing 101

This post is a test to ensure the Dev Container and Jekyll can correctly render images from my local directory.

### 1. Standard Markdown Image
This is the most common way to add an image. We use a relative path starting from the root `/`.

![Test Image Description](/assets/img/test-pic.jpg)

### 2. Image with a Caption (HTML)
If you want to center your image or add a specific width (useful for those 200g chicken meal photos!), use HTML:

<div style="text-align: center;">
    <img src="{{ site.baseurl }}/assets/img/test-pic.jpg" alt="Centered Test" style="width: 80%; border-radius: 10px;">
    <p><em>Figure 1: A centered test image with rounded corners.</em></p>
</div>

---

### Why use `{{ site.baseurl }}`?
Using the Liquid tag `{{ site.baseurl }}` ensures that if your site is hosted at `phaneesh-git.github.io/project-name/` instead of the root, the image link won't break.
