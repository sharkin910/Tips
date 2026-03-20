DIの使い方深堀
==============

`.NET` の `Microsoft.Extensions.DependencyInjection` による依存性注入（DI）は、**疎結合・テスト容易性・拡張性・構成の一元化**を実現するための標準基盤です。  
ここでは、

1.  **どんな場面で有効か（ユースケース）**
2.  **基本の使い方（登録・解決・ライフタイム）**
3.  **実戦パターン（ASP.NET Core/コンソール/バックグラウンド/Options/HttpClient/EF Core/ロギング）**
4.  **テストでの使い方**
5.  **設計ベストプラクティス & ありがちな落とし穴**
6.  **拡張テクニック（モジュール化、スキャン、置換）**

の順でまとめます。

***

## 1) どういった場面で有効か

*   **テスト容易性**：実装を差し替えてユニットテスト（モック/スタブ）しやすくなる
    *   例：`IEmailSender` の本番実装とテスト用ダミー実装の切替
*   **実装の差し替え/機能フラグ**：環境（Dev/QA/Prod）や設定で簡単に実装を入れ替え
    *   例：本番は SendGrid、開発はローカル SMTP 風の実装
*   **横断的関心事の一元化**：ロギング、リトライ、認証/認可、キャッシュ、ポリシーなどをパイプラインで統合
    *   例：`IHttpClientFactory` + Polly で外部API呼び出しのリトライ/サーキットブレーカー
*   **構成の集中管理**：接続文字列やAPIキーを `IOptions<T>` で型付きに管理し、依存先に注入
*   **拡張性の高いプラグイン構造**：外部ライブラリが `AddXxx()` 拡張メソッドで自前のサービスを登録できる
*   **スコープ管理が必要なリソース**：DBコンテキストやスコープ付きキャッシュなどのライフタイム管理

***

## 2) 基本の使い方

### 登録（Service Registration）

`IServiceCollection` に「インターフェイス ⇒ 具象」マッピングを登録します。

*   ライフタイムの基本3種：
    *   **Singleton**：アプリ全体で1個
    *   **Scoped**：スコープ単位（Web用。リクエストごと）
    *   **Transient**：解決のたびに新規

```csharp
public interface IMessageSender { void Send(string msg); }
public class EmailSender : IMessageSender { public void Send(string msg) { /*...*/ } }

var services = new ServiceCollection();

// ライフタイム別の例
services.AddSingleton<IMessageSender, EmailSender>();   // 全体で1つ
services.AddScoped<MyDbContext>();                      // リクエスト/スコープごと
services.AddTransient<MyService>();                     // 毎回新規
```

### 解決（Service Resolution）

*   **推奨**：コンストラクタインジェクション（最も明示的でテスト容易）
*   フレームワーク側が自動解決（Controller, Minimal API のハンドラ, HostedService など）

```csharp
public class OrderHandler
{
    private readonly IMessageSender _sender;
    public OrderHandler(IMessageSender sender) => _sender = sender;

    public void Handle() => _sender.Send("ordered.");
}
```

> 直接 `serviceProvider.GetRequiredService<T>()` を多用するのは **サービスロケータパターン** になりがちで非推奨（後述）。

***

## 3) 実戦パターン

### A. ASP.NET Core（Minimal API）の例

```csharp
var builder = WebApplication.CreateBuilder(args);
var services = builder.Services;

// 依存の登録
services.AddScoped<IMessageSender, EmailSender>();
services.AddDbContext<AppDbContext>(opt =>
    opt.UseNpgsql(builder.Configuration.GetConnectionString("Default")));

// Options パターン（appsettings.json → 型付き）
services.Configure<MailOptions>(builder.Configuration.GetSection("Mail"));

var app = builder.Build();

// DI されたサービスをエンドポイントで受け取る
app.MapPost("/notify", (Order order, IMessageSender sender) =>
{
    sender.Send($"Order {order.Id}");
    return Results.Ok();
});

app.Run();
```

ポイント：

*   エンドポイント引数に直接インジェクションできる
*   `AddDbContext` はスコープ（リクエスト）単位が既定。`DbContext` は `Scoped` と相性が良い

