グローバル変数廃止案
====================

グローバル変数を多用している既存 C# コードを、**大規模改修なしで安全・整理された構造に近づける**ための現実的な改善策をまとめます。  
ポイントは「**取得コストの高い処理は避ける**」「**既存コードを壊さない**」「**段階的に改善できる**」ことです。

---

## 最初に押さえるべき方向性
グローバル変数の問題は主に次の3つです。

- 依存関係が見えにくく、バグの温床になる  
- テストがしにくい  
- 変更に弱い構造になる  

ただし、既存コードを大きく変えずに改善するには、**「完全なDI化」や「アーキテクチャ刷新」ではなく、段階的なラッピングと責務の分離**が現実的です。

---

## 改善策 1：グローバル変数を「静的プロパティを持つ管理クラス」に集約する
### 目的  
散らばったグローバル変数を **1つの静的クラスに集約**し、アクセス場所を統一する。

### メリット  
- 既存コードの `Global.X` を `AppState.X` に置き換えるだけで済む  
- 依存関係が見える化される  
- 後で DI に移行しやすい

### 例
```csharp
public static class AppState
{
    public static UserConfig Config { get; set; }
    public static NetworkStatus CurrentNetwork { get; set; }
    public static string AppVersion { get; set; }
}
```

既存コード側は最小変更で済む：

```csharp
var ip = AppState.CurrentNetwork.IpAddress;
```

---

## 改善策 2：取得コストの高い値は「Lazy<T>」で遅延初期化する
グローバル変数の中に「重い処理で取得する値」がある場合、`Lazy<T>` が最適です。

###メリット  
- 初回アクセス時だけ計算  
- 以降はキャッシュ  
- スレッドセーフ

### 例
```csharp
public static class AppState
{
    private static readonly Lazy<NetworkStatus> _networkStatus =
        new Lazy<NetworkStatus>(() => NetworkScanner.Scan());

    public static NetworkStatus NetworkStatus => _networkStatus.Value;
}
```

---

## 改善策 3：グローバル変数を「読み取り専用」に寄せる
書き換え可能なグローバル変数はバグの原因になりやすいので、**まずは読み取り専用化**を進める。

### 方法  
- `readonly`  
- `private set`  
- 不変クラス（Immutable Object）

### 例
```csharp
public static class AppState
{
    public static UserConfig Config { get; private set; }

    public static void LoadConfig(UserConfig config)
    {
        Config = config;
    }
}
```

---

## 改善策 4：副作用のある処理は「サービスクラス」に切り出す
グローバル変数の中に「処理ロジック」が混ざっている場合は、**サービスクラスに移動**するだけで大幅に整理される。

### 例  
```csharp
public class NetworkService
{
    public void ApplyIpSetting(IpSetting setting)
    {
        // netsh 実行など
    }
}
```

グローバルにはインスタンスだけ置く：

```csharp
public static class Services
{
    public static NetworkService Network { get; } = new NetworkService();
}
```

既存コードはこうなる：

```csharp
Services.Network.ApplyIpSetting(setting);
```

---

## 改善策 5：段階的に DI（依存性注入）へ移行できる構造にする
いきなり DI コンテナ導入は大変なので、まずは「**手動 DI**」に近い形に寄せる。

### 例  
```csharp
public static class App
{
    public static NetworkService NetworkService { get; private set; }

    public static void Initialize()
    {
        NetworkService = new NetworkService();
    }
}
```

後で DI コンテナに置き換えやすい。

---

## 改善策 6：グローバル変数の使用箇所をログで可視化する
改善の順番を決めるために、まず「どこで使われているか」を把握するのが重要。

### 方法  
- Visual Studio の「参照を検索」  
- Roslyn Analyzer で警告を出す  
- 一時的に getter にログを仕込む  

```csharp
private static UserConfig _config;
public static UserConfig Config
{
    get
    {
        Debug.WriteLine("Config accessed");
        return _config;
    }
    set => _config = value;
}
```

---

## 全体の改善ロードマップ（大規模改修なし）
1. **グローバル変数を1つの静的クラスに集約**  
2. **重い処理は Lazy<T> 化してキャッシュ**  
3. **読み取り専用化を進める**  
4. **処理ロジックをサービスクラスに切り出す**  
5. **静的クラスを「手動 DI コンテナ」として整理**  
6. **必要になったら本格的な DI に移行**

この流れなら、既存コードを壊さずに段階的に改善できます。

---
