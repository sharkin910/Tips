MS DI（C#での標準的なDIライブラリ）について
===========================================

WinForm でも問題なく使えて、C# の世界で標準的といえる DI（依存性注入）ライブラリは **Microsoft.Extensions.DependencyInjection（通称：MS DI）** です。  
ASP.NET Core の DI と同じ仕組みで、.NET 公式が提供しているため、今後の拡張性・保守性の面でも最も安全な選択肢になります。

---

## WinForm で使える標準 DI：Microsoft.Extensions.DependencyInjection

### 特徴
- .NET 公式の DI コンテナ  
- ASP.NET Core と同じ書き方ができる  
- WinForm / WPF / Console でも利用可能  
- 軽量で学習コストが低い  
- 拡張性が高く、後からサービス追加しやすい  

WinForm の場合、**Program.cs で DI コンテナを構築し、Form に依存性を注入する**形が基本になります。

---

## WinForm + MS DI の最小構成例

### 1. NuGet パッケージを追加
```
Microsoft.Extensions.DependencyInjection
Microsoft.Extensions.Hosting
```

### 2. Program.cs を DI 対応にする
```csharp
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

internal static class Program
{
    [STAThread]
    static void Main()
    {
        ApplicationConfiguration.Initialize();

        var host = Host.CreateDefaultBuilder()
            .ConfigureServices((context, services) =>
            {
                // サービス登録
                services.AddSingleton<NetworkService>();
                services.AddSingleton<ConfigService>();

                // Form も DI で生成
                services.AddTransient<MainForm>();
            })
            .Build();

        // DI から MainForm を取得して起動
        var mainForm = host.Services.GetRequiredService<MainForm>();
        Application.Run(mainForm);
    }
}
```

---

## 3. Form に依存性を注入する

### MainForm.cs
```csharp
public partial class MainForm : Form
{
    private readonly NetworkService _networkService;
    private readonly ConfigService _configService;

    public MainForm(NetworkService networkService, ConfigService configService)
    {
        InitializeComponent();
        _networkService = networkService;
        _configService = configService;
    }

    private void button1_Click(object sender, EventArgs e)
    {
        var ip = _networkService.GetCurrentIp();
        MessageBox.Show(ip);
    }
}
```

---

## 4. サービスクラスの例

```csharp
public class NetworkService
{
    public string GetCurrentIp()
    {
        // netsh や WMI で取得する処理
        return "192.168.1.10";
    }
}

public class ConfigService
{
    public string LoadConfig()
    {
        return "config loaded";
    }
}
```

---

## WinForm で DI を使うと何が嬉しいか

- **グローバル変数をほぼゼロにできる**  
  → AppState や static クラスに依存しなくて済む

- **テストしやすい**  
  → NetworkService をモックに差し替え可能

- **依存関係が明確になる**  
  → Form のコンストラクタに必要なものだけ書く

- **後から機能追加しやすい**  
  → services.AddSingleton / AddTransient を追加するだけ

- **WinForm の古い構造でも無理なく導入できる**  
  → Program.cs に数行追加するだけで導入完了

---

## ライフサイクル（Singleton / Scoped / Transient）

| ライフサイクル | WinForm での意味 |
|----------------|------------------|
| **Singleton** | アプリ起動中ずっと同じインスタンス（設定・サービス向き） |
| **Transient** | 呼び出すたびに新しいインスタンス（Form 向き） |
| **Scoped** | WinForm では基本使わない（Web のリクエスト単位の概念） |

WinForm では **Singleton と Transient の2つだけ覚えれば十分** です。

---

## もう一歩進んだ使い方（必要なら）

- **IOptions パターンで設定管理**  
- **ILogger でログを統一管理**  
- **HostedService でバックグラウンド処理**  
- **Factory パターンで動的に Form を生成**

WinForm でも ASP.NET Core と同じ設計ができるようになります。

---

理解しておくべき DI の仕組み
==========================

結論から言うと、**MS DI（Microsoft.Extensions.DependencyInjection）は、コンストラクターの引数を自動的に解決して注入します**。  
ただし、**同じインターフェイスを複数登録した場合は自動では選べず、例外が発生する**か、**IEnumerable<T> として複数まとめて受け取る**必要があります。

以下、理解しておくべき DI の仕組みを、内部動作レベルまで整理します。

---

## 1. コンストラクター引数はどうやって自動的にセットされるのか

