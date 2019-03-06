# UniRx 動作確認リファレンス

## Observable.Return(T)

T は次のストリームに流れる値

Hot Observable

つまり、Subscribe が実行された時点で呼び出される

### Subscribe しない場合
#### 実行サンプル
```cs
void Awake()
{
    Observable.Return(Unit.Default)
        .Do(_ => Debug.logger.Log("do!"));
}
```

#### 実行結果
```
なにもでない
```

### Subscribe した場合
#### 実行サンプル
```cs
void Awake()
{
    Observable.Return(Unit.Default)
        .Do(_ => Debug.logger.Log("do!"))
        .Subscribe(_ => Debug.logger.Log("subscribe!"));
}
```

#### 実行結果
```
do!
subscribe!
```

### T を変えた場合
#### 実行サンプル
```cs
void Awake()
{
    Observable.Return(1)
        .Subscribe(value => Debug.logger.Log(value));
}
```

#### 実行結果
```
1
```
