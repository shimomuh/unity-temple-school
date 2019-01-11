# Original Unity と UniRx の違い

こと通信において。

## Original Unity
このアプリケーションの WebSokectClient と Chat をメインに一部抜粋してみる

### Subject となりうる WebSocketClient
```cs
    public class WebSocketClient
    {
        public Action<Message> ReceiveMessageCallback;
        public Action ConnectFailureCallback;
        public Action ConnectSuccessCallback;
        public Action<ErrorEventArgs> ConnectErrorCallback;

        public void OnOpen(object sender, EventArgs e)
        {
            ConnectSuccessCallback();
        }

        public void OnMessage(object sender, MessageEventArgs e)
        {
            ReceiveMessageCallback(message);
        }

        public void OnError(object sender, ErrorEventArgs e)
        {
            ConnectErrorCallback(e);
        }

        public void OnClose(object sender, CloseEventArgs e)
        {
            ConnectFailureCallback();
        }
    }
```

### Observable なオブジェクトに対して処理を追記する Chat
```cs
    public class Chat : MonoBehaviour
    {
        private void Awake()
        {
            webSocketClient.ReceiveMessageCallback = ReceiveMessageCallback;
            webSocketClient.ConnectSuccessCallback = ConnectSuccessCallback;
            webSocketClient.ConnectFailureCallback = ConnectFailureCallback;
            webSocketClient.ConnectErrorCallback = ConnectErrorCallback;
        }

        public void ReceiveMessageCallback(Message message)
        {
             // メインスレッドに処理を戻す
            context.Post(__ => {
                if (user.id == message.user.id) {
                    return;
                }
                inputFieldPresenter.ReceiveMessage(message);
            }, null);
        }

        public void ConnectSuccessCallback()
        {
            Debug.Log("WebSocket Connect");
        }

        public void ConnectFailureCallback()
        {
            Debug.Log("WebSocket TryAgain");
            if (!webSocketClient.IsAbortTryConnect)
            {
                StartCoroutine(Retry());
            }
        }

        public void ConnectErrorCallback(ErrorEventArgs e)
        {

            Debug.Log("WebSocket Error Message: " + e.Message);
        }
    }
```

こうして書いてみると、 UniRx っぽく書くことは可能。
`IObservable<Object>` として連結する部分を `Action<Object>` として読み替えれば良いだけなので、大きな学習コストもかからない。
今回 OnNext にあたるストリームの実行は WebSocket-Sharp 内部に隠蔽されているが、モジュールから呼び出されると考えればなんら不思議はない。

## UniRx の真価は？
個人的に寿命の管理かなと思っている。
Action を使えば同じようなことができるが、 `AddTo(DisposableObject)` や `AddTo(this)` (特にこれ) で破棄を手軽にできるのは魅力的である。
DirectX のときもメモリを圧迫するオブジェクトがあったらゾンビプロセス化せずにちゃんと破棄されているかを確認することがあったが、このあたりの問題を脳死で `AddTo(this)` とすれば概ねそのオブジェクトが破棄されるであろう場面で一緒に捨ててくれるのは魅力的である。
