---
title: '[Hexo] ソーシャルボタンを常に表示'
date: 2018-10-17 11:07:07
tags:
  - hexo
---

このブログでは Hexo の [Landscape](https://github.com/hexojs/hexo-theme-landscape) テーマを使っている。デフォルトのやつだ。

エントリの右下に「共有」リンクがあり、それをクリックすることでソーシャルボタンがポップアップするのだが、なんだか分かり辛いような気がしたので常に表示するように変更した。

<!-- more -->

今の表示はこんな感じ

![わかりづらい](/images/46-03.png)

# ソーシャルボタンの常時表示

まずソーシャルボタンの表示/非表示を切り替えている個所を探す。

ソースを読み進めていくと、[themes/landscape/layout/\_partial/article.ejs](https://github.com/hexojs/hexo-theme-landscape/blob/master/layout/_partial/article.ejs#L25)と[themes/landscape/source/js/script.js](https://github.com/hexojs/hexo-theme-landscape/blob/master/source/js/script.js#L34)辺りのようだ。

article.ejs

```html
<footer class="article-footer">
  <a data-url="<%- post.permalink %>" data-id="<%= post._id %>" class="article-share-link"><%= __('share') %></a>
  <% if (post.comments && config.disqus_shortname){ %>
    <a href="<%- post.permalink %>#disqus_thread" class="article-comment-link"><%= __('comment') %></a>
  <% } %>
  <%- partial('post/tag') %>
</footer>
```

script.js

```js
var html = [
  '<div id="' + id + '" class="article-share-box">',
  '<input class="article-share-input" value="' + url + '">',
  '<div class="article-share-links">',
  '<a href="https://twitter.com/intent/tweet?url=' +
    encodedUrl +
    '" class="article-share-twitter" target="_blank" title="Twitter"></a>',
  '<a href="https://www.facebook.com/sharer.php?u=' +
    encodedUrl +
    '" class="article-share-facebook" target="_blank" title="Facebook"></a>',
  '<a href="http://pinterest.com/pin/create/button/?url=' +
    encodedUrl +
    '" class="article-share-pinterest" target="_blank" title="Pinterest"></a>',
  '<a href="https://plus.google.com/share?url=' +
    encodedUrl +
    '" class="article-share-google" target="_blank" title="Google+"></a>',
  '<a href="https://www.linkedin.com/shareArticle?mini=true&url=' +
    encodedUrl +
    '" class="article-share-linkedin" target="_blank" title="LinkedIn"></a>',
  '</div>',
  '</div>',
].join('');

var box = $(html);

$('body').append(box);
```

雑に説明すると`.article-share-link`がクリックされた時、リンクの一覧を組み立てて body に追加している。

ということはここからクリックイベントを抜いてしまえばいい。というわけで以下のコードを同じ場所に追加する。

```js
(function() {
  var $this = $('.article-share-items'),
    url = $this.attr('data-url'),
    encodedUrl = encodeURIComponent(url),
    id = 'article-share-items-' + $this.attr('data-id');

  var html = [
    '<div id="' + id + '" class="article-share-links">',
    '<a href="https://twitter.com/intent/tweet?url=' +
      encodedUrl +
      '" class="article-share-twitter" target="_blank" title="Twitter"></a>',
    '<a href="https://www.facebook.com/sharer.php?u=' +
      encodedUrl +
      '" class="article-share-facebook" target="_blank" title="Facebook"></a>',
    '<a href="http://pinterest.com/pin/create/button/?url=' +
      encodedUrl +
      '" class="article-share-pinterest" target="_blank" title="Pinterest"></a>',
    '<a href="https://plus.google.com/share?url=' +
      encodedUrl +
      '" class="article-share-google" target="_blank" title="Google+"></a>',
    '</div>',
  ].join('');

  var box = $(html);
  $('footer').append(box);
})();
```

article.ejs も以下のように変更する。

```html
<footer class="article-footer">
  <div data-url="<%- post.permalink %>" data-id="<%= post._id %>" class="article-share-items"></div>
  <!-- <a data-url="<%- post.permalink %>" data-id="<%= post._id %>" class="article-share-link"><%= __('share') %></a> -->
  <% if (post.comments && config.disqus_shortname){ %>
    <a href="<%- post.permalink %>#disqus_thread" class="article-comment-link"><%= __('comment') %></a>
  <% } %>
  <%- partial('post/tag') %>
</footer>
```

もともとあった a タグ（共有リンク）をコメントアウトし、その前に同じ属性を持った div タグを追加している。これは`data-url`と`data-id`を script.js に引き継ぐためだ。

酷く不格好な形だがしょうがない。僕は jQuery が得意では無いし、あまり時間も掛けたくない。。。！

## css も直す

このままだとボタン群が左に寄せられてしまうので、以下のように右寄せにする。

themes/landscape/source/css/\_partial/article.styl

```stylus
.article-share-links
  // clearfix()
  // background: color-background
  float: right
  margin-left: 20px
```

もう一つ、footer にボタンを表示するようにすると、`.article-footer`の css の影響を受けてしまい、ボタンを hover してもアイコンの色が変わらない。

その為、以下のように`.article-share-link`の color を`!important`することで優先させる。

```stylus
$article-share-link
  width: 50px
  height: 36px
  display: block
  float: left
  position: relative
  color: #999
  text-shadow: 0 1px #fff
  &:before
    font-size: 20px
    font-family: font-icon
    absolute-center(@font-size)
    text-align: center
  &:hover
    // color: #fff
    color: #fff !important
```

ここまででボタンが常に表示されるようになった。

![ボタンを常に表示](/images/46-01.png)

# はてなブックマークボタンを追加

続いてはてなブックマークボタンを追加してみたい。何故なら僕が他人のブログを見る時、一番よく使うボタンだからだ。

## font-awesome にアイコンが無い問題

twitter だとか facebook だとか、これらのアイコンは font-awesome のものを使っている。css(stylus)の指定はこんな感じになっている。

themes\landscape\source\css_partial\article.styl

```stylus
.article-share-twitter
  @extend $article-share-link
  &:before
    content: "\f099" // <- ココ
  &:hover
    background: color-twitter
    text-shadow: 0 1px darken(color-twitter, 20%)
```

しかし font-awesome にはてなブックマークのアイコンは無いので、何とかしなければならない…

で、色々探していたら[普通のフォントで代用できるんじゃないの？](https://hayashikejinan.com/webwork/css/913/)というサイトがあった。これは良さそう！

上記サイトを参考に、以下の定義を追加する。

```stylus
.article-share-hatebu
  @extend $article-share-link
  &:before
    content: "B!"
    font-family: Verdana
    font-weight: bold
  &:hover
    background: color-hatebu
    text-shadow: 0 1px darken(color-hatebu, 20%)
```

はてぶカラーも追加

themes\landscape\source\css_variables.styl

```stylus
// Colors
color-default = #555
color-grey = #999
color-border = #ddd
color-link = #258fb8
color-background = #eee
color-sidebar-text = #777
color-widget-background = #ddd
color-widget-border = #ccc
color-footer-background = #262a30
color-mobile-nav-background = #191919
color-twitter = #00aced
color-facebook = #3b5998
color-pinterest = #cb2027
color-google = #dd4b39
color-hatebu = #00a4de // <- 追加
```

最後に、script.js にリンクを追加して完了

```js
var html = [
  '<div id="' + id + '" class="article-share-links">',
  '<a href="https://twitter.com/intent/tweet?url=' +
    encodedUrl +
    '" class="article-share-twitter" target="_blank" title="Twitter"></a>',
  '<a href="https://www.facebook.com/sharer.php?u=' +
    encodedUrl +
    '" class="article-share-facebook" target="_blank" title="Facebook"></a>',
  '<a href="http://pinterest.com/pin/create/button/?url=' +
    encodedUrl +
    '" class="article-share-pinterest" target="_blank" title="Pinterest"></a>',
  '<a href="https://plus.google.com/share?url=' +
    encodedUrl +
    '" class="article-share-google" target="_blank" title="Google+"></a>',
  '<a href="http://b.hatena.ne.jp/entry/' +
    encodedUrl +
    '" class="article-share-hatebu" target="_blank" title="Hatena Bookmark"></a>', // <- 追加
  '</div>',
].join('');
```

はてなブックマークボタンが追加された！

![はてなブックマークボタンの追加](/images/46-02.png)

めでたしめでたし

# 参考

- [Font Awesome などアイコンフォントにないはてなブックマークを自力で追加する簡単な方法](https://hayashikejinan.com/webwork/css/913/)
