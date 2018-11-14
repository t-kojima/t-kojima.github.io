---
title: '[Hexo] Blogに草生やす'
date: 2018-11-05 22:17:45
categories:
  - Hexo
tags:
  - hexo
---

[ブログのサイドバーに GitHub の「草」を載せました（その動機と「ブログに草」のススメ） - give IT a try](https://blog.jnito.com/entry/2018/11/04/200916)に触発されて草を生やしてみた。

<!-- more -->

# どのように実現しているのか

ネタ元の[GitHub の草状況を PNG 画像で返す heroku app をつくってみた - えいのうにっき](https://blog.a-know.me/entry/2016/01/09/222210)で語られているが、そもそもは Github のプロフィールから草部分（SVG）を抜いているようだ。

なるほどなー

と、もしかしてこれ自分にもできるんじゃないか？

## JavaScriptで抜いてこようとする

[Grass-Graph / Imaging your GitHub Contributions Graph](https://grass-graph.moshimo.works/)
