# 既存のメインスレッドで処理している内容をスレッドに分けて実行する

## 前提

* UniRx を使っている人向け
* Unity 2017.1 以前のプロダクトで .NET 2 系を使っている人向け
* スレッドに分けたい処理が Monobehaviour に依存していない

## なぜなら

* UniRx を前提とした実装例を示す
* Unity 2017.1, .NET 4 系を採用できる人は Task async/await や UniTask でも処理の分離が可能なのでそちらを参照
* Monobehaviour はスレッドセーフにできていないため、 Monobehavoiur に依存する処理はスレッドに切り離すことができない

## 本題

### スレッドに分けたいストリーム

```cs
// T は任意の型
public T SomeFunction()
{
    // Monobehaviour に依存しない処理
}

public void Main()
{
    Observable.ReturnUnit()
        .Select(_ => SomeFunction())
        .StrictSubscribe()
        .AddTo(this);
}
```

この処理はメインスレッドで動作する

仮にメインスレッドで動作するか確かめてみる

```cs
public void Main()
{
    Observable.ReturnUnit()
        .Do(_ => UnityEngine.Debug.Log("ThreadId: " + System.Threading.Thread.CurrentThread.ManagedThreadId))
        .Select(_ => SomeFunction())
        .Do(_ => UnityEngine.Debug.Log("ThreadId: " + System.Threading.Thread.CurrentThread.ManagedThreadId))
        .StrictSubscribe()
        .AddTo(this);
}
```

実行ログ

```
ThreadId: 1
ThreadId: 1
```

よってメインスレッドで実行されていることがわかる

### スレッドに処理分けしてみる

```cs
public void Main()
{
    Observable.ReturnUnit()
        .Select(_ => SomeFunction())
        .SubscribeOn(Scheduler.ThreadPool)
        .ObserveOnMainThread()
        .StrictSubscribe()
        .AddTo(this);
}
```

たったこれだけ。

`SubscribeOn(Scheduler.ThreadPool)` に記述されたそれ以前の処理は全てサブスレッドに処理が委譲され、 `ObserveOnMainThread` によってメインスレッドに処理が戻される

メインスレッドに戻す処理まで書いた理由はいくら非同期でも戻り値をメインスレッドに戻したいことが往往にしてあると思ったからである

続いてスレッド状況もみてみよう

```cs
public void Main()
{
    Observable.ReturnUnit()
        .Do(_ => UnityEngine.Debug.Log("ThreadId: " + System.Threading.Thread.CurrentThread.ManagedThreadId))
        .Select(_ => SomeFunction())
        .SubscribeOn(Scheduler.ThreadPool)
        .ObserveOnMainThread()
        .Do(_ => UnityEngine.Debug.Log("ThreadId: " + System.Threading.Thread.CurrentThread.ManagedThreadId))
        .StrictSubscribe()
        .AddTo(this);
}
```

実行スレッド

```
ThreadId: 13
ThreadId: 1
```

ちなみにスレッド ID は順番とは限らない（これは僕が他のプロセスも動かしていたからかもしれないが）