MS DI は **「コンストラクターの依存関係を再帰的に解決する」** 仕組みを持っています。

### 仕組みの流れ
1. `GetRequiredService<MainForm>()` を呼ぶ  
2. DI コンテナが MainForm のコンストラクターを調べる  
3. 引数の型（NetworkService, ConfigService）を確認  
4. それぞれの型が DI に登録されているかチェック  
5. 登録されていればインスタンスを生成  
6. 生成したインスタンスをコンストラクターに渡す  
7. MainForm が完成する

### 例
```csharp
public MainForm(NetworkService networkService, ConfigService configService)
```

この2つの型が `services.AddSingleton<NetworkService>()` などで登録されていれば、**自動的に注入される**。

---

## 2. コンストラクターが複数あったらどうなる？

MS DI は **「最も引数の多いコンストラクター」を選ぶ** というルールがあります。

例：

```csharp
public MainForm() { }

public MainForm(NetworkService networkService) { }
```

→ **NetworkService を受け取る方が選ばれる**

ただし、依存関係が解決できない場合は例外になります。

---

## 3. 同じインターフェイスを持つものが複数登録されていたら？

### 結論
- **単一の T を要求した場合 → 例外（Ambiguous）**
- **IEnumerable<T> を要求した場合 → 全部注入される**

### 例：複数登録

```csharp
services.AddSingleton<ILogger, FileLogger>();
services.AddSingleton<ILogger, DbLogger>();
```

### 1) 単一の ILogger を要求した場合

```csharp
public MainForm(ILogger logger)
```

→ **例外が発生する**  
（どれを渡すべきか DI が判断できないため）

### 2) IEnumerable<ILogger> を要求した場合

```csharp
public MainForm(IEnumerable<ILogger> loggers)
```

→ **FileLogger と DbLogger の2つが配列として渡される**

---

## 4. 「どれか1つだけ選びたい」場合の方法

### 方法A：名前付き登録は MS DI には存在しない  
（Autofac などの DI コンテナにはあるが、MS DI にはない）

### 方法B：ファクトリーパターンを使う（最も一般的）

```csharp
services.AddSingleton<FileLogger>();
services.AddSingleton<DbLogger>();

services.AddSingleton<Func<string, ILogger>>(provider => key =>
{
    return key switch
    {
        "file" => provider.GetRequiredService<FileLogger>(),
        "db"   => provider.GetRequiredService<DbLogger>(),
        _      => throw new ArgumentException("unknown key")
    };
});
```

Form 側：

```csharp
public MainForm(Func<string, ILogger> loggerFactory)
{
    var logger = loggerFactory("file");
}
```

---

## 5. DI が解決できない場合はどうなる？

### 例外が発生する（InvalidOperationException）

代表的な原因：

- 登録されていない型を要求した  
- 複数の実装が登録されていて単一解決できない  
- 循環依存（A → B → A）  
- コンストラクターが複数あり、選ばれたコンストラクターの依存が解決できない  

---

## 6. WinForm での DI の注意点

### Form は基本的に Transient にする
```csharp
services.AddTransient<MainForm>();
```

### Form を new しない
```csharp
var form = new MainForm(); // * DI が効かない
```

必ず DI から取得する：

```csharp
var form = provider.GetRequiredService<MainForm>(); // *
```

---

## 7. DI の内部動作（もう少し深い話）

MS DI は以下の順序で依存を解決します。

1. **型の登録情報を検索**
2. **ライフサイクル（Singleton / Transient）に応じてインスタンスを生成 or 再利用**
3. **コンストラクターを選択（最大引数のもの）**
4. **引数の型を再帰的に解決**
5. **循環依存があれば例外**
6. **最終的にインスタンスを返す**

つまり、**DI は「型のグラフ」を構築して、必要な順にインスタンスを作っていく仕組み**です。

---

ログ出力にDIを利用するサンプル
============================

ログ出力に DI を使うと **「差し替え・テスト・拡張」が圧倒的に楽になる**というメリットがあります。  
ここでは WinForm を前提に、**DI を使わない場合 → DI を使った場合**の比較、そして **DI のメリットが明確に出るサンプルコード**を示します。

---

## ログ出力に DI を使う最大のメリット（要点）
- **ログの実装を差し替えられる**（ファイル → デバッグ出力 → DB → 何でも）
- **テスト時にモックを注入できる**（ファイル書き込みを行わない）
- **Form やサービスがログの実装に依存しなくなる**
- **グローバル変数や static ロガーが不要になる**
- **複数のログ出力を同時に使える（IEnumerable<ILogger>）**

