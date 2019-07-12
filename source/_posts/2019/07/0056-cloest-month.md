---
title: '直近のとある月を得たい'
date: 2019-07-10 21:59:32
categories:
  - Ruby
tags:
  - ruby
---

小ネタ
直近の n 月を取得したい。例えば 10 月なら、今日（7 月 10 日）から一番近いのは 2018 年 10 月、という感じ。

<!-- more -->

お題は単純に思えるけど、意外とスキッと書けない。
Rails の[Date](http://railsdoc.com/references/Date)をみても、翌月とか前月とか、n ヶ月前とか取得するメソッドはあれど、直近 n 月を取得するメソッドはない。

## 思いついたままやる

とりあえず何も考えず愚直に書いてみる

```ruby
def closest_month(target_month, date)
  year = date.month >= target_month ? date.year : date.prev_year.year
  Date.new(year, target_month, 1)
end

# 今日から見て直近の10月を取得（したい）
closest_month(10, Date.current)
```

この長さならワンライナーでもいい気がするな...

```ruby
def closest_month(target_month, date)
  Date.new(date.month >= target_month ? date.year : date.prev_year.year, target_month, 1)
end
```

## if文を排除

if文というか、三項演算子で条件判定が入っているのがなんか気になる。
対象の月が前だろうと後だろうと同じ式から導き出せないだろうか？と思って考えた。

```ruby
def closest_month(target_month, date)
  month_array = (target_month..12).to_a + (1...target_month).to_a
  date.prev_month(month_array.index(date.month)).beginning_of_month
end

# 今日から見て直近の10月を取得（したい）
closest_month(10, Date.current)
```

ちょっと複雑かな？と思うので解説

`month_array = (target_month..12).to_a + (1...target_month).to_a`ここで、例えば`target_month`が`10`とすると
`[10, 11, 12, 1, 2, 3, 4, 5, 6, 7, 8, 9]`という配列が得られる。

この配列から、当月のindexを引くと対象月との差分が得られる。
つまり当月が7月なら、`index=9`になるので、9ヶ月前が対象月の10月になる。

なので、`date.prev_month(month_array.index(date.month)).beginning_of_month`で差分だけ前の月を取得するという形だ。

うーん、なんかうまいことやってる感は出たけど
最初のワンライナーがシンプルでわかりやすいかも。

## もう少し簡素に

`[対象月から始まる配列].index(現在月) = 差分`という式で上の計算式は成り立っていてが
`[0〜11の配列][現在月 - 対象月] = 差分`という式でもいける。こっちのほうがちょっと短いかな

```ruby
def closest_month(target_month, date)
  date.prev_month((0...12).to_a[date.month - target_month]).beginning_of_month
end

# 今日から見て直近の10月を取得（したい）
closest_month(10, Date.current)
```

少しシンプルになった。。。気がする。

でも最初のワンライナーがわかりやすいかなぁ。
