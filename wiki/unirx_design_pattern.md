# UniRx のデザインパターン

UniRx は Observable パターンと呼ばれるデザインパターンを採用している

Observer （観察者）が Subject （通知オブジェクト側）を観察することから名付けられており、出版-購読型モデルとも呼ばれる

用語については [Observer パターン - wiki](https://ja.wikipedia.org/wiki/Observer_%E3%83%91%E3%82%BF%E3%83%BC%E3%83%B3) を参考にしてほしい

[作りながら理解するUniRx](https://qiita.com/mattak/items/106dfd0974653aa06fbc) を書き換えながら理解を深める

## 1. Pull 型 Observer パターン

```cs
// 監視者 (寿司を食べる人)
public class SushiObserver : IObserver
{
    public SushiChefSubject Subject;

    // 寿司が来た
    public void OnNotified()
    {
        Debug.LogFormat("{0}おいしいです (^q^)", this.Subject.MakedNeta);
    }
}

// 監視対象 (寿司職人)
public class SushiChefSubject : ISubject
{
    // 握った寿司
    public string MakedNeta;

    // お客さん
    private List<IObserver> observers = new List<IObserver>();

    // 寿司を作ったことを通知
    public void NotifyObservers()
    {
        foreach (var observer in this.observers)
        {
            observer.OnNotified();
        }
    }

    // お客さんを監視する (イベントを購読する)
    public void Subscribe(IObserver observer)
    {
        this.observers.Add(observer);
    }

    // お客さんの監視をやめる (イベントを非購読する)
    public void Unsubscribe(IObserver observer)
    {
        this.observers.Remove(observer);
    }
}
```

実行結果

```cs
var chef = new SushiChefSubject();
var customer1 = new SushiObserver() {Subject = chef};
var customer2= new SushiObserver() {Subject = chef};

chef.Subscribe(customer1);
chef.Subscribe(customer2);

chef.MakedNeta = "まぐろ";
chef.NotifyObservers();

// output:
// まぐろおいしいです (^q^)
// まぐろおいしいです (^q^)
```

この実装の欠点

* 寿司が来たということはわかるが、具体的にどんなネタを作ったかがわからない
* 通知された人が通知した人 (Subject) にデータを参照しにいかなければならない
* よって、密結合になり監視者側が監視対象を意識しなくてはならないので拡張性に欠ける

## 2. Push 型 Observer パターン

Pull 型の欠点を解決したデザインパターン。

監視者が監視対象を参照せず、データのみ受け取るようなInterface

```cs
// 監視者 (寿司を食べる人)
public class SushiObserver : IObserver<string>
{
    public void OnNotified(string neta)
    {
        Debug.LogFormat("{0}おいしいです (^q^)", neta);
    }
}

// 監視対象 (寿司職人)
public class SushiChefSubject : ISubject<string>
{
    // お客さん
    private List<IObserver<string>> observers = new List<IObserver<string>>();

    public void NotifyObservers(string neta)
    {
        foreach (var observer in this.observers)
        {
            observer.OnNotified(neta);
        }
    }

    // お客さんを監視する (イベントを購読する)
    public void Subscribe(IObserver<string> observer)
    {
        this.observers.Add(observer);
    }

    // お客さんの監視をやめる (イベントを非購読する)
    public void Unsubscribe(IObserver<string> observer)
    {
        this.observers.Remove(observer);
    }
}

```

実行結果

```cs
var chef = new SushiShokuninSubject();
var customer1 = new SushiObserver();
var customer2 = new SushiObserver();

chef.Subscribe(customer1);
chef.Subscribe(customer2);

chef.NotifyObservers("サーモン");

// output:
// サーモンおいしいです (^q^)
// サーモンおいしいです (^q^)
```

IObserverと ISubjectの依存（SushiObserver が Subject.MakedNeta を知らないといけないという密結合）を分離することができた

## 3. OnNext, OnComplete, OnError

```cs
// ちょっと一般化して、定義順を Subject → Observer に変えます
public class Subject<TNext> : ISubject<TNext>
{
    private List<IObserver<TNext>> observers = new List<IObserver<TNext>>();

    // お客さんに次のネタを作ったことを知らせる
    public void NotifyNext(TNext next)
    {
        foreach (var observer in this.observers)
        {
            observer.OnNext(next);
        }
    }

    // お客さんにエラーを知らせる
    public void NotifyError(Exception error)
    {
        foreach (var observer in this.observers)
        {
            observer.OnError(error);
        }
    }

    // お客さんが食べ終わった状態を見る
    public void NotifyComplete()
    {
        foreach (var observer in this.observers)
        {
            observer.OnComplete();
        }
    }

    // お客さんを監視する
    public void Subscribe(IObserver<TNext> observer)
    {
        this.observers.Add(observer);
    }

    // お客さんの監視をやめる
    public void Unsubscribe(IObserver<TNext> observer)
    {
        this.observers.Remove(observer);
    }
}

public class SushiChefSubject : Subject<string>
{
}

public class SushiObserver : IObserver<string>
{
    public void OnNext(string neta)
    {
        Debug.LogFormat("{0}おいしいです (^q^)", neta);
    }

    public void OnError(Exception exception)
    {
        Debug.LogFormat("{0} (￣Д￣;)", exception.Message);
    }

    public void OnComplete()
    {
        Debug.Log("ごちそうさま (^O^)");
    }
}
```

実行結果

```cs
var chef = new SushiShokuninSubject();
var customer1 = new SushiObserver();
var customer2 = new SushiObserver();

chef.Subscribe(customer1);
chef.NotifyNext("ほたて");
chef.NotifyComplete();
chef.Unsubscribe(customer1);

chef.Subscribe(customer2);
chef.NotifyError(new Exception("ほたてはもうありません"));
chef.Unsubscribe(customer2);

// output:
// ほたておいしいです (^q^)
// ごちそうさま (^O^)
// ほたてはもうありません (￣Д￣;)
```

かなり Rx に近づいて来たがこの実装にも以下の課題がある

* 監視をやめたい（Unsubscribe）だけなのに、ISubjectのclassを参照しなくてはならないのは拡張性にかける

## 4. Disposable

課題を解決するために C# には IDisposableというリソース開放のための interfaceが備わっているので、これを用いて購読停止をする

IDisposable

* https://msdn.microsoft.com/ja-jp/library/system.idisposable(v=vs.110).aspx

IDisposableを具象化した購読管理クラスをSubscriptionとして実装した結果が以下の通り。

```cs
// 監視対象
public interface ISubject<TValue>
{
    // データを通知
    void NotifyNext(TValue value);

    // エラーを通知
    void NotifyError(Exception error);

    // データ終了を通知
    void NotifyComplete();

    // 監視者が購読する
    IDisposable Subscribe(IObserver<TValue> observer);
}

public class Subject<TNext> : ISubject<TNext>
{
    private List<IObserver<TNext>> observers = new List<IObserver<TNext>>();

    public void NotifyNext(TNext next)
    {
        foreach (var observer in this.observers)
        {
            observer.OnNext(next);
        }
    }

    public void NotifyError(Exception error)
    {
        foreach (var observer in this.observers)
        {
            observer.OnError(error);
        }

        this.observers.Clear();
    }

    public void NotifyComplete()
    {
        foreach (var observer in this.observers)
        {
            observer.OnComplete();
        }

        this.observers.Clear();
    }

    public IDisposable Subscribe(IObserver<TNext> observer)
    {
        this.observers.Add(observer);
        // 購読管理のclassを返す
        return new Subscription(this, observer);
    }

    private void Unsubscribe(IObserver<TNext> observer)
    {
        this.observers.Remove(observer);
    }

    // 購読管理をするclass. Dispose()を呼ぶことで購読をやめる
    class Subscription : IDisposable
    {
        private IObserver<TNext> observer;
        private Subject<TNext> subject;

        public Subscription(Subject<TNext> subject, IObserver<TNext> observer)
        {
            this.subject = subject;
            this.observer = observer;
        }

        public void Dispose()
        {
            this.subject.Unsubscribe(this.observer);
        }
    }
}
```

実行結果

```cs
var subject = new SushiChefSubject();
var observer = new SushiObserver();

var disposable = subject.Subscribe(observer);
subject.NotifyNext("えんがわ");

// 購読停止（お客さんを見るのをやめる）
// Dispose() の対象は disposable を instansiate したときの observer
//
disposable.Dispose();

// observer に通知されない（お客さんを無視して作ってるので作ったけどお客さんの手元には給仕されない）
subject.NotifyNext("いか");

// output:
// えんがわおいしいです (^q^)
```

## 5. Observable

Dispose 実装時の以下の部分をみてください

```cs
public class Subject<TNext> : ISubject<TNext>
{
    private List<IObserver<TNext>> observers = new List<IObserver<TNext>>();

    public void NotifyNext(TNext next)
    {
        foreach (var observer in this.observers)
        {
            observer.OnNext(next);
        }
    }
    // ...(略)...
}
```

これより

ISubjectの

* NotifyNext(value)
* NotifyError(error)
* NotifyComplete()

は、interfaceの関数の入出力が IObserver の

* OnNext(value)
* OnError(error)
* OnComplete()

と1対1であることがわかる。

監視対象の ISubject はイベントの実行タイミングで考えると、IObserver と同等と言える。

つまり、Subjectは 監視者(IObserver) であり監視対象 (IObservable) であるとみなせる。

よって、先程定義した ISubjectの `IDisposable Subscribe(IObserver<TValue> observer)` を、監視可能なもの（IObservable）の interface として外出しすると以下のようになる。

```cs
// 監視者
public interface IObserver<TValue>
{
    // データがきた
    void OnNext(TValue value);

    // エラーが起きた
    void OnError(Exception error);

    // データはもう来ない
    void OnComplete();
}

// 監視可能であることを示す
public interface IObservable<TValue>
{
    // 監視者が購読する
    IDisposable Subscribe(IObserver<TValue> observer);
}

// 監視対象
public interface ISubject<TValue> : IObserver<TValue>, IObservable<TValue>
{
    // 3つのイベントは IObserverと同一のinterfaceなので委譲した
    // データを通知: NofityNext(value) => OnNext(value)
    // エラーを通知: NotifyError(error) => OnError(error)
    // データ終了を通知: NotifyComplete() => OnComplete()

    // 監視可能であることは IObservalbeに委譲した
    // IDisposable Subscribe(IObserver<TValue> observer);
}
```

ISubject は、IObserver（監視者）であり、IObservable（監視可能）であることが明示された。

Rx の文脈では、Subject のようにすぐに来たイベントを流してしまうような IObservable のことを Hot であると呼称している。

## 6. Cold Observable

実は先程までの寿司屋では釣ったばかりのネタをそのまま捌いて提供している（HotObservable）。

しかし、お客さんが来た時に都合よく釣ったばかりの魚があるとは限らない。

釣った魚を冷凍しておいて、お客さんが来たときに解凍して提供してあげればより利便性が上がることがわかる。

```cs
// 講読時に監視するためのclass
public class Observable<TValue> : IObservable<TValue>
{
    private Func<IObserver<TValue>, IDisposable> creator;

    private Observable(Func<IObserver<TValue>, IDisposable> creator)
    {
        this.creator = creator;
    }

    public IDisposable Subscribe(IObserver<TValue> observer)
    {
        // Subscribeした瞬間に関数を実行するのが特徴
        return this.creator(observer);
    }

    // Observableを直接渡したくないため、Createメソッドを作っておく.
    public static IObservable<TValue> Create(Func<IObserver<TValue>, IDisposable> creator)
    {
        return new Observable<TValue>(creator);
    }
}

// 購読解除するつもりがないときに返す Disposable
public class EmptyDisposable : IDisposable
{
    public void Dispose()
    {
    }
}
```

このように監視をSubscribe時に開始するclassをつくることで、ネタの鮮度を維持しつつ、客が来たときに振る舞うことが可能になる。

実行結果

```cs
// 冷凍寿司
var observable = Observable<string>.Create(_observer =>
{
    // ネタが解凍できたら寿司を握って提供する
    Debug.Log("ネタを解凍します");
    _observer.OnNext("ぶり");
    _observer.OnComplete();
    return new EmptyDisposable();
});

var observer = new SushiObserver();
Debug.Log("冷凍されたぶりが届きました！");
observable.Subscribe(observer);

// output:
// 冷凍されたぶりが届きました！
// ネタを解凍します
// ぶりおいしいです (^q^)
// ごちそうさま (^O^)
```

実行結果を見てみると、監視するときに処理内容を関数として保持しておくことで、講読（Subscribe）が実行されるまで内部の処理は実行されないことが確認できる。

Rx の文脈では、このような Observable を Cold であるという。

Hot Observableに対して、Cold Observableは即時実行でない点が特徴であり、Rxの非同期な利用を許す上で重要な概念になっている。

## 7. Operator: Where

ここから先、寿司屋から回転寿司に置き換えた方が説明しやすいので置き換えて説明する。

現状だと寿司屋の提供してものは全てお客さんが食べる前提でしたが、今後は回転寿司とすることで自由に選べるようにする。

既存実装がどうなるのかについては [作りながら理解するUniRx](https://qiita.com/mattak/items/106dfd0974653aa06fbc)  を参考にしてください。

その問題を解決する概念が Operator の存在で、今回は Where を例に挙げます。

```cs
// イベント関数をそのまま実行するだけの Observer
public class Observer<TNext> : IObserver<TNext>
{
    private Action<TNext> next;
    private Action<Exception> error;
    private Action complete;

    private Observer(Action<TNext> next, Action<Exception> error, Action complete)
    {
        this.next = next;
        this.error = error;
        this.complete = complete;
    }

    public void OnNext(TNext value)
    {
        this.next(value);
    }

    public void OnError(Exception error)
    {
        this.error(error);
    }

    public void OnComplete()
    {
        this.complete();
    }

    public static IObserver<TNext> Create(Action<TNext> next, Action<Exception> error, Action complete)
    {
        return new Observer<TNext>(next, error, complete);
    }
}

// Nextの値によって通知するかしないかを変更する
public class WhereObservable<TNext> : IObservable<TNext>
{
    private Func<TNext, bool> operation;
    private IObservable<TNext> observable;

    public WhereObservable(IObservable<TNext> observable, Func<TNext, bool> operation)
    {
        this.operation = operation;
        this.observable = observable;
    }

    public IDisposable Subscribe(IObserver<TNext> observer)
    {
        var disposable = this.observable.Subscribe(Observer<TNext>.Create(
            next =>
            {
                // nextの値によって、次に処理を流すかどうかを決定. operationはboolを返却する
                if (this.operation(next)) observer.OnNext(next);
            },
            error => observer.OnError(error),
            () => observer.OnComplete()
        ));

        return disposable;
    }
}

// IObserverをメソッドチェインするためのExtension
public static class IObservableExtension
{
    // 条件をしぼるObservable
    public static IObservable<TNext> Where<TNext>(this IObservable<TNext> observable, Func<TNext, bool> operation)
    {
        return new WhereObservable<TNext>(observable, operation);
    }
}
```

ポイントは、IObservable の拡張関数として新しい Observable を返却するように定義している点で、 `observable.Where(...).Where(...).Where(...).Subscribe()` という形で関数を鎖でつなげていくように、IObservable から新しい IObservable へ変換できる。

実行結果

```cs
var subject = new SushiLaneSubject();
var observer = new SushiObserver();

// マグロだけ食べたい
subject
    .Where(neta => neta == "まぐろ")
    .Subscribe(observer);

subject.OnNext("すずき");
subject.OnNext("まぐろ");
subject.OnNext("たこ");
subject.OnNext("まぐろ");
subject.OnComplete();

// output:
// まぐろおいしいです (^q^)
// まぐろおいしいです (^q^)
// ごちそうさま (^O^)
```

他にも Operator は存在するが他はいずれもここまでの複合形のため一旦割愛する。

他の Select / SelectMany について知りたければ [作りながら理解するUniRx](https://qiita.com/mattak/items/106dfd0974653aa06fbAc) を参考にしてもらうと良い。

## 8. Unity での利用

Rx の基本的な仕組みは上記で説明できた。

これを Unity に適応したことを考える。

まずはボタンのクリックイベントを監視可能にすることを考える。

```cs
// 何もないことを示すイベントのデータclass
public class Unit
{
    public static readonly Unit Default = new Unit();

    private Unit()
    {
    }
}

// ボタンが押されたら、イベントを通知する Observable
public class ButtonObservable : IObservable<Unit>
{
    private Button button;

    public ButtonObservable(Button button)
    {
        this.button = button;
    }

    public IDisposable Subscribe(IObserver<Unit> observer)
    {
        this.button.onClick.AddListener(() =>
        {
            observer.OnNext(Unit.Default);
        });

        // 本当は講読管理をしっかり考えなければならない
        return new EmptyDisposable();
    }
}

public static class ButtonExtension
{
    // ボタンのOnClickを受けて、イベントを流す拡張関数
    public static IObservable<Unit> OnClickAsObservable(this Button button)
    {
        return new ButtonObservable(button);
    }
}
```

ボタンが押されたら OnNext を流すだけの Observable の実装が上記。

Unit は UniRx でよく利用されるデータで、イベントのタイミングだけ知らせたいや通知するデータが何もないときに利用するclass を指す。

また、Subscribe に IObserver の class をいちいち実装するのが少し面倒だったので、便利になるように event 関数のみを引数に取れるようにしたのが以下。

```cs
// IObserverをメソッドチェインするためのExtension
public static class IObservableExtension
{
    // ...

    // 面倒なので IObserverクラスをいちいち作らなくて良いようにする.
    public static IDisposable Subscribe<TNext>(
        this IObservable<TNext> observable,
        Action<TNext> next,
        Action<Exception> error,
        Action complete)
    {
        return observable.Subscribe(Observer<TNext>.Create(next, error, complete));
    }

    // next, errorのみでもOK
    public static IDisposable Subscribe<TNext>(
        this IObservable<TNext> observable,
        Action<TNext> next,
        Action<Exception> error)
    {
        return observable.Subscribe(next, error, () => { });
    }

    // nextのみでもOK
    public static IDisposable Subscribe<TNext>(
        this IObservable<TNext> observable,
        Action<TNext> next)
    {
        return observable.Subscribe(next, error => { }, () => { });
    }
}
```

利用例

```cs
using UnityEngine;
using UnityEngine.UI;

[RequireComponent(typeof(Button))]
public class ButtonExample : MonoBehaviour
{
    void Start()
    {
        var button = this.GetComponent<Button>();

        // ボタンがおされたら、ログを流す
        button.OnClickAsObservable()
            .Subscribe(_ => Debug.Log("Clicked!"));
    }
}
```

加えて、ThrottleFirstという一定時間はNextを流さないoperatorを定義しておけば、ボタン連打防止なんかもできる

```cs
button.OnClickAsObservable().ThrottleFirst(TimeSpan.FromSeconds(1)).Subscribe(...);
```

他の例として、 MonoBehaviour の Update が起きたら Next を流す処理を考える

```cs
public class EveryUpdateObservable : IObservable<Unit>
{
    private Component component;

    public EveryUpdateObservable(Component component)
    {
        this.component = component;
    }

    public IDisposable Subscribe(IObserver<Unit> observer)
    {
        var updator = this.component.GetComponent<EveryUpdateComponent>();

        if (updator == null)
        {
            updator = this.component.gameObject.AddComponent<EveryUpdateComponent>();
        }

        updator.Observer = observer;
        return updator;
    }
}

public class EveryUpdateComponent : MonoBehaviour, IDisposable
{
    public IObserver<Unit> Observer { get; set; }

    private void Update()
    {
        // 毎frameでイベント送信する
        if (this.Observer != null)
        {
            this.Observer.OnNext(Unit.Default);
        }
    }

    private void OnDestroy()
    {
        this.Dispose();
    }

    public void Dispose()
    {
        if (this.Observer != null)
        {
            this.Observer.OnComplete();
        }
    }
}

public static class ComponentExtension
{
    // フレームごとにイベント送信する
    public static IObservable<Unit> OnEveryUpdate(this Component component)
    {
        return new EveryUpdateObservable(component);
    }
}
```

このように Update の監視を Rx で実装することができる。

```cs
public class EveryUpdateExample : MonoBehaviour
{
    void Start()
    {
        // 10フレームごとにログを吐く
        this.OnEveryUpdate()
            .Select(_ => Time.frameCount)
            .Where(frame => frame % 10 == 0)
            .Subscribe(frame => Debug.LogFormat("{0} frame passed", frame));
    }
}
```

実際の UniRx では、他にも Unity のライフサイクルに合わせて購読停止してくれる `IDisposable.AddTo(component)` 関数や、衝突したときのイベントを取ってくれる Observable など Unity ならではの Rx 拡張が多々用意されている。

## 引用元

* [作りながら理解するUniRx](https://qiita.com/mattak/items/106dfd0974653aa06fbc)
* [Observer パターン - wiki](https://ja.wikipedia.org/wiki/Observer_%E3%83%91%E3%82%BF%E3%83%BC%E3%83%B3)
* [【JSでデザインパターン】オブザーバ編](https://qiita.com/kenju/items/3cee3a1907fbd71b7b76)
