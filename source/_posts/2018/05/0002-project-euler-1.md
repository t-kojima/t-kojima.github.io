---
title: 'Project Euler # 1'
date: 2018-05-10 00:40:20
tags:
  - euler
---

Haskell の勉強がてら、[Project Euler](https://projecteuler.net/)を解いてみる。

<!-- more -->

## Multiples of 3 and 5

<a href="https://projecteuler.net/problem=1" class="embedly-card" data-card-image="0" data-card-controls="0" data-card-align="left"></a>

> If we list all the natural numbers below 10 that are multiples of 3 or 5, we get 3, 5, 6 and 9. The sum of these multiples is 23.
> Find the sum of all the multiples of 3 or 5 below 1000.

意訳）1000**未満**の 3 もしくは 5 の倍数の合計を示せ

`below 10`で 10 未満、以下の場合は`10 or below`と表すらしい。

### Haskell

```haskell
main = do
    print $ sum (filter (\x -> mod x 3 == 0 || mod x 5 == 0) [1..999])
```

### C＃

```csharp
using System;
using System.Linq;

public class Test
{
	public static void Main()
	{
		Console.WriteLine(Enumerable.Range(1, 999).Where(i => i % 3 == 0 || i % 5 == 0).Sum());
	}
}
```

### Python

```python
print(sum([i for i in range(1, 1000) if i % 3 == 0 or i % 5 == 0]))
```

### Ruby

```ruby
puts (1...1000).select { |i| (i % 3).zero? || (i % 5).zero? }.inject(:+)
```

### JavaScript

```js
console.log(
  [...Array(1000).keys()]
    .filter(i => i % 3 == 0 || i % 5 == 0)
    .reduce((c, i) => c + i)
)
```

C#の`Sum()`の圧倒的楽さ、Haskell と Python がスッと入ってこないのは慣れなのかなぁ...
JS は`Range`を用意すべし
