DIによるグローバル変数の廃止
============================

`Program.Config` のような「静的グローバル」に依存すると、

*   クラス間の結合が強くなりテストが難しい
*   初期化順序の隠れた前提が増える
*   将来の拡張（差し替え・設定のリロード）が辛い

…といった問題が出やすいです。ここでは **依存性の注入（DI）** をベースに、実装しやすく・取り出しコストも軽い現実的な選択肢をいくつか示します。サンプルは最小限のコードで、「重くならない・書き散らさない」ことを重視しています。

***

## 方針の全体像（推奨順）

1.  **最小構成の DI（手組み / `Microsoft.Extensions.DependencyInjection`）**
    *   シングルトン登録して **コンストラクタ注入**（取り出しコストはフィールド参照のみで最軽量）
2.  **Generic Host を用いた DI（WPF/WinForms 対応）**
    *   将来の拡張（ロギング、設定ファイル、背景タスク）も容易
3.  **移行期の足場：一時的 Service Locator（最小ラッパ）**
    *   既存コード量が多い場合の段階的移行用（最終的には撤去）

> 「共通オブジェクトの取り出しが頻繁」「軽い方がありがたい」という要件に対しては、**コンストラクタ注入のシングルトン**が最も軽く、実行時コストは**1 度解決 → 以降はフィールド参照**だけになります。

***

## 1) 最小構成の DI（手組み / ServiceCollection）

`Config` をシングルトンとして登録し、`App` には **コンストラクタ注入**します。`Program` 静的参照は不要になります。

```csharp
using Microsoft.Extensions.DependencyInjection;

namespace Test
{
    internal static class Program
    {
        [STAThread]
        private static void Main()
        {
            // ここが Composition Root（依存関係の集約点）
            var services = new ServiceCollection();

            // Config を組み立ててシングルトン登録
            services.AddSingleton(new Config { Value = "A" });

            // App も登録（必要に応じて他も）
            services.AddSingleton<App>();

            using var provider = services.BuildServiceProvider();

            // 以後、解決は起動時に一回。App 内ではフィールド参照のみで軽い
            var app = provider.GetRequiredService<App>();
            app.Process();
        }
    }

    class Config
    {
        public string Value { get; set; } = "";
    }

    class App
    {
        private readonly Config _config; // ← フィールド参照が最軽量

        public App(Config config)
        {
            _config = config;
        }

        public void Process()
        {
            var v = _config.Value;
            // use v for processing
        }
    }
}
```

*   **取り出しコスト**: `App` 生成時に 1 回だけ DI コンテナが解決。以降は `_config` フィールド参照で **ゼロに等しい**。
*   **可読性**: 「この型は何に依存するか」がコンストラクタで明瞭。
*   **テスト容易性**: `new App(new FakeConfig())` と直に差し替え可。

> 既存コードで多くのクラスが `Program.Config` を参照している場合も、**各クラスのコンストラクタに `Config`（または `IAppConfig`）を生やす**だけで、段階的に除去できます。

***

## 2) Generic Host を用いた DI（WPF/WinForms 向け）

### WinForms の例

```csharp
using Microsoft.Extensions.DependencyInjection;
using System;
using System.Windows.Forms;

internal static class Program
{
    [STAThread]
    static void Main()
    {
        ApplicationConfiguration.Initialize();

        var services = new ServiceCollection();

        services.AddSingleton(new Config { Value = "A" });
        services.AddSingleton<App>();            // ビジネスロジック等
        services.AddSingleton<MainForm>();       // フォームも DI で生成

        using var provider = services.BuildServiceProvider();

        var mainForm = provider.GetRequiredService<MainForm>();
        Application.Run(mainForm);
    }
}

public class MainForm : Form
{
    private readonly App _app;

    public MainForm(App app)
    {
        _app = app;
        // UI 初期化...
    }
}
```

### WPF の例（`App.xaml.cs`）

