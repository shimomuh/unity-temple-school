# 概要

サクッと手元のシェルで C# コードをコンパイル・実行して挙動を確認できるようにする

# 手順

## Mono をインストールする

```bash
brew install mono
```

## コードを書く

```cs
using System;
public class HelloWorld
{
    static public void Main ()
    {
        Console.WriteLine ("Hello Mono World");
    }
}
```

ファイル名を hello.cs で保存する

## コンパイルと実行

```bash
mcs hello.cs
# => hello.exe が生成される

mono hello.exe
```

以上！