### B. コンソールアプリ（Generic Host）

```csharp
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.DependencyInjection;

await Host.CreateDefaultBuilder(args)
    .ConfigureServices(services =>
    {
        services.AddTransient<IMain, MainApp>();
        services.AddHttpClient(); // IHttpClientFactory
        services.AddHostedService<Worker>(); // バックグラウンド処理
    })
    .RunConsoleAsync();

public interface IMain { Task RunAsync(); }
public class MainApp : IMain
{
    private readonly IHttpClientFactory _httpClientFactory;
    public MainApp(IHttpClientFactory httpClientFactory) => _httpClientFactory = httpClientFactory;

    public async Task RunAsync()
    {
        var client = _httpClientFactory.CreateClient();
        var html = await client.GetStringAsync("https://example.com");
        // ...
    }
}

public class Worker : BackgroundService
{
    private readonly IMain _main;
    public Worker(IMain main) => _main = main;
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await _main.RunAsync();
    }
}
```

### C. Options パターン（設定の型付き注入）

```csharp
public class MailOptions
{
    public string SmtpHost { get; set; } = default!;
    public int Port { get; set; }
}

services.Configure<MailOptions>(configuration.GetSection("Mail"));

public class EmailSender : IMessageSender
{
    private readonly MailOptions _opt;
    public EmailSender(IOptions<MailOptions> options) => _opt = options.Value;
    public void Send(string msg) { /* _opt.SmtpHost を使用 */ }
}
```

*   変更監視が必要なら `IOptionsMonitor<T>`
*   スコープに閉じたいなら `IOptionsSnapshot<T>`（Web専用、リクエストごとにスナップショット）

### D. 名前付き/型付き HttpClient（ポリシー/ヘッダ/ベースURLの共通化）

```csharp
services.AddHttpClient("github", client =>
{
    client.BaseAddress = new Uri("https://api.github.com/");
    client.DefaultRequestHeaders.UserAgent.ParseAdd("MyApp");
})
// Polly のリトライなど（Polly拡張パッケージが必要）
.AddTransientHttpErrorPolicy(p => p.RetryAsync(3));
```

解決側：

```csharp
public class GitHubClient
{
    private readonly HttpClient _client;
    public GitHubClient(IHttpClientFactory factory) => _client = factory.CreateClient("github");
}
```

### E. EF Core（DbContext のスコープ管理）

```csharp
services.AddDbContext<AppDbContext>(opt =>
    opt.UseSqlServer(configuration.GetConnectionString("Default")));
```

*   `DbContext` は **Scoped** が原則。`Singleton` にしてはいけない（スレッドアンセーフ）

### F. ロギング

```csharp
var logger = app.Services.GetRequiredService<ILogger<Program>>();
logger.LogInformation("Started");
```

*   本来はコンストラクタに `ILogger<T>` を注入して使うのが自然

***

## 4) テストでの使い方

### シンプルな差し替え（モック/スタブ）

xUnit + Moq 例：

```csharp
var services = new ServiceCollection();
services.AddTransient<OrderHandler>();
var mock = new Mock<IMessageSender>();
services.AddSingleton<IMessageSender>(mock.Object);

var sp = services.BuildServiceProvider();
var handler = sp.GetRequiredService<OrderHandler>();
handler.Handle();

mock.Verify(x => x.Send(It.IsAny<string>()), Times.Once);
```

### Web アプリの統合テスト

*   ASP.NET Core の `WebApplicationFactory<TEntryPoint>` で `ConfigureServices` 上書きすることで、DB を InMemory に差し替えたり外部APIをスタブ化できます。

***

## 5) ベストプラクティス & ありがちな落とし穴

### ベストプラクティス

*   **コンストラクタインジェクションを基本に**：必須依存のみ注入。可変/オプションは「メソッド引数」や `IOptions` で
*   **拡張メソッドで登録をモジュール化**：
    ```csharp
    public static class MyModule
    {
        public static IServiceCollection AddMyModule(this IServiceCollection services, IConfiguration config)
        {
            services.Configure<MailOptions>(config.GetSection("Mail"));
            services.AddScoped<IMessageSender, EmailSender>();
            return services;
        }
    }
    // Program.cs
    services.AddMyModule(builder.Configuration);
    ```
