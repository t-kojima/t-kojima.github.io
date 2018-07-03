---
title: "[Hexo] リンクをブログカード風にしてみる"
date: 2018-06-29 17:12:20
tags:
- hexo
---

<blockquote class="embedly-card" data-card-controls="0"><h4><a href="https://embed.ly/">Embedly makes your content more engaging and easier to share | Embedly</a></h4><p>Embedly delivers the ultra-fast, easy to use products and tools for richer sites and apps.</p></blockquote>

[embed.ly](https://embed.ly/) の cards というサービスを使って、リンクをブログカード風に変えました。

<!-- more -->

## 発端

今までマークダウンのリンクを[こんな感じで](https://t-kojima.github.io/)そのまま貼っていたので、はてなブログカードの見た目がイケててうらやましかった。

## Embedly Card

hexo のプラグインで色々探していたけどとても丁度いいのがなかったが、そんな折り Embedly を見つけた。

<a href="https://nelog.jp/embedly" class="embedly-card" data-card-image="0" data-card-controls="0" data-card-align="left"></a>

### デフォルトだとちょっと使い辛い

Embedly のサイトから Card を生成できるが、デフォルトだと以下の点が気になった。

- アイキャッチがやたらでかい
- description がサイトによってはやたら長い
- 中央寄せしかできない

サイトのジェネレータからは一部の属性しか付与できないが、以下のドキュメントを見るに手動で追加できる属性がいくつかありそうだ

<a href="https://docs.embed.ly/docs/cards" class="embedly-card" data-card-image="0" data-card-controls="0" data-card-align="left"></a>

そこで以下のようにタグに属性を追加してやることで、上記の気になる点はいくつか解消できそうだ

```html
<a href="<url>" class="embedly-card" data-card-image="0" data-card-controls="0" data-card-align="left"></a>
```

これでアイキャッチは非表示にし、左寄せのカードになっている。場合によっては `data-card-description="0"` で description を消してしまってもいい

### CDN を読みこませる

上記の a タグは class を指定しているだけなので、当然あれだけではブログカードっぽくならない。以下のスクリプトもセットだ

```html
<script async src="//cdn.embedly.com/widgets/platform.js" charset="UTF-8"></script>
```

こちらはページロード時に一度読みこめばいい。

ただヘッダに追加してしまうと、明らかにページのロードが遅くなるので、Body の末尾に追加する。landscape テーマだと以下の場所だ

```html
<!-- themes/landscape/layout/layout.ejs -->
<%- partial('_partial/head') %>
<body>
  <div id="container">
    <!-- 省略 -->
  </div>
  <script async src="//cdn.embedly.com/widgets/platform.js" charset="UTF-8"></script>
</body>
</html>
```

## タグの自動生成

これで表示は問題ないが、あの a タグを毎回打ちたくはないので自動生成しよう

`ボクが普段使ってるのはFirefoxなので、Firefoxアドオンでやります`

<a href="https://addons.mozilla.org/ja/firefox/addon/format-link3/?src=userprofile" class="embedly-card" data-card-image="0" data-card-controls="0" data-card-align="left"></a>

Format Link というアドオンを Firefox に追加する。そして FormatLink settings に以下のフォーマットを追加してあげる

```html
<a href="{{url}}" class="embedly-card" data-card-image="0" data-card-controls="0" data-card-align="left"></a>
```

そしてリンクを作りたいサイトでコンテキストメニューを開き、Format Link を選ぶと上記のフォーマット形式で対象のリンクがクリップボードにコピーされる。あとはそれを貼り付けるだけだ。

## おわりに

はてなブログカードには及ばないと思うけど、結構見た目はいいし、マークダウンデフォルトのリンクよりかはかなりマシになったのではないかな！

## 7/4 追記

ブログカードの背景色が無しだとブログの文章とブログカードの内容の判別がつきにくいので、ブログカードに背景色をつけた。

`/themes/landscape/source/css/_embedly.styl` というファイルを作り以下の内容にして

```css
.embedly-card
    padding: 5px
.embedly-card-hug
    padding: 5px !important
    border-radius: 10px
    background: #EEE
```

`/themes/landscape/source/css/style.styl`に import する。これで見やすくなった。
