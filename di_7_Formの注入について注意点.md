DIですべてのFormを登録しておく方法を取ったら、使用しないケースでも芋づる式に生成されてしまう無駄が発生しますが、なにか解決策は？
===========

おっしゃる通り、コンストラクター注入（Constructor Injection）だけで突き進むと、アプリ起動時に「一度も開かないかもしれないサブ画面」まで依存関係が解決され、インスタンス化の連鎖（オブジェクトグラフの肥大化）が起きてしまいます。

WinFormsでこの「芋づる式生成」を回避するための現実的な解決策は3つあります。

---

### 1. `IServiceProvider` を直接注入する（サービスロケーター的アプローチ）
最もシンプルで確実な方法です。親フォームには「フォームそのもの」ではなく「コンテナ」を渡し、**必要な瞬間（ボタンクリック時など）にだけ** 取り出します。

```csharp
public partial class MainForm : Form
{
    private readonly IServiceProvider _serviceProvider;

    public MainForm(IServiceProvider serviceProvider) : this()
    {
        _serviceProvider = serviceProvider;
    }

    private void btnOpenSubForm_Click(object sender, EventArgs e)
    {
        // 呼び出された瞬間にはじめて SubForm が生成される
        var subForm = _serviceProvider.GetRequiredService<SubForm>();
        subForm.Show();
    }
}
```

### 2. `Func<T>` などの Factory を利用する
`IServiceProvider` を直接振り回すのが「サービスロケーター・アンチパターン」として気になる場合は、**Factory（工場）** を注入します。

MS DI は標準で `Func<T>` を自動解決してくれませんが、登録時に少し工夫するか、自作の Factory クラスを介すとスマートです。

```csharp
// サービス登録側
services.AddTransient<SubForm>();
services.AddSingleton<Func<SubForm>>(x => () => x.GetRequiredService<SubForm>());

// フォーム側
public MainForm(Func<SubForm> subFormFactory) : this()
{
    _subFormFactory = subFormFactory;
}

private void btnOpen_Click(object sender, EventArgs e)
{
    var subForm = _subFormFactory(); // 実行時まで生成を遅延できる
    subForm.Show();
}
```

---

### まとめ：どれを使うべき？

* **小～中規模:** `IServiceProvider` を注入して `GetRequiredService<T>` するのが一番手っ取り早く、コードも煩雑になりません。ただしアンチパターンなのでFormに限るほうが良いかも。
* **大規模・厳格な設計:** 自作の `IFormFactory` インターフェースを作り、そこで生成ロジックを隠蔽するのが「依存性の注入」の原則に最も忠実です。

> [!NOTE]
> ちなみに、`AddTransient<Form>()` で登録していれば、**「使わなければ生成されない（＝インスタンス化されない）」** ので、登録自体がメモリを圧迫することはありません。問題なのは「親のコンストラクターで子のインスタンスを要求してしまうこと」ですので、上記のように **「取得のタイミングを遅らせる」** だけで解決します！

