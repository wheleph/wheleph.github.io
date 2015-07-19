---
layout: post
title: How this blog is implemented
comments: true
---

I didn't write a lot of technical posts in my [legacy blog](http://wheleph.blogspot.com/) but when I did it was a bit painful because of lack of native syntax highlighting support.

Recently I started writing in [GFM](https://help.github.com/articles/github-flavored-markdown/) and realized that using markdown for my blog would be great. There are [different ways](http://blog.saliya.org/2014/08/blogging-with-markdown-in-blogger.html) to write posts in markdown. My choice was [Jekyll](http://jekyllrb.com/) which basically converts markdown files to static HTML.

With Jekyll you can:

- Write posts in markdown
- Use any text editor. I personally like GitHub's [Atom](https://atom.io/) because of its excellent GFM support
- Store posts under version control
- Host your Jekyll blog on [GitHub Pages](https://pages.github.com/) **for free**
- Install Jekyll locally and preview the posts (and even the whole blog) before publishing
- Publish via Git push

After some investigation I stumbled upon an [article](http://joshualande.com/jekyll-github-pages-poole/) that nicely describes experience of building a GitHub-hosted blog using [Jekyll](http://jekyllrb.com/) (with  [poole](https://github.com/poole/poole)). It also contains useful tweaks like implementing comments using [Disqus](https://disqus.com/) and adding [Google Analytics](http://www.google.com/analytics/) support. So if you're interested, go check it out!
