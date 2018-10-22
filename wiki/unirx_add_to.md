# IDisposable AddTo&lt;T&gt;(this T disposable) について

実装を読むと

```cs
namespace UniRx
{
    public static partial class DisposableExtensions
    {
        // ...(略)...

        /// <summary>Dispose self on target gameObject has been destroyed. Return value is self disposable.</summary>
        public static T AddTo<T>(this T disposable, Component gameObjectComponent)
            where T : IDisposable
        {
            if (gameObjectComponent == null)
            {
                disposable.Dispose();
                return disposable;
            }

            return AddTo(disposable, gameObjectComponent.gameObject);
        }
        // ...(略)...
```

とあるので、 `gameObject.Do(_ => HogeHoge()).Subscribe(_ => FooBar()).AddTo(this)` とした場合には、 `gameObject` が破棄されたときに、 `Subscribe(_ => Foobar())` の戻り値である `Disposable` が登録されているので、破棄される。

その具体を見るために [UniRxのシンプルなサンプル その6(購読の停止)](https://qiita.com/Marimoiro/items/819ddb3e68aab7ee3b95) をみてほしい

```cs
using UnityEngine;
using System.Collections;
using UniRx;
using System;

public class DeadUpdate : Base {

    // Use this for initialization
    void Start () {

        gameObject.transform.position = new Vector2(0, 1f);

        //5秒後にgameObjectが死ぬ
        Observable.Timer(TimeSpan.FromSeconds(5))
           .Subscribe(_ => Destroy(gameObject));

        //0.5秒ごとに0.05右に移動
        Observable.Interval(TimeSpan.FromMilliseconds(500))
            .Subscribe(l => Move(0.5f, 0));
    }

    private void Move(float x, float y)
    {
        gameObject.transform.position = new Vector2(
            gameObject.transform.position.x + x,
            gameObject.transform.position.y + y
        );
    }
}

```

このコードを実行した場合、 5秒後に `gameObject` の破棄と同時に例外を吐く。

なぜなら、 `Destroy(gameObject)` 実行時に `gameObject` は削除されるのだが、 `Subscribe` の戻り値である `IDisposable` を `Dispose()` しないと購読を停止していないことになるので、 `Observer` は依然として `IObservable` （ストリーム）を実行しようとしてしまい、読み出したい `gameObject` はすでに破棄されているので、エラーを吐いてしまうというわけである。

## 「ストリーム」の実態と Subject の削除

ストリームの管理する上で重要なのが、そのストリームは誰の持ち物なのか？ということである。

基本的に、ストリームの実体は「Subject」であり、Subjectが破棄されればストリームも破棄されることになる。

![](../images/subject-and-stream.png)

Subscribe とは Subject 関数を登録する処理である。つまり、ストリームの実体はSubjectが内部に保持する「呼び出す関数リスト(及びその関数に連なるメソッドチェーン)」でであり、Subject がストリームを管理しているということになる。

UniRx ではわかりやすいことに、 Subscribe メソッドの戻り値は Subject (IDisposable) である。

よって、 IDisposable が破棄されれば、それに紐づくストリームは全て破棄される。

そして、その対象となる `IObservable<gameObject>` の gameObject が破棄されたときに、同時に `IDisposable` も破棄されるように bind する処理が `.AddTo(this)` である。

## 引用元

* [UniRxのシンプルなサンプル その6(購読の停止)](https://qiita.com/Marimoiro/items/819ddb3e68aab7ee3b95)
* [UniRx入門 その2 - メッセージの種類/ストリームの寿命](https://qiita.com/toRisouP/items/851087b4c990d87641e6)