*   **`IDisposable`/`IAsyncDisposable` の適切な破棄**：`BuildServiceProvider` したらアプリ終了時に `Dispose()` されるように `Host` を使う
*   **`IHttpClientFactory` を使う**：`new HttpClient()` 直はソケット枯渇の危険。必ずファクトリで
*   **Options のバリデーション**：`services.AddOptions<MailOptions>().Bind(...).ValidateDataAnnotations();`

### よくある落とし穴

*   **サービスロケータ化**：`IServiceProvider.GetService(...)` をあちこちで直接呼ぶ  
    → テスタビリティ低下。基本はコンストラクタインジェクション
*   **ライフタイムの不整合（Captive Dependency）**：
    *   `Singleton` が `Scoped` や `Transient` を保持するのは危険（より短命の依存を長命がキャプチャしてしまう）
    *   原則：**長命 ⇒ 短命** の参照は避ける。`Scoped` は `Singleton` に注入しない
*   **`DbContext` を Singleton にする**：厳禁。スレッドアンセーフ & ステート保持
*   **スコープの誤用**：Web 以外（コンソール/Worker）でスコープ必要な処理を `using var scope = provider.CreateScope()` せずに使う
*   **`async` 破棄忘れ**：`IAsyncDisposable` を持つサービス（例えば `Channel`/一部のクライアント）を適切に `await using` またはホストが破棄するように

***

## 6) 拡張テクニック

### A. 条件分岐で実装差し替え

```csharp
if (builder.Environment.IsDevelopment())
{
    services.AddSingleton<IEmailSender, ConsoleEmailSender>();
}
else
{
    services.AddSingleton<IEmailSender, SendGridEmailSender>();
}
```

### B. オープンジェネリック登録

```csharp
services.AddTransient(typeof(IRepository<>), typeof(EfRepository<>));
```

### C. スキャン（外部コンテナ or ライブラリ併用）

標準 DI はアセンブリスキャンを組み込み提供しないため、必要なら Scrutor 等を併用：

```csharp
// services.Scan(scan => scan
//     .FromAssembliesOf(typeof(SomeType))
//     .AddClasses(classes => classes.InNamespaces("MyApp.Services"))
//     .AsImplementedInterfaces()
//     .WithScopedLifetime());
```

### D. 既定コンテナを他の DI コンテナへ差し替え

*   標準 DI はシンプルで十分なことが多いですが、**デコレーション**や**プロパティ注入**、**高度なスキャン**が必須なら  
    Autofac / Lamar 等でホスト統合が可能（ただしプロジェクトの複雑度は上がる）

***

## まとめ（いつ使うべき？）

*   **Web API / MVC / Minimal API では必須級**：リクエストスコープやミドルウェア、Controller で自動注入されるため
*   **バックグラウンド処理（Worker/HostedService）**：スコープ管理・HttpClient・Options・ログが一貫
*   **コンソール/バッチ**：Generic Host を使うと構成・ログ・DI が統一
*   **ライブラリ/プラグイン化**：`AddXxx()` 拡張メソッドで外部から容易に組み込み可能

***

## すぐ試せる最小サンプル（コンソール）

```csharp
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;

await Host.CreateDefaultBuilder(args)
    .ConfigureServices(services =>
    {
        services.AddTransient<IHelloService, HelloService>();
        services.AddHostedService<Runner>();
    })
    .RunConsoleAsync();

public interface IHelloService { void Say(string name); }
public class HelloService : IHelloService
{
    private readonly ILogger<HelloService> _logger;
    public HelloService(ILogger<HelloService> logger) => _logger = logger;
    public void Say(string name) => _logger.LogInformation("Hello, {Name}!", name);
}

public class Runner : BackgroundService
{
    private readonly IHelloService _hello;
    public Runner(IHelloService hello) => _hello = hello;

    protected override Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _hello.Say("世界");
        return Task.CompletedTask;
    }
}
```

***
