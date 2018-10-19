# static とは？

* int 変数 = メンバ変数
* int static 変数 = クラス変数

上記でほとんど説明は終わっているのだが、 Ruby になれてしまった自分自身が理解しやすいためにコードで説明する。

## ruby

```ruby
class User
  @@num = 0

  def initialize (name)
    # 非 static 変数（インスタンス変数）
    @name = name
    # static 変数（クラス変数）
    @@num += 1
  end

  # 非 static メソッド（インスタンスメソッド）
  def calledName
    @name
  end

  # static メソッド（クラスメソッド）
  def self.count
    @@num
  end
end

user1 = User.new('taro')
p user1.name
p User.count
user2 = User.new('hanako')
p user2.name
p User.count
```

実行結果

```text
taro
1
hanako
2
```

## C#

```cs
using System;

public class User
{
    string name;
    static int num;

    static User ()
    {
        num = 0;
    }

    public User (string name)
    {
        this.name = name;
        num++;
    }

    public string getName ()
    {
        return name;
    }

    public static int count ()
    {
        return num;
    }
}

public class Program
{
    public static void Main ()
    {
        var user1 = new User("taro");
        Console.WriteLine(user1.getName());
        Console.WriteLine(User.count());
        var user2 = new User("hanako");
        Console.WriteLine(user2.getName());
        Console.WriteLine(User.count());
    }
}
```

実行結果

```text
taro
1
hanako
2
```
