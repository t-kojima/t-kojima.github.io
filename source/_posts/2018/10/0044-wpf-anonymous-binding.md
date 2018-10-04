---
title: '[WPF] Bindingに匿名クラスを使う'
date: 2018-10-04 10:09:53
tags:
  - csharp
---

WPF の双方向バインディング、便利ですよね。ただ実際使っていると双方向が必要なケースってあまりなくて、大部分は単一方向で済む場合が多いはず。

この時プロパティには`getter`だけあればいいんだけど、匿名クラスを返せると表現力が上がるね！という話。

<!-- more -->

# 基本的なバインディングの使い方

ほんとにベーシックかどうかはわからないけど、僕は基本だと思っている使い方。

TextBlock にファイルの名前を表示させる。

## 双方向バインディングで表示

まずは双方向バインディングで表示する例

#### MainWindow.xaml

```xml
<TextBlock Text="{Binding File.Name}">
```

#### ViewModel.cs

```cs
public class ViewModel : INotifyPropertyChanged {
    public event PropertyChangedEventHandler PropertyChanged;

　　// FileにFileInfoを詰めるなんらかの処理

    private FileInfo file;
    public FileInfo File {
        get { return file; }
        set {
            if (file == value)
                return;
            file = value;
            PropertyChanged?.Invoke(this, new PropertyChangedEventArgs("File"));
        }
    }
}
```

特に問題ないと思うが、`File` が Public である必要がある。時には private な `file` しかない場合で、`file.Name` を表示したい場合もあるだろう。そういう場合は次のように getter のみを作って公開する。

## 単一方向バインディング

#### MainWindow.xaml

```xml
<TextBlock Text="{Binding File.Name}">
```

#### ViewModel.cs

```cs
public class ViewModel : INotifyPropertyChanged {
    public event PropertyChangedEventHandler PropertyChanged;

　　// fileにFileInfoを詰めるなんらかの処理
    // 同時に PropertyChanged?.Invoke(this, new PropertyChangedEventArgs("File"));を呼ぶ必要がある。

    private FileInfo file;

    public string File {
        get { return file; }
    }
}
```

上記のように File プロパティとして getter のみを実装し、`file`を返している。表示のみの場合はこれで十分だが、`file`に FileInfo を設定する際、`PropertyChanged?.Invoke(this, new PropertyChangedEventArgs("File"));`を実行する必要はある。（しないと View に変更が通知されない）

## 匿名クラスを返す

前述の単一方向バインディングの例だが、匿名クラスを返すことでも実装することができる。

#### MainWindow.xaml

```xml
<TextBlock Text="{Binding File.Name}">
```

#### ViewModel.cs

```cs
public class ViewModel : INotifyPropertyChanged {
    public event PropertyChangedEventHandler PropertyChanged;

　　// fileにFileInfoを詰めるなんらかの処理
    // 同時に PropertyChanged?.Invoke(this, new PropertyChangedEventArgs("File"));を呼ぶ必要がある。

    private FileInfo file;

    public object File {
        get { return new { Name = file.Name }; }
    }
}
```

ポイントはプロパティの型を object とする点と、文字通り匿名クラスを返す点だ。

こうすることで、バインディング対象が FileInfo のインスタンスから、`Name`プロパティのみを持つ匿名クラスのインスタンスに変わった。

これだけだと特になんてことない例になってしまうので、具体的な使い方を見てみたい。

# 表示内容を動的に組み立てる例

とあるディレクトリに、TXT ファイルと JPEG ファイルがいくつか入っているとしよう。

まずは FileInfo のリストとして取得する。

```cs
var fileInfoList = new DirectoryInfo("<とあるディレクトリのパス>").GetFiles();
```

さて、このリストに TXT ファイルと JPEG ファイルがそれぞれ何個含まれているか、ListView に表示したいとする。以下のような形だ。

| 形式 | 個数 |
| ---- | ---- |
| TXT  | 12   |
| JPEG | 8    |

ちなみに XAML はこんな感じになる。匿名クラスを使っても使わなくても同じ

```xml
<ListView ItemsSource="{Binding Summaries}">
  <ListView.View>
    <GridView>
      <GridViewColumn Header="形式" DisplayMemberBinding="{Binding Type}" />
      <GridViewColumn Header="個数" DisplayMemberBinding="{Binding Count}" />
    </GridView>
  </ListView.View>
</ListView>
```

## 匿名クラスを使わない場合

ListView にバインドするコレクションの要素として、クラスか構造体を定義する必要がある。

```cs
public struct Item {
    public string Type { get; set; }
    public int Count { get; set; }
}

public IEnumerable<Item> Summaries {
    get {
        return fileInfoList.GroupBy(f => f.Extension)
            .Select(g => new Item() { Type = g.Key.TrimStart('.').ToUpper(), Count = g.Count() });
    }
}
```

## 匿名クラスを使う場合

対して匿名クラスを使う場合は、コレクションの要素となるクラスや構造体を用意する必要がない。

```cs
public IEnumerable<object> Summaries {
    get {
        return fileInfoList.GroupBy(f => f.Extension)
            .Select(g => new { Type = g.Key.TrimStart('.').ToUpper(), Count = g.Count() });
    }
}
```

こんな風に書いてもいい。こっちだと`Type`に自由な文字を入れることができる。

```cs
public IEnumerable<object> Summaries {
    get {
        yield return new { Type = "TXT", Count = fileInfoList.Count(f => f.Extension == '.txt') };
        yield return new { Type = "JPEG", Count = fileInfoList.Count(f => f.Extension == '.jpg') };
    }
}
```

# まとめ

表示のみに使うクラス（構造体）をわざわざ定義する必要はなく、匿名クラスで記述すると簡易に表現することができた。

# 環境

- Windows 10
- .NetFramework 4.0
