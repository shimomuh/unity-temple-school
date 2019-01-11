# Async / Await

## 前提

[さては非同期だなオメー！ async / await 完全に理解しよう](https://www.slideshare.net/UnityTechnologiesJapan/unite-tokyo-2018asyncawait) を理解すること
　
## 2018/05 Unity 2018.1 の登場

厳密には、 Unity 2018.1 が登場したことがすごいのではなく、 Unity 2018.1 によって .NET 4.x が利用できることと C# が 4.0 -> 6.0 になったことが重要。

特に重要なのは [さては非同期だなオメー！ async / await 完全に理解しよう](https://www.slideshare.net/UnityTechnologiesJapan/unite-tokyo-2018asyncawait) p59〜

```cs
IEnumerator Coroutine()
{
    UnityWebRequest request = UnityWebRequest.Get("https://api.etherscan.io/api? module=proxy&action=eth_getBlockByNumber&tag=0x517df3&boolean=true&apikey=YourApiKeyToken");
    yield return request.SendWebRequest();

    var text = request.downloadHandler.text;
    var data = (JObject)JsonConvert.DeserializeObject(text);
    var result = (JObject)data["result"];
    var transactions = (JArray)result["transactions"];

    Debug.Log(transactions.Count);
}
```

コルーチンで書いた場合、 Coroutine に書かれた実行内容が完了するまでメインスレッド占有するので、上記例だと 66.35 ms もの間、メインスレッドを占有してしまう（await していたタスク実行後の JSON パースの処理もメインスレッドに乗ってしまう）

これが、Task + async / await を使うと

```cs
async Task Async()
{
    UnityWebRequest request = UnityWebRequest.Get("https://api.etherscan.io/api? module=proxy&action=eth_getBlockByNumber&tag=0x517df3&boolean=true&apikey=YourApiKeyToken");

    await request.SendWebRequest();

    var text = request.downloadHandler.text;
    await Task.Run(() => {
        var data = (JObject)JsonConvert.DeserializeObject(text);
        var result = (JObject)data["result"];
        var transactions = (JArray)result["transactions"];

        Debug.Log(JsonConvert.DeserializeObject(text));
    });
}
```

await Task.Run 内のクロージャはメインスレッドでは処理されなくなり、結果重かった JSON Parse の処理が非同期（メインスレッド外）に追い出され 21.27 ms だけメインスレッドで実行されることになる

よって、 Coroutine よりも Task Async / Await を使っていく方がよい

同時に、 UniRx を使っているプロジェクトの場合は、 [UniRx.Async（UniTask）機能紹介](https://qiita.com/toRisouP/items/4445b6b9bf00e49eb147) によると、 .NET 4.x に切り替えることと Incremental Compiler の導入さえできれば UniRx.Async が採用できるので積極的に使っていきたいですね！


