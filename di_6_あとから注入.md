事前に生成したインスタンスを、そのままシングルトンとしてDIコンテナに注入する方法
========================

## 1) インスタンスをそのまま渡して登録（最も簡単）

> **用途**: 既に作った `Config` などの素朴な値オブジェクトをそのまま使いたい  
> **Dispose**: **コンテナは破棄しません**（所有権は呼び出し側）

```csharp
using Microsoft.Extensions.DependencyInjection;

var services = new ServiceCollection();

var config = new Config { Value = "A" };   // 事前に生成
services.AddSingleton(config);              // そのまま登録

// インターフェイス経由で参照したい場合
// services.AddSingleton<IAppConfig>(config);

using var provider = services.BuildServiceProvider();

var app = provider.GetRequiredService<App>();
```

*   `AddSingleton<TService>(TService implementationInstance)` を使います。
*   この方法では **コンテナがインスタンスを作っていないため、IDisposable でもコンテナは Dispose しません**。
    *   破棄が必要なら、**自分でライフサイクル管理**してください（`using`, アプリ終了時に明示 Dispose など）。

***

## 2) ファクトリで登録（コンテナに所有させたい / 依存解決が必要）

> **用途**: コンストラクタ引数に他サービスを注入したい、または**コンテナに破棄を任せたい**  
> **Dispose**: **コンテナが所有し、破棄します**

```csharp
services.AddSingleton<Config>(sp =>
{
    // 他サービス/設定から組み立て可能
    var env = sp.GetRequiredService<IHostEnvironment>();
    var value = env.IsDevelopment() ? "Dev" : "Prod";
    return new Config { Value = value };
});
```

*   `AddSingleton<TService>(Func<IServiceProvider, TService>)` を使うと、**他サービスを使って組み立て**られます。
*   この方法で生成されたインスタンスは **コンテナが所有**し、`ServiceProvider.Dispose()` 時に**自動で Dispose**されます。

> ?? **注意（DI の原則）**  
> Singleton のファクトリ内で **Scoped サービスを解決してキャプチャしない**でください（ライフタイム不整合）。どうしても必要なら、`Func<TScoped>` を注入して **毎回スコープ内で解決**する設計に。

***

## 3) 1つのインスタンスを複数サービスにマップする

> **用途**: 実体は 1 つだが、\*\*複数のサービス型（インターフェイス）\*\*で解決したい

```csharp
var core = new CoreService();
services.AddSingleton(core);
services.AddSingleton<ICoreReader>(sp => sp.GetRequiredService<CoreService>());
services.AddSingleton<ICoreWriter>(sp => sp.GetRequiredService<CoreService>());
```

*   まずコンクリートを登録し、他のサービス型は **既存の登録を引き回す**形で同一インスタンスを返します。

***

## 4) 既に `provider` を作った後は登録できない

*   `BuildServiceProvider()` **後**にサービスを追加することはできません。  
    → 追加が必要なら **最初から登録**し直して `ServiceProvider` を作り直すか、**別のコンテナ/スコープ**を用意してください。

***
