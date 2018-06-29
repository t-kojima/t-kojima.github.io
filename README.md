# target site

Todays commit : [https://t-kojima.github.io/](https://t-kojima.github.io/)

## add new post (or draft)

```bash
hexo new post <page name>
hexo new draft <page name>
```

## deploy

```bash
hexo clean # clean public directory
hexo deploy -g
```

## blog card generator

[https://embed.ly/code](https://embed.ly/code)
[https://docs.embed.ly/docs/cards](https://docs.embed.ly/docs/cards)

### basic usage

```html
<a href="<url>" class="embedly-card" data-card-image="0" data-card-controls="0" data-card-align="left"></a>

<!-- if hide description -->
<a href="<url>" class="embedly-card" data-card-image="0" data-card-controls="0" data-card-align="left" data-card-description="0"></a>
```

```html
<!-- themes/landscape/layout/layout.ejs  -->
<script async src="//cdn.embedly.com/widgets/platform.js" charset="UTF-8"></script>
```

### firefox option

[https://addons.mozilla.org/ja/firefox/addon/format-link3/?src=userprofile](https://addons.mozilla.org/ja/firefox/addon/format-link3/?src=userprofile)