```csharp
using Microsoft.Extensions.DependencyInjection;
using System;
using System.Windows;

public partial class App : Application
{
    public IServiceProvider Services { get; private set; } = default!;

    protected override void OnStartup(StartupEventArgs e)
    {
        base.OnStartup(e);

        var sc = new ServiceCollection();
        sc.AddSingleton(new Config { Value = "A" });
        sc.AddSingleton<AppCore>();          // ロジック
        sc.AddSingleton<MainWindow>();       // ウィンドウも DI で

        Services = sc.BuildServiceProvider();

        var main = Services.GetRequiredService<MainWindow>();
        main.Show();
    }
}

public class MainWindow : Window
{
    private readonly AppCore _core;
    public MainWindow(AppCore core)
    {
        _core = core;
        InitializeComponent();
    }
}
```

> **ポイント**
>
> *   UI（Form/Window）もコンストラクタで依存を受け取り、**内部ではフィールド参照のみ**。
> *   設定リロードやロギング、バックグラウンド処理が必要になっても `Host`/DI の枠組みで自然に拡張できます。

***

## 3) 段階的移行のための「最小 Service Locator ラッパ」（暫定策）

既存コードが大量に `Program.Config` を読む前提で、**一気にコンストラクタ注入へ直すのが難しい**場合の「足場」です。最終的には除去する前提で使ってください。

```csharp
public static class AppServices
{
    public static IServiceProvider Provider { get; set; } = default!;

    public static T Get<T>() where T : notnull
        => Provider.GetRequiredService<T>();
}
```

`Main`（Composition Root）で設定:

```csharp
var services = new ServiceCollection();
services.AddSingleton(new Config { Value = "A" });
// ... 他登録
AppServices.Provider = services.BuildServiceProvider();
```

既存コードでは一時的に：

```csharp
var v = AppServices.Get<Config>().Value;
```

> **注意**: これは **Service Locator パターン**であり、設計上は非推奨です（依存が隠れるため）。**移行の間だけ**使い、最終的にコンストラクタ注入へ寄せてください。

***

## 設計のこつ（性能・保守性）

*   **シングルトン + コンストラクタ注入**が最軽量  
    一度解決したらフィールド参照のみ。頻繁アクセスでもコストは無視できるレベルです。
*   **設定の可変性**
    *   値を実行中に更新するなら、**不変オブジェクトを丸ごと差し替える**か、イベント通知で購読する設計にする。
    *   `Microsoft.Extensions.Options` を使えば `IOptionsSnapshot/IOptionsMonitor` で再読込も可能（必要性が出たら）。
*   **テスト容易性**  
    `Config` を `IAppConfig` インターフェイスで抽象化すると、ユニットテストで差し替えが簡単。
*   **段階的移行**
    1.  `Config` を DI コンテナに登録
    2.  新規コードからコンストラクタ注入に統一
    3.  既存の静的参照は重要度の高い順に置換
    4.  暫定的な Service Locator を撤去

***

## 参考（DI化最小版）

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

            services.AddSingleton(new Config { Value = "A" });
            services.AddSingleton<App>();

            using var provider = services.BuildServiceProvider();
            var app = provider.GetRequiredService<App>();
            app.Process();
        }
    }

    class Config
    {
        public string Value { get; set; } = "";
    }

    class App
    {
        private readonly Config _config;
        public App(Config config) => _config = config;

        public void Process()
        {
            var v = _config.Value;
            // use v for processing;
        }
    }
}
```

***

## もし「もっと軽く」したいときは？

*   取り出し回数が非常に多い箇所では、**一度注入したらローカル/フィールドにキャッシュ**（上例のとおり）。
*   コンテナから **毎回解決しない**（= 解決は Composition Root / ファクトリ / 生成時に一度だけ）。
*   シングルトンのフィールド参照で限界まで軽い。これ以上の最適化は通常不要。

***

