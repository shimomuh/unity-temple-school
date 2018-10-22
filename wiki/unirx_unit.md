# IObservable.Return(Unit.Default) を紐解く

## Unit とは何か

[いまさら聞けないReactive Extensions.3](https://qiita.com/Temarin/items/107fa2e39f427e591dbd) によると、Unit は特にデータや戻り値が無いことを表す関数で、 void と同じ役割を果たす。

```cs
Unit Function( int x ){
   return Unit.Default;
}
```

> `IObservable<T>` を基準として考える

という説明がある通り、 UniRx では `Observable<T>` （ストリーム）を扱うことが一般的であり、 T として処理したいものがない場合は `IObservable<Unit>` を返すとおぼえておけば差し支えない。

## Unit.Default を紐解く

Unit = JS でいう void function () {} であるということは理解できた。

その上で、 Unit.Default が何かをみてみる。

UniRx の現在の実装は以下の通り。

```cs
// from Rx Official, but convert struct to class(for iOS AOT issue)

using System;

namespace UniRx
{
    [Serializable]
    public struct Unit : IEquatable<Unit>
    {
        static readonly Unit @default = new Unit();

        public static Unit Default { get { return @default; } }

        public static bool operator ==(Unit first, Unit second)
        {
            return true;
        }

        public static bool operator !=(Unit first, Unit second)
        {
            return false;
        }

        public bool Equals(Unit other)
        {
            return true;
        }
        public override bool Equals(object obj)
        {
            return obj is Unit;
        }

        public override int GetHashCode()
        {
            return 0;
        }

        public override string ToString()
        {
            return "()";
        }
    }
}
```

Unit クラスは IEquatable を継承しているので、 IEquatable を見てみると

```cs
namespace System
{
  public interface IEquatable<T>
  {
    bool Equals(T other);
  }
}
```

System のインターフェースなので、かなり多くのクラスが利用しているインターフェースである。

少し横道にそれるが、このクラスが持つ `bool Equals(T other)` の挙動を見て見る

```cs
using System;

class Program
{
    public static void Main ()
    {
        var a = new { A = 516 };
        var b = new { A = 516 };
        Console.WriteLine(a == b); // 参照が等価か
        Console.WriteLine(a.Equals(b)); // 値が等価か
        Console.WriteLine(a.A); // おまけ
    }
}

// => False
// => True
// => 516
```

ちなみに、ここで使われている `new { A = 516 }` というのは匿名型（Annonymous Type）と呼ばれるクラスで、引数として指定したものを変数として取り込んで小さなクラスを定義できる。

Unit クラスはこのインターフェースを受け継いでいることに加えて、シングルトンである実装がされているため、参照すら等価となる（void を意味するオブジェクトなので、複数インスタンスを生成する必要性がないため）

```cs
using System;

namespace UniRx
{
    [Serializable]
    public struct Unit : IEquatable<Unit>
    {
        static readonly Unit @default = new Unit();

        public static Unit Default { get { return @default; } }

        public static bool operator ==(Unit first, Unit second)
        {
            return true;
        }

        public static bool operator !=(Unit first, Unit second)
        {
            return false;
        }

        public bool Equals(Unit other)
        {
            return true;
        }
        public override bool Equals(object obj)
        {
            return obj is Unit;
        }

        public override int GetHashCode()
        {
            return 0;
        }

        public override string ToString()
        {
            return "()";
        }
        public static void Main () {
            var a = new Unit();
            var b = new Unit();
            var c = Unit.Default;
            Console.WriteLine(a == b);
            Console.WriteLine(a.Equals(b));
            Console.WriteLine(a == c);
            Console.WriteLine(a.Equals(c));
        }
    }
}

// => True
// => True
// => True
// => True
```

### Observable.Return

コードを覗くと

```cs
        /// <summary>
        /// Return single sequence Immediately.
        /// </summary>
        public static IObservable<T> Return<T>(T value)
        {
            return Return<T>(value, Scheduler.DefaultSchedulers.ConstantTimeOperations);
        }

        /// <summary>
        /// Return single sequence on specified scheduler.
        /// </summary>
        public static IObservable<T> Return<T>(T value, IScheduler scheduler)
        {
            return new ReturnObservable<T>(value, scheduler);
        }
```

となっており、`Scheduler.DefaultSchedulers.ConstantTimeOperations` は以下であることから、 constantTime が実行時に定義されていない限り、ただちに実行されることがわかる

```cs
namespace UniRx
{
    // Scheduler Extension
    public static partial class Scheduler
    {
        // configurable defaults
        public static class DefaultSchedulers
        {
            static IScheduler constantTime;
            public static IScheduler ConstantTimeOperations
            {
                get
                {
                    return constantTime ?? (constantTime = Scheduler.Immediate);
                }
                set
                {
                    constantTime = value;
                }
            }
            // ...(略)...
```

続いて、 `ReturnObservable<T>(value, scheduler)` については、オリジナルは以下のようになっている。

```cs
namespace UniRx.Operators
{
    internal class ReturnObservable<T> : OperatorObservableBase<T>
    {
        readonly T value;
        readonly IScheduler scheduler;

        public ReturnObservable(T value, IScheduler scheduler)
            : base(scheduler == Scheduler.CurrentThread)
        {
            this.value = value;
            this.scheduler = scheduler;
        }
        // ...(略)...
```

よって、`UniRx.Operators` であることがわかる。

戻り値からもネームスペースからもわかるように Operator なので、 IObservable を返す。

IObservable 内で扱われる対象の value は引数の type に依存する。

結果として `Observable.Return(Unit.Default)` は `Observable<Unit>` を返すことがわかる

## 引用元

* [UniRx入門 その2 - メッセージの種類/ストリームの寿命](https://qiita.com/toRisouP/items/851087b4c990d87641e6)
* [いまさら聞けないReactive Extensions.3](https://qiita.com/Temarin/items/107fa2e39f427e591dbd)
