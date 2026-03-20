シングルトンの状態変化を利用側へ通知する方法
============================================

DI で注入した **Singleton の状態が変わっても、利用側へは自動で通知してくれない**。

通知は **別の仕組み**で実現する必要がある。

WinForm で現実的に使える方法は次の 4 つ。

---

# ① 最もシンプル：イベントで通知する（WinForm では王道）

### サービス側（Singleton）
```csharp
public class ConfigService
{
    public event EventHandler? ConfigChanged;

    private string _value;
    public string Value
    {
        get => _value;
        set
        {
            if (_value != value)
            {
                _value = value;
                ConfigChanged?.Invoke(this, EventArgs.Empty);
            }
        }
    }
}
```

### 利用側（Form）
```csharp
public partial class MainForm : Form
{
    private readonly ConfigService _config;

    public MainForm(ConfigService config)
    {
        InitializeComponent();
        _config = config;

        _config.ConfigChanged += (_, __) =>
        {
            label1.Text = _config.Value;
        };
    }
}
```

### メリット
- WinForm と相性が良い  
- シンプルで分かりやすい  
- DI と完全に共存できる  

---

# ② ViewModel 的に使う：INotifyPropertyChanged を使う

WPF でよく使うけど、WinForm でも普通に使える。

### サービス側
```csharp
public class AppState : INotifyPropertyChanged
{
    public event PropertyChangedEventHandler? PropertyChanged;

    private string _status;
    public string Status
    {
        get => _status;
        set
        {
            if (_status != value)
            {
                _status = value;
                PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(nameof(Status)));
            }
        }
    }
}
```

### 利用側（Form）
```csharp
public MainForm(AppState state)
{
    InitializeComponent();

    state.PropertyChanged += (_, e) =>
    {
        if (e.PropertyName == nameof(AppState.Status))
        {
            labelStatus.Text = state.Status;
        }
    };
}
```

### メリット
- プロパティ単位で通知できる  
- UI バインディングに近い書き方ができる  

---

# ③ 複数の通知をまとめたいなら「イベントアグリゲーター（メッセージバス）」

規模が大きくなると、イベントが増えて管理が大変になる。  
その場合は「通知専用サービス」を DI に登録する。

### メッセージバス例
```csharp
public class EventBus
{
    public event EventHandler<string>? MessagePublished;

    public void Publish(string message)
    {
        MessagePublished?.Invoke(this, message);
    }
}
```

### サービス側
```csharp
_eventBus.Publish("ConfigUpdated");
```

### 利用側
```csharp
_eventBus.MessagePublished += (_, msg) =>
{
    if (msg == "ConfigUpdated")
        RefreshUI();
};
```

### メリット
- 大規模アプリで強い  
- 依存関係がシンプルになる  

---

# ④ Reactive Extensions（IObservable）を使う（高度だけど強力）

```csharp
public class StateService
{
    private readonly BehaviorSubject<int> _counter = new(0);
    public IObservable<int> Counter => _counter;

    public void Increment() => _counter.OnNext(_counter.Value + 1);
}
```

利用側：

```csharp
_stateService.Counter.Subscribe(value =>
{
    label1.Text = value.ToString();
});
```

### メリット
- 非同期・連続イベントに強い  
- UI 更新がとても書きやすい  

---

# どれを選ぶべき？

WinForm なら次の順番で考えると良いよ。

| 規模 | おすすめ |
|------|----------|
| 小規模 | **イベント** |
| 中規模 | **INotifyPropertyChanged** |
| 大規模 | **イベントバス（メッセージバス）** |
| 非同期・連続更新 | **IObservable（Rx）** |

---

# まとめ

DI で Singleton を注入しただけでは、**状態変化は自動通知されない**。

通知したいなら、  
- イベント  
- INotifyPropertyChanged  
- メッセージバス  
- IObservable  
のどれかを組み合わせる必要がある。

---