---

## 1. DI を使わないログ出力（悪い例）

### FileLogger を直接 new して使う
```csharp
public class NetworkService
{
    private readonly FileLogger _logger = new FileLogger("log.txt");

    public void Connect()
    {
        _logger.Write("Connecting...");
        // 処理
    }
}
```

### 問題点
- FileLogger に **強く依存**している  
- テスト時にファイル書き込みが走る  
- ログの出力先を変えたいときに **NetworkService のコードを修正する必要がある**  
- グローバル変数や static ロガーを使うと依存関係が不透明になる  

---

## 2. DI を使ったログ出力（良い例）

### インターフェイスを定義
```csharp
public interface ILogger
{
    void Write(string message);
}
```

### 実装1：ファイルログ
```csharp
public class FileLogger : ILogger
{
    private readonly string _path;

    public FileLogger(string path)
    {
        _path = path;
    }

    public void Write(string message)
    {
        File.AppendAllText(_path, message + Environment.NewLine);
    }
}
```

### 実装2：デバッグログ
```csharp
public class DebugLogger : ILogger
{
    public void Write(string message)
    {
        System.Diagnostics.Debug.WriteLine(message);
    }
}
```

---

## 3. DI コンテナに登録（Program.cs）

```csharp
services.AddSingleton<ILogger, FileLogger>(provider =>
    new FileLogger("app.log"));

// 別の実装に切り替えたいならここを変えるだけ
// services.AddSingleton<ILogger, DebugLogger>();
```

---

## 4. NetworkService に DI で注入

```csharp
public class NetworkService
{
    private readonly ILogger _logger;

    public NetworkService(ILogger logger)
    {
        _logger = logger;
    }

    public void Connect()
    {
        _logger.Write("Connecting...");
        // 処理
        _logger.Write("Connected.");
    }
}
```

### メリット
- NetworkService は **ILogger にしか依存しない**  
- FileLogger か DebugLogger かは **Program.cs が決める**  
- NetworkService のコードは一切変更不要  

---

## 5. Form にも DI で注入

```csharp
public partial class MainForm : Form
{
    private readonly NetworkService _networkService;
    private readonly ILogger _logger;

    public MainForm(NetworkService networkService, ILogger logger)
    {
        InitializeComponent();
        _networkService = networkService;
        _logger = logger;
    }

    private void button1_Click(object sender, EventArgs e)
    {
        _logger.Write("Button clicked.");
        _networkService.Connect();
    }
}
```

---

## 6. テスト時のメリット（モック注入）

### テスト用のモックロガー
```csharp
public class MockLogger : ILogger
{
    public List<string> Messages { get; } = new();

    public void Write(string message)
    {
        Messages.Add(message);
    }
}
```

### テストコード
```csharp
var logger = new MockLogger();
var service = new NetworkService(logger);

service.Connect();

Assert.Contains("Connecting...", logger.Messages);
```

### メリット
- ファイル書き込みなし  
- ログ内容を検証できる  
- NetworkService のコードは一切変更不要  

---

## 7. 複数ログ出力を同時に使う（IEnumerable<ILogger>）

### Program.cs
```csharp
services.AddSingleton<ILogger, FileLogger>(provider => new FileLogger("app.log"));
services.AddSingleton<ILogger, DebugLogger>();
```

### NetworkService
```csharp
public class NetworkService
{
    private readonly IEnumerable<ILogger> _loggers;

    public NetworkService(IEnumerable<ILogger> loggers)
    {
        _loggers = loggers;
    }

    public void Connect()
    {
        foreach (var logger in _loggers)
            logger.Write("Connecting...");

        // 処理
    }
}
```

### メリット
- ファイルにもデバッグにも同時にログが出る  
- 実装追加は Program.cs に AddSingleton を足すだけ  

---

## まとめ：DI を使うことで得られるログ出力のメリット

| 項目 | DI なし | DI あり |
|------|---------|---------|
| ログ実装の差し替え | コード修正が必要 | Program.cs の1行変更だけ |
| テスト | ファイル書き込みが走る | モックでテスト可能 |
| 依存関係 | 強結合 | インターフェイスで疎結合 |
| 複数ログ出力 | 面倒 | IEnumerable<ILogger> で簡単 |
| グローバル変数 | 必要になりがち | 不要 |

---
