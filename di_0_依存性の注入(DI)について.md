依存性の注入(DI)について
=======================

DI（Dependency Injection：依存性注入）とは、プログラム内でクラスが依存する別のオブジェクトを、内部で生成（new）せず、外部から渡す設計パターンです。
これによりクラス間の結びつきを弱め（疎結合）、テストの容易性や保守性を向上させます。

---

## 何が嬉しいか

- **グローバル変数の廃止が進む**  
  → 依存関係が見え易くなり、テストも容易になる。
  → 初期化順序や差し替えの管理が集約できる。

- **依存関係が明確になる**  
  → コンポーネントが独立するため、コードの再利用や変更がしやすくなる。
  → クラスの入れ子構造をなくしていける。

- **テストしやすい**  
  → インターフェースに依存させれば、本番用サービスをモックに差し替え可能。

- **オブジェクトの生成と消滅を自動化できる**  
  → 自動でコンストラクターを呼んで必要なものを生成。最後に破棄。

---

## C#での標準的なDIライブラリは？

WinForm でも問題なく使えて、C# の世界で“標準的”といえる DIライブラリは **Microsoft.Extensions.DependencyInjection（通称：MS DI）** です。  
ASP.NET Core の DI と同じ仕組みで、.NET 公式が提供しているため、今後の拡張性・保守性の面でも最も安全な選択肢になります。

### 特徴
- .NET 公式の DI コンテナ  
- ASP.NET Core と同じ書き方ができる  
- WinForm / WPF / Console でも利用可能  
- 軽量で学習コストが低い  

### NuGet パッケージ追加が必要
```
Microsoft.Extensions.DependencyInjection
Microsoft.Extensions.Hosting // こちらは必要に応じて
```

---

## 考え方

DIの対象になるのは **コントローラー、サービス、ビジネスロジックなど** です。
データや一時オブジェクトについては考えなくてよいです。

1. クラスをstaticにしない。（例外はユーティリティークラス。例：Math）
2. クラス内で使うクラスは、できるだけコンストラクターで与える。
3. シングルトンとして使うか、都度生成させるかを決めて、DIに登録する。
4. 生成はDIにまかせる。（芋づる式に生成される）

---

## サンプル（コンソールプログラム）

```csharp
using Microsoft.Extensions.DependencyInjection;

namespace Test
{
    internal static class Program
    {
        [STAThread]
        private static void Main()
        {
            var services = new ServiceCollection();
            // クラス名を指定して、シングルトンとして登録（後で生成）
            services.AddSingleton<Config>();
            services.AddSingleton<MyApp>();

            using var provider = services.BuildServiceProvider();

            // Configを取り出して値を設定（自動でnewしてくれる）
            var config = provider.GetRequiredService<Config>();
            config.Value = "A";

            // MyAppを生成（Configは自動で渡される）
            var app = provider.GetRequiredService<MyApp>();
            app.Process();

            // 試しに、再度Configを取り出してみる（2重生成されてないことがわかる）
            var config2 = provider.GetRequiredService<Config>();
            Console.WriteLine(config2.Value);
        }
    }

    class Config // 設定保存用クラスの例
    {
        public string Value { get; set; } = "";
    }

    class MyApp // 機能クラスの例
    {
        private readonly Config _config;

        public MyApp(Config config)
        {
            _config = config;
        }

        public void Process()
        {
            Console.WriteLine(_config.Value); // 機能を実装
            _config.Value = "B";
        }
    }
}
```

---

## 登録するときにライフサイクルを指定

| ライフサイクル | WinForm での意味 |
|----------------|------------------|
| **Singleton** | アプリ起動中ずっと同じインスタンス（設定・サービス向き） |
| **Transient** | 呼び出すたびに新しいインスタンス（Form 向き） |
| **Scoped** | Web のリクエスト単位の概念 |

WinForm では **Singleton と Transient の2つで十分**。

---

## コンストラクター引数は自動でセットされる

### 例：MyAppが生成の流れ

1. `GetRequiredService<MyApp>()` を呼ぶ  
2. DI コンテナが MyApp の最も引数の多いコンストラクターを調べる  
3. 引数の型（Config）を確認  
4. 型が DI に登録されているかチェック（無ければ例外発生）  
5. 登録されていればインスタンスを生成  
6. 生成したインスタンスをMyAppのコンストラクターに渡す  
7. MyApp が完成する

---

## インターフェイスで登録すれば差し替えが容易

例えばこの例では、FileLoggerをDebugLoggerに差し替えても利用側は修正不要。

### 例）登録側（クラスの型ではなくインターフェイス指定で登録）

```csharp
services.AddSingleton<ILogger, FileLogger>();
```

### 例）利用側

```csharp
public MainService(ILogger logger) { ... }
```

---

## 同じインターフェイスが複数登録されていたら？

配列で受け取るなどの工夫が必要。 
詳細は別紙。

---

## WinForm での注意点

- Form は基本的に Transient にする

```csharp
services.AddTransient<MainForm>();
```

- 必ず DI から取得する

```csharp
var form = provider.GetRequiredService<MainForm>();
//var form = new MainForm(); // これだと DI が効かない
```

- 引数無しのコンストラクターは必要（デザイナーのため）

```csharp
public partial class MainForm : Form
{
    private readonly IMyService _service;

    // 1. デザイナー用（引数なし）（デザイン時、Visual Studio はここを通る）
    public MainForm()
    {
        InitializeComponent();
    }

    // 2. 実行時 DI 用（引数あり）（DI コンテナはここを優先して呼び出す）
    public MainForm(IMyService service) : this()
    {
        _service = service;
    }
```
---

## WinForms の実装例

```csharp
using Microsoft.Extensions.DependencyInjection;

internal static class Program
{
    [STAThread]
    static void Main()
    {
        ApplicationConfiguration.Initialize();

        var services = new ServiceCollection();
        services.AddSingleton(new Config { Value = "A" });
        services.AddSingleton<MyApp>();          // ビジネスロジック等
        services.AddSingleton<MainForm>();       // フォームも DI で生成

        using var provider = services.BuildServiceProvider();

        var mainForm = provider.GetRequiredService<MainForm>();
        Application.Run(mainForm);
    }
}

public class MainForm : Form
{
    private readonly MyApp _app;

    public MainForm(MyApp app)
    {
        _app = app;
        // ...
    }
}

// COnfigとMyAppの実証は前と同じため省略
```

---

## 利用しているインスタンスの状態変化を知る方法

例えばConfigの中身が変わっても、**利用側へは自動で通知されない** ので、
必要なら、イベントなど**別の仕組み**で通知する必要がある。 
詳細は別紙。

---

## 自前でnewしてから登録したら破棄も自前

**IDisposable** なクラスは自動でDisposeしてくれる。 
だが、**自前で作ってから注入する場合は、自前で破棄する必要あり**。

```csharp
static void Main()
{
    // 事前に生成するときは、usingするか、あとでDispose()する！
    using var config = new AppConfig();
    config.Load(); // などなど

    var services = new ServiceCollection();
    services.AddSingleton(config);      // 生成後に登録

    // ...
}

public class AppConfig : IDisposable
{
    public void Dispose() { ... }
}
```

ちなみにFormは、Closeしたり、Application.Runから抜けたら、Disposeされる。

---
