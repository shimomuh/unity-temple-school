# Generic という仕組み

`IOBservable<Unit>` の `<Unit>`。

データ型が異なるだけの同じようなソースコードを１つで書けるようにする仕組みのことだが、コードを元に説明していく。

## Generic を使わない場合

値を入れ替える Swap というメソッドを作ろうとする。

```cs
public static void Swap(ref int a, ref int b)
{
  int c;
  c = a;
  a = b;
  b = c;
}
```

この実装の欠点は、double だったり、float だったりすると同じ関数を何個も書かないといけない。

この課題を解決するのが Generic という仕組みである。

```cs
public static void Swap<T>(ref T a, ref T b)
{
  T c;
  c = a;
  a = b;
  b = c;
}
```

メソッドの後に T と書くことで、利用時に型を指定すればその型で処理を行ってくれる

```cs
double a = 1.0;
double b = 1.0;
Swap<double>(ref a, ref b);
```

ちなみに、引数の型から型推論してくれるので `<double>` は省略することができる

```cs
double a = 1.0;
double b = 1.0;
Swap(ref a, ref b);
```

## では、&lt;Unit&gt; のように指定がある場合は何か？

[ジェネリック - C# によるプログラミング入門 | ++C++; // 未確認飛行 C](http://ufcpp.net/study/csharp/sp2_generics.html) をみるにまず、 `<T>` の T を理解する必要がある。

T は Type Parameter の先頭文字で「型引数」と呼ばれる。

この値は自由に命名することができるので、`<Unit>` としているのは関数内で明示的に型がわかりやすいようにひょうげんするためで、この限りでなくても呼び出し側で `<Unit>` と指定すれば Unit オブジェクトとして扱うことができる

```cs
Unit u1 = new Unit();
Unit u2 = new Unit();
// もちろん型推論できるため、省略可能である
Swap<Unit>(ref u1, ref u2);
```

## 補足

複数 Generic を指定したり制約の指定をすることもできる

```cs
using System;
using System.Collections.Generic;

class X<TItem, TList>
    where TItem : class, IEquatable<TItem>, new()
    where TList : struct, IList<TItem>
{
}
```

## 引用

* [C#のジェネリックを使おう](https://araramistudio.jimdo.com/2017/12/26/c-%E3%81%AE%E3%82%B8%E3%82%A7%E3%83%8D%E3%83%AA%E3%83%83%E3%82%AF%E3%82%92%E4%BD%BF%E3%81%8A%E3%81%86/)
* [ジェネリック - C# によるプログラミング入門 | ++C++; // 未確認飛行 C](http://ufcpp.net/study/csharp/sp2_generics.html)
