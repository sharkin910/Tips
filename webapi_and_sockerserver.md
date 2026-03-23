C#のsocketサーバー（Windows Formアプリへ）のWeb API追加
===============

ソケットサーバーが既に動いている中で、さらにWeb APIを統合する場合、**ASP.NET Coreの「自己ホスト（Self-hosting）」**を利用するのが、最もクリーンで強力な方法です。

---

## 推奨される構成：ASP.NET Core Kestrel の統合

以前は `Microsoft.AspNet.WebApi.SelfHost`（古いWeb API 2）が使われていましたが、現在は **ASP.NET Core Kestrel** をWinForms内でバックグラウンドタスクとして起動させるのがベストです。

### なぜこの方法が良いのか？

* **非同期処理:** WinFormsのUIスレッドを止めずにAPIを動かせる。
* **モダンな開発:** 最新の `System.Text.Json` や依存注入（DI）が使える。
* **拡張性:** 将来的にSwagger（APIドキュメント）などを簡単に追加できる。

---

## 実装のステップ

### 1. 必要なパッケージの導入

プロジェクトファイル（.csproj）を編集するか、NuGetで以下を意識してください。最近の .NET (6/7/8以降) であれば、SDKスタイルであれば非常に簡単です。

### 2. コードの実装例

`Form1.cs` などで、アプリ起動時にWeb APIを別タスクで立ち上げるイメージです。

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.DependencyInjection;

// WinFormsのボタンイベントなどで起動
private async void Form1_Load(object sender, EventArgs e)
{
    // APIをバックグラウンドで開始
    await Task.Run(() => StartWebApi());
}

private void StartWebApi()
{
    var builder = WebApplication.CreateBuilder();

    // ポート番号などの設定
    builder.WebHost.ConfigureKestrel(options =>
    {
        options.ListenLocalhost(5000); // http://localhost:5000
    });

    var app = builder.Build();

    // 簡単なエンドポイントの作成
    app.MapGet("/status", () => new { 
        Message = "WinForms Server is running", 
        Timestamp = DateTime.Now 
    });

    // WinForms側のデータにアクセスしたい場合は、
    // 適切なインスタンスを注入するか、静的クラスを経由させます。
    app.MapPost("/control", (Command cmd) => {
        // ここでソケットサーバーの操作などを実行
        return Results.Ok($"Command {cmd.Name} received");
    });

    app.Run();
}

```

---

## 注意すべきポイント

### 1. スレッドセーフな操作

Web APIのコントローラーは**別スレッド**から呼ばれます。API経由でWinFormsのテキストボックスを書き換えたり、UIを操作する場合は、必ず `Control.Invoke` を使用してください。

### 2. ポートの競合と権限

* 既存のソケットサーバーが使っているポートと被らないように注意してください。
* Windowsのセキュリティ設定により、`localhost` 以外（外部IP）で公開する場合は管理者権限やURL予約（`netsh http add urlacl`）が必要になることがあります。

### 3. クリーンアップ

アプリを閉じる際、APIホストを正しくシャットダウン（`StopAsync`）するように実装すると、リソース漏れを防げます。

--------

実は、最新の .NET（.NET 6/7/8/9以降）を使用している場合、**個別の NuGet パッケージを追加する必要はありません。**

むしろ、NuGet で個別に `Microsoft.AspNetCore.*` を入れるのは古い手法であり、現在はプロジェクトファイル（`.csproj`）を1行書き換えるだけで、ASP.NET Core の全機能が使えるようになります。

---

## 1. プロジェクトファイルの書き換え（最重要）

WinForms プロジェクトの実体である `.csproj` ファイルを右クリックして「プロジェクトファイルを編集する」を選択し、以下の `<FrameworkReference>` を追加してください。

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>WinExe</OutputType>
    <TargetFramework>net8.0-windows</TargetFramework> <UseWindowsForms>true</UseWindowsForms>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>

  <ItemGroup>
    <FrameworkReference Include="Microsoft.AspNetCore.App" />
  </ItemGroup>

</Project>

```

### なぜ NuGet ではないのか？

ASP.NET Core は現在「共有フレームワーク」として .NET SDK に含まれています。`FrameworkReference` を使うことで、バージョン競合の心配がなく、最適化された標準ライブラリ一式が利用可能になります。

---

## 3. 実装時の Tips：UIスレッドとの連携

API から WinForms のコントロール（ラベルやリストボックス）を操作したい場合は、必ず `Invoke` を使ってください。

```csharp
app.MapPost("/send-message", (string msg) => {
    // APIスレッドからUIスレッドへ安全にアクセス
    form.Invoke(new Action(() => {
        form.textBoxLog.AppendText($"API受信: {msg}\r\n");
    }));
    return Results.Ok();
});

```

---

## 次のステップ

これで Web API を動かす準備は整いました。
次は、**「ソケットサーバー側のデータやメソッドを API からどうやって呼び出すか」**（DI：依存注入の具体的な書き方）について解説が必要でしょうか？
