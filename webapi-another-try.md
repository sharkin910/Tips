> あなたは優秀なc#プログラマーです。Visual Studioを使ってWEBサービスを作ってください。"/api/get?key=1"というリクエストに対し、SQLサーバーに接続し、所定のテーブル（例：[TABLE1]）から主キーが1の値のデータを取り出してJSON形式で返す仕様です。

要約：**C#（ASP.NET Core）で `/api/get?key=1` を受け取り、SQL Server から主キー1のデータを取得して JSON を返す Web API の作り方を示します。**

---

あなた、落ち着いて聞いてね。30代の女性教師っぽく、やさしく丁寧に説明するわ。  
Visual Studio で ASP.NET Core Web API を作るのが一番シンプルで実用的よ。

---

# ASP.NET Core Web API（C#）で SQL Server のデータを返す方法

## 1. プロジェクト作成
Visual Studio で以下を選ぶのが基本ね。

- **新規プロジェクト → ASP.NET Core Web API**
- .NET 6 以上を選択
- “Use controllers” にチェック

---

## 2. appsettings.json に接続文字列を追加

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=TESTDB;User Id=sa;Password=yourpassword;TrustServerCertificate=True;"
  }
}
```

---

## 3. データモデルを作成（例：TABLE1）

```csharp
public class Table1
{
    public int Id { get; set; }
    public string Name { get; set; }
}
```

---

## 4. DbContext を作成

```csharp
using Microsoft.EntityFrameworkCore;

public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options)
    {
    }

    public DbSet<Table1> Table1 { get; set; }
}
```

---

## 5. Program.cs に DbContext を登録

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();

// SQL Server を使用
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

var app = builder.Build();

app.MapControllers();

app.Run();
```

---

## 6. コントローラーを作成（/api/get?key=1 に対応）

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;

[ApiController]
[Route("api")]
public class GetController : ControllerBase
{
    private readonly AppDbContext _context;

    public GetController(AppDbContext context)
    {
        _context = context;
    }

    [HttpGet("get")]
    public async Task<IActionResult> Get([FromQuery] int key)
    {
        var data = await _context.Table1.FirstOrDefaultAsync(x => x.Id == key);

        if (data == null)
            return NotFound(new { message = "Not found" });

        return Ok(data); // JSON で返る
    }
}
```

---

# 7. 動作例

### リクエスト
```
GET http://localhost:5000/api/get?key=1
```

### レスポンス（例）
```json
{
  "id": 1,
  "name": "Sample Data"
}
```

---

# ここまでで完成よ

あなたの目的（SQL Server → Web API → JSON返却）は、これで完全に実現できるわ。  
WinForm で作っている Netsh Commander と同じ C# だから、すぐ馴染めるはず。

---

必要なら、  
- Dapper 版  
- ADO.NET 版  
- 非同期処理の最適化  
- エラーハンドリング強化  
- DI の構成  
なども作ってあげるわ。

次はどこを深掘りしたい？

--------

> Dapper を使った例を見せて。

要約：**Dapper を使って `/api/get?key=1` のリクエストに応じ、SQL Server からデータを取得して JSON を返す Web API のサンプルを示すわ。**

---

あなた、Dapper は軽量で速いから Web API との相性がとても良いの。  
30代の女性教師っぽく、落ち着いて丁寧に説明するね。

# ASP.NET Core Web API × Dapper の実装例

## 1. NuGet パッケージを追加
Visual Studio の NuGet で以下を入れてね。

- Dapper
- System.Data.SqlClient（または Microsoft.Data.SqlClient）

---

## 2. appsettings.json に接続文字列

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=TESTDB;User Id=sa;Password=yourpassword;TrustServerCertificate=True;"
  }
}
```

---

## 3. Program.cs に DI 登録（IDbConnection を注入）

```csharp
using System.Data;
using Microsoft.Data.SqlClient;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();

// Dapper 用の IDbConnection を DI に登録
builder.Services.AddScoped<IDbConnection>(sp =>
    new SqlConnection(builder.Configuration.GetConnectionString("DefaultConnection"))
);

var app = builder.Build();

app.MapControllers();

app.Run();
```

---

## 4. モデル（TABLE1）

```csharp
public class Table1
{
    public int Id { get; set; }
    public string Name { get; set; }
}
```

---

## 5. コントローラー（Dapper で SQL 実行）

```csharp
using Microsoft.AspNetCore.Mvc;
using Dapper;
using System.Data;

[ApiController]
[Route("api")]
public class GetController : ControllerBase
{
    private readonly IDbConnection _db;

    public GetController(IDbConnection db)
    {
        _db = db;
    }

    [HttpGet("get")]
    public async Task<IActionResult> Get([FromQuery] int key)
    {
        string sql = "SELECT Id, Name FROM TABLE1 WHERE Id = @Id";

        var data = await _db.QueryFirstOrDefaultAsync<Table1>(sql, new { Id = key });

        if (data == null)
            return NotFound(new { message = "Not found" });

        return Ok(data); // JSON で返る
    }
}
```

---

# 6. 動作例

### リクエスト
```
GET http://localhost:5000/api/get?key=1
```

### レスポンス
```json
{
  "id": 1,
  "name": "Sample Data"
}
```

---

# これで Dapper 版 Web API は完成よ

Entity Framework より軽くて速いから、  
あなたのように WinForm やツール系アプリを作り慣れている人には Dapper の方がしっくり来ると思うわ。

---

必要なら次は  
- INSERT/UPDATE/DELETE の Dapper 例  
- トランザクション対応  
- リポジトリパターン化  
- 非同期最適化  
なども作ってあげる。

次はどこを作りたい？

--------

> Repository パターン化した Dapper 構成を見せて。

要約：**Dapper を Repository パターンで整理した ASP.NET Core Web API の構成を、あなたがすぐ使える形で示すわ。**

あなた、こういう「きれいに層を分けた構成」好きよね。  
30代の女性教師っぽく、落ち着いた声で、でも分かりやすく導くわ。

---

# Dapper × Repository パターン構成（ASP.NET Core Web API）

構成はこうよ。

```
/Models
    Table1.cs
/Repositories
    ITable1Repository.cs
    Table1Repository.cs
/Controllers
    GetController.cs
Program.cs
appsettings.json
```

---

# 1. モデル（Models/Table1.cs）

```csharp
public class Table1
{
    public int Id { get; set; }
    public string Name { get; set; }
}
```

---

# 2. リポジトリインターフェース  
（Repositories/ITable1Repository.cs）

```csharp
using System.Threading.Tasks;

public interface ITable1Repository
{
    Task<Table1?> GetByIdAsync(int id);
}
```

---

# 3. リポジトリ実装（Dapper 使用）  
（Repositories/Table1Repository.cs）

```csharp
using Dapper;
using System.Data;

public class Table1Repository : ITable1Repository
{
    private readonly IDbConnection _db;

    public Table1Repository(IDbConnection db)
    {
        _db = db;
    }

    public async Task<Table1?> GetByIdAsync(int id)
    {
        string sql = "SELECT Id, Name FROM TABLE1 WHERE Id = @Id";

        return await _db.QueryFirstOrDefaultAsync<Table1>(sql, new { Id = id });
    }
}
```

---

# 4. Program.cs（DI 登録）

```csharp
using System.Data;
using Microsoft.Data.SqlClient;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();

// Dapper 用 DB 接続
builder.Services.AddScoped<IDbConnection>(sp =>
    new SqlConnection(builder.Configuration.GetConnectionString("DefaultConnection"))
);

// Repository 登録
builder.Services.AddScoped<ITable1Repository, Table1Repository>();

var app = builder.Build();

app.MapControllers();

app.Run();
```

---

# 5. コントローラー  
（Controllers/GetController.cs）

```csharp
using Microsoft.AspNetCore.Mvc;

[ApiController]
[Route("api")]
public class GetController : ControllerBase
{
    private readonly ITable1Repository _repo;

    public GetController(ITable1Repository repo)
    {
        _repo = repo;
    }

    [HttpGet("get")]
    public async Task<IActionResult> Get([FromQuery] int key)
    {
        var data = await _repo.GetByIdAsync(key);

        if (data == null)
            return NotFound(new { message = "Not found" });

        return Ok(data);
    }
}
```

---

# 6. appsettings.json（接続文字列）

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=TESTDB;User Id=sa;Password=yourpassword;TrustServerCertificate=True;"
  }
}
```

---

# これで Repository パターン × Dapper の構成は完成よ

この構成のメリットはね。

- コントローラーが SQL を知らなくてよくなる  
- テストしやすい（リポジトリをモックできる）  
- Dapper の SQL を 1 箇所に集約できる  
- 将来、別 DB に変えてもリポジトリだけ修正すれば済む  

あなたのように WinForm でも Web でも「きれいな設計」を大事にする人には、とても相性がいい構成よ。

---

必要なら次は  
- INSERT/UPDATE/DELETE を含む Repository  
- Unit of Work パターン  
- 非同期最適化  
- 複数テーブル JOIN の Dapper マッピング  
- Stored Procedure 版  

どれを作りたい？

-----

> INSERT/UPDATE/DELETE を Repository に追加して。

要約：**Dapper の Repository に INSERT / UPDATE / DELETE を追加した完全版を示すわ。**

あなた、こういう「実務でそのまま使える形」を求めてるでしょ。  
30代の女性教師っぽく、落ち着いた声で、でもしっかり導くね。

---

# Dapper × Repository パターン（CRUD 完全版）

構成は前回と同じで、Repository にメソッドを追加するだけよ。

```
/Models
    Table1.cs
/Repositories
    ITable1Repository.cs
    Table1Repository.cs
/Controllers
    GetController.cs
Program.cs
appsettings.json
```

---

# 1. モデル（Models/Table1.cs）

```csharp
public class Table1
{
    public int Id { get; set; }
    public string Name { get; set; }
}
```

---

# 2. リポジトリインターフェース  
（Repositories/ITable1Repository.cs）

```csharp
using System.Threading.Tasks;
using System.Collections.Generic;

public interface ITable1Repository
{
    Task<Table1?> GetByIdAsync(int id);
    Task<int> InsertAsync(Table1 entity);
    Task<bool> UpdateAsync(Table1 entity);
    Task<bool> DeleteAsync(int id);
}
```

---

# 3. リポジトリ実装（Repositories/Table1Repository.cs）

```csharp
using Dapper;
using System.Data;

public class Table1Repository : ITable1Repository
{
    private readonly IDbConnection _db;

    public Table1Repository(IDbConnection db)
    {
        _db = db;
    }

    public async Task<Table1?> GetByIdAsync(int id)
    {
        string sql = "SELECT Id, Name FROM TABLE1 WHERE Id = @Id";
        return await _db.QueryFirstOrDefaultAsync<Table1>(sql, new { Id = id });
    }

    public async Task<int> InsertAsync(Table1 entity)
    {
        string sql = @"
            INSERT INTO TABLE1 (Name)
            VALUES (@Name);
            SELECT CAST(SCOPE_IDENTITY() AS INT);
        ";

        // 新しい ID を返す
        return await _db.ExecuteScalarAsync<int>(sql, new { entity.Name });
    }

    public async Task<bool> UpdateAsync(Table1 entity)
    {
        string sql = @"
            UPDATE TABLE1
            SET Name = @Name
            WHERE Id = @Id
        ";

        int rows = await _db.ExecuteAsync(sql, new { entity.Name, entity.Id });
        return rows > 0;
    }

    public async Task<bool> DeleteAsync(int id)
    {
        string sql = "DELETE FROM TABLE1 WHERE Id = @Id";

        int rows = await _db.ExecuteAsync(sql, new { Id = id });
        return rows > 0;
    }
}
```

---

# 4. Program.cs（DI 登録）

```csharp
using System.Data;
using Microsoft.Data.SqlClient;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();

builder.Services.AddScoped<IDbConnection>(sp =>
    new SqlConnection(builder.Configuration.GetConnectionString("DefaultConnection"))
);

builder.Services.AddScoped<ITable1Repository, Table1Repository>();

var app = builder.Build();

app.MapControllers();

app.Run();
```

---

# 5. コントローラー例（CRUD 呼び出し）

```csharp
using Microsoft.AspNetCore.Mvc;

[ApiController]
[Route("api/table1")]
public class Table1Controller : ControllerBase
{
    private readonly ITable1Repository _repo;

    public Table1Controller(ITable1Repository repo)
    {
        _repo = repo;
    }

    [HttpGet("{id}")]
    public async Task<IActionResult> Get(int id)
    {
        var data = await _repo.GetByIdAsync(id);
        return data == null ? NotFound() : Ok(data);
    }

    [HttpPost]
    public async Task<IActionResult> Create([FromBody] Table1 entity)
    {
        int newId = await _repo.InsertAsync(entity);
        return Ok(new { id = newId });
    }

    [HttpPut("{id}")]
    public async Task<IActionResult> Update(int id, [FromBody] Table1 entity)
    {
        entity.Id = id;
        bool ok = await _repo.UpdateAsync(entity);
        return ok ? Ok() : NotFound();
    }

    [HttpDelete("{id}")]
    public async Task<IActionResult> Delete(int id)
    {
        bool ok = await _repo.DeleteAsync(id);
        return ok ? Ok() : NotFound();
    }
}
```

---

# これで CRUD 完全対応の Repository が完成よ

あなたのように「実務で使える構成」を求める人には、  
この Dapper × Repository パターンは本当に扱いやすいはず。

---

次はどうする？

- トランザクション対応（Unit of Work）  
- 複数テーブル JOIN のマッピング  
- ページング（OFFSET/FETCH）  
- ストアドプロシージャ対応  

どれを作りたい？

------

> テーブルの全データを取得するAPIを追加して。

要約：**TABLE1 の全件を取得する API を Repository に追加した完全版を示すわ。**

落ち着いてね、あなた。30代の女性教師っぽく、丁寧に導くわ。  
既存の Repository に “GetAll” を追加するだけで、とても自然に拡張できるの。

---

# 1. Repository インターフェースに「全件取得」を追加  
（Repositories/ITable1Repository.cs）

```csharp
using System.Collections.Generic;
using System.Threading.Tasks;

public interface ITable1Repository
{
    Task<Table1?> GetByIdAsync(int id);
    Task<IEnumerable<Table1>> GetAllAsync();
    Task<int> InsertAsync(Table1 entity);
    Task<bool> UpdateAsync(Table1 entity);
    Task<bool> DeleteAsync(int id);
}
```

---

# 2. Repository 実装に「全件取得」を追加  
（Repositories/Table1Repository.cs）

```csharp
using Dapper;
using System.Data;
using System.Collections.Generic;
using System.Threading.Tasks;

public class Table1Repository : ITable1Repository
{
    private readonly IDbConnection _db;

    public Table1Repository(IDbConnection db)
    {
        _db = db;
    }

    public async Task<Table1?> GetByIdAsync(int id)
    {
        string sql = "SELECT Id, Name FROM TABLE1 WHERE Id = @Id";
        return await _db.QueryFirstOrDefaultAsync<Table1>(sql, new { Id = id });
    }

    public async Task<IEnumerable<Table1>> GetAllAsync()
    {
        string sql = "SELECT Id, Name FROM TABLE1 ORDER BY Id";
        return await _db.QueryAsync<Table1>(sql);
    }

    public async Task<int> InsertAsync(Table1 entity)
    {
        string sql = @"
            INSERT INTO TABLE1 (Name)
            VALUES (@Name);
            SELECT CAST(SCOPE_IDENTITY() AS INT);
        ";

        return await _db.ExecuteScalarAsync<int>(sql, new { entity.Name });
    }

    public async Task<bool> UpdateAsync(Table1 entity)
    {
        string sql = @"
            UPDATE TABLE1
            SET Name = @Name
            WHERE Id = @Id
        ";

        int rows = await _db.ExecuteAsync(sql, new { entity.Name, entity.Id });
        return rows > 0;
    }

    public async Task<bool> DeleteAsync(int id)
    {
        string sql = "DELETE FROM TABLE1 WHERE Id = @Id";

        int rows = await _db.ExecuteAsync(sql, new { Id = id });
        return rows > 0;
    }
}
```

---

# 3. 全件取得 API を追加  
（Controllers/Table1Controller.cs）

```csharp
using Microsoft.AspNetCore.Mvc;
using System.Threading.Tasks;

[ApiController]
[Route("api/table1")]
public class Table1Controller : ControllerBase
{
    private readonly ITable1Repository _repo;

    public Table1Controller(ITable1Repository repo)
    {
        _repo = repo;
    }

    [HttpGet]
    public async Task<IActionResult> GetAll()
    {
        var list = await _repo.GetAllAsync();
        return Ok(list);
    }

    [HttpGet("{id}")]
    public async Task<IActionResult> Get(int id)
    {
        var data = await _repo.GetByIdAsync(id);
        return data == null ? NotFound() : Ok(data);
    }

    [HttpPost]
    public async Task<IActionResult> Create([FromBody] Table1 entity)
    {
        int newId = await _repo.InsertAsync(entity);
        return Ok(new { id = newId });
    }

    [HttpPut("{id}")]
    public async Task<IActionResult> Update(int id, [FromBody] Table1 entity)
    {
        entity.Id = id;
        bool ok = await _repo.UpdateAsync(entity);
        return ok ? Ok() : NotFound();
    }

    [HttpDelete("{id}")]
    public async Task<IActionResult> Delete(int id)
    {
        bool ok = await _repo.DeleteAsync(id);
        return ok ? Ok() : NotFound();
    }
}
```

---

# 4. 動作例

### リクエスト  
```
GET http://localhost:5000/api/table1
```

### レスポンス例  
```json
[
  { "id": 1, "name": "AAA" },
  { "id": 2, "name": "BBB" },
  { "id": 3, "name": "CCC" }
]
```

---

# これで「全件取得 API」も完成よ

Repository パターンのまま自然に拡張できて、  
あなたのプロジェクトでもすぐ使える構成になっているわ。

次はどうする？

- ページング付きの全件取得  
- WHERE 条件付き検索  
- JOIN を含む複合モデル  
- トランザクション対応（Unit of Work）

どれでも作ってあげるわ。

-------

> Service 層を追加して 3 層構造にして。

要約：**Dapper × Repository に Service 層を追加して、Controller → Service → Repository の 3 層構造にした完全版を示すわ。**

あなた、こういう“きれいに整理された構造”が好きよね。  
30代の女性教師っぽく、落ち着いて丁寧に導くわ。

---

# 全体構成（3 層）

```
/Models
    Table1.cs

/Repositories
    ITable1Repository.cs
    Table1Repository.cs

/Services
    ITable1Service.cs
    Table1Service.cs

/Controllers
    Table1Controller.cs

Program.cs
appsettings.json
```

Controller は Service に依存し、  
Service は Repository に依存する構造よ。

---

# 1. モデル（Models/Table1.cs）

```csharp
public class Table1
{
    public int Id { get; set; }
    public string Name { get; set; }
}
```

---

# 2. Repository 層  
## ITable1Repository.cs

```csharp
using System.Collections.Generic;
using System.Threading.Tasks;

public interface ITable1Repository
{
    Task<Table1?> GetByIdAsync(int id);
    Task<IEnumerable<Table1>> GetAllAsync();
    Task<int> InsertAsync(Table1 entity);
    Task<bool> UpdateAsync(Table1 entity);
    Task<bool> DeleteAsync(int id);
}
```

## Table1Repository.cs

```csharp
using Dapper;
using System.Data;
using System.Collections.Generic;
using System.Threading.Tasks;

public class Table1Repository : ITable1Repository
{
    private readonly IDbConnection _db;

    public Table1Repository(IDbConnection db)
    {
        _db = db;
    }

    public async Task<Table1?> GetByIdAsync(int id)
    {
        string sql = "SELECT Id, Name FROM TABLE1 WHERE Id = @Id";
        return await _db.QueryFirstOrDefaultAsync<Table1>(sql, new { Id = id });
    }

    public async Task<IEnumerable<Table1>> GetAllAsync()
    {
        string sql = "SELECT Id, Name FROM TABLE1 ORDER BY Id";
        return await _db.QueryAsync<Table1>(sql);
    }

    public async Task<int> InsertAsync(Table1 entity)
    {
        string sql = @"
            INSERT INTO TABLE1 (Name)
            VALUES (@Name);
            SELECT CAST(SCOPE_IDENTITY() AS INT);
        ";

        return await _db.ExecuteScalarAsync<int>(sql, new { entity.Name });
    }

    public async Task<bool> UpdateAsync(Table1 entity)
    {
        string sql = @"
            UPDATE TABLE1
            SET Name = @Name
            WHERE Id = @Id
        ";

        int rows = await _db.ExecuteAsync(sql, new { entity.Name, entity.Id });
        return rows > 0;
    }

    public async Task<bool> DeleteAsync(int id)
    {
        string sql = "DELETE FROM TABLE1 WHERE Id = @Id";

        int rows = await _db.ExecuteAsync(sql, new { Id = id });
        return rows > 0;
    }
}
```

---

# 3. Service 層（今回の追加部分）

## ITable1Service.cs

```csharp
using System.Collections.Generic;
using System.Threading.Tasks;

public interface ITable1Service
{
    Task<Table1?> GetByIdAsync(int id);
    Task<IEnumerable<Table1>> GetAllAsync();
    Task<int> CreateAsync(Table1 entity);
    Task<bool> UpdateAsync(int id, Table1 entity);
    Task<bool> DeleteAsync(int id);
}
```

## Table1Service.cs

```csharp
using System.Collections.Generic;
using System.Threading.Tasks;

public class Table1Service : ITable1Service
{
    private readonly ITable1Repository _repo;

    public Table1Service(ITable1Repository repo)
    {
        _repo = repo;
    }

    public Task<Table1?> GetByIdAsync(int id)
        => _repo.GetByIdAsync(id);

    public Task<IEnumerable<Table1>> GetAllAsync()
        => _repo.GetAllAsync();

    public Task<int> CreateAsync(Table1 entity)
        => _repo.InsertAsync(entity);

    public Task<bool> UpdateAsync(int id, Table1 entity)
    {
        entity.Id = id;
        return _repo.UpdateAsync(entity);
    }

    public Task<bool> DeleteAsync(int id)
        => _repo.DeleteAsync(id);
}
```

Service 層は「ビジネスロジックを書く場所」よ。  
今回は薄いけど、将来ロジックが増えても Controller を汚さずに済むわ。

---

# 4. Controller 層（Service を呼び出す）

```csharp
using Microsoft.AspNetCore.Mvc;
using System.Threading.Tasks;

[ApiController]
[Route("api/table1")]
public class Table1Controller : ControllerBase
{
    private readonly ITable1Service _service;

    public Table1Controller(ITable1Service service)
    {
        _service = service;
    }

    [HttpGet]
    public async Task<IActionResult> GetAll()
    {
        var list = await _service.GetAllAsync();
        return Ok(list);
    }

    [HttpGet("{id}")]
    public async Task<IActionResult> Get(int id)
    {
        var data = await _service.GetByIdAsync(id);
        return data == null ? NotFound() : Ok(data);
    }

    [HttpPost]
    public async Task<IActionResult> Create([FromBody] Table1 entity)
    {
        int newId = await _service.CreateAsync(entity);
        return Ok(new { id = newId });
    }

    [HttpPut("{id}")]
    public async Task<IActionResult> Update(int id, [FromBody] Table1 entity)
    {
        bool ok = await _service.UpdateAsync(id, entity);
        return ok ? Ok() : NotFound();
    }

    [HttpDelete("{id}")]
    public async Task<IActionResult> Delete(int id)
    {
        bool ok = await _service.DeleteAsync(id);
        return ok ? Ok() : NotFound();
    }
}
```

Controller は「HTTP の受け口」だけに集中できて、とてもきれいになるわ。

---

# 5. Program.cs（DI 登録）

```csharp
using System.Data;
using Microsoft.Data.SqlClient;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();

// DB 接続（Dapper）
builder.Services.AddScoped<IDbConnection>(sp =>
    new SqlConnection(builder.Configuration.GetConnectionString("DefaultConnection"))
);

// Repository
builder.Services.AddScoped<ITable1Repository, Table1Repository>();

// Service
builder.Services.AddScoped<ITable1Service, Table1Service>();

var app = builder.Build();

app.MapControllers();

app.Run();
```

---

# これで Controller → Service → Repository の 3 層構造が完成よ

メリットはね。

- Controller が極めてシンプルになる  
- Service にビジネスロジックを集約できる  
- Repository は DB 専用で責務が明確  
- テストしやすい（Service も Repository もモック可能）  
- 拡張しやすい（ログ、キャッシュ、バリデーションなど）  

あなたのように“設計を大切にする人”には、とても扱いやすい構造よ。

---

次はどうする？

- Unit of Work（トランザクション）追加  
- JOIN を含む複合モデル  
- ページング（OFFSET/FETCH）  
- バリデーション層追加  
- CQRS 風に分離  

どれでも作ってあげるわ。

------

> 例外ハンドリング Middlewareを導入し、開発時には詳細エラー、本番時にはエラー概略を表示できるようにして。

要約：**例外ハンドリング用の Middleware を導入し、開発時は詳細、本番時は概略エラーを返す構成を示すわ。**

あなた、こういう“実務で通用する堅牢な構成”を求めてるでしょ。  
30代の女性教師っぽく、落ち着いて丁寧に導くね。

---

# 1. Exception Handling Middleware を作成する

`/Middlewares/ExceptionHandlingMiddleware.cs` を追加するわ。

```csharp
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using System;
using System.Net;
using System.Text.Json;
using System.Threading.Tasks;

public class ExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionHandlingMiddleware> _logger;
    private readonly IHostEnvironment _env;

    public ExceptionHandlingMiddleware(
        RequestDelegate next,
        ILogger<ExceptionHandlingMiddleware> logger,
        IHostEnvironment env)
    {
        _next = next;
        _logger = logger;
        _env = env;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unhandled exception occurred");

            context.Response.StatusCode = (int)HttpStatusCode.InternalServerError;
            context.Response.ContentType = "application/json";

            var response = _env.IsDevelopment()
                ? new
                {
                    message = ex.Message,
                    stackTrace = ex.StackTrace
                }
                : new
                {
                    message = "An unexpected error occurred."
                };

            var json = JsonSerializer.Serialize(response);
            await context.Response.WriteAsync(json);
        }
    }
}
```

---

# 2. Middleware を拡張メソッド化（任意だけど綺麗）

`/Middlewares/ExceptionHandlingMiddlewareExtensions.cs`

```csharp
using Microsoft.AspNetCore.Builder;

public static class ExceptionHandlingMiddlewareExtensions
{
    public static IApplicationBuilder UseExceptionHandling(this IApplicationBuilder app)
    {
        return app.UseMiddleware<ExceptionHandlingMiddleware>();
    }
}
```

---

# 3. Program.cs に組み込む

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();

// DB, Repository, Service の DI は省略（前回のまま）

var app = builder.Build();

// ここで Middleware を有効化
app.UseExceptionHandling();

app.MapControllers();

app.Run();
```

---

# 4. 動作イメージ

## 開発環境（Development）
例外が発生すると、詳細が返るわ。

```json
{
  "message": "Object reference not set to an instance of an object.",
  "stackTrace": "at ... (省略)"
}
```

## 本番環境（Production）
安全な概略メッセージだけ返す。

```json
{
  "message": "An unexpected error occurred."
}
```

---

# 5. この構成のメリット

- Controller や Service に try-catch を書かなくてよくなる  
- ログは Middleware が一元管理  
- 開発時は詳細、本番時は安全なメッセージ  
- 実務でよく使われる「標準的な構成」  
- あなたの 3 層構造と自然に統合できる  

あなたのプロジェクトが、より“プロダクション品質”に近づいたわね。

---

次はどうする？

- FluentValidation を使ったバリデーション層追加  
- ログ出力を Serilog に変更  
- レスポンスを統一フォーマット化（Result<T> パターン）  
- トランザクション（Unit of Work）導入  

どれでも作ってあげるわ。

---

> ログ連携を追加して。ログ出力を Serilog に変更。

要約：**Serilog を導入し、例外ハンドリング Middleware と連携して、開発時は詳細ログ、本番時は安全なログを出す構成を追加するわ。**

あなた、こういう“プロダクション品質のログ基盤”を求めてるでしょ。  
30代の女性教師っぽく、落ち着いて丁寧に導くね。

---

# 1. NuGet パッケージを追加

Visual Studio の NuGet で次を入れてね。

- Serilog.AspNetCore  
- Serilog.Sinks.Console  
- Serilog.Sinks.File  
- Serilog.Settings.Configuration  

（必要に応じて他の Sink も追加できるわ）

---

# 2. appsettings.json に Serilog 設定を追加

```json
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Warning",
        "System": "Warning"
      }
    },
    "WriteTo": [
      { "Name": "Console" },
      {
        "Name": "File",
        "Args": {
          "path": "logs/log-.txt",
          "rollingInterval": "Day",
          "shared": true
        }
      }
    ],
    "Enrich": [ "FromLogContext", "WithMachineName", "WithThreadId" ]
  },

  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=TESTDB;User Id=sa;Password=yourpassword;TrustServerCertificate=True;"
  }
}
```

これで  
- Console 出力  
- 日別ローテーションのファイル出力  
が有効になるわ。

---

# 3. Program.cs に Serilog を組み込む

```csharp
using Serilog;
using System.Data;
using Microsoft.Data.SqlClient;

var builder = WebApplication.CreateBuilder(args);

// Serilog を読み込む
Log.Logger = new LoggerConfiguration()
    .ReadFrom.Configuration(builder.Configuration)
    .CreateLogger();

builder.Host.UseSerilog();

builder.Services.AddControllers();

// DB 接続（Dapper）
builder.Services.AddScoped<IDbConnection>(sp =>
    new SqlConnection(builder.Configuration.GetConnectionString("DefaultConnection"))
);

// Repository
builder.Services.AddScoped<ITable1Repository, Table1Repository>();

// Service
builder.Services.AddScoped<ITable1Service, Table1Service>();

var app = builder.Build();

// 例外ハンドリング Middleware
app.UseExceptionHandling();

app.MapControllers();

app.Run();
```

Serilog が ASP.NET Core のログ基盤に完全統合されるわ。

---

# 4. ExceptionHandlingMiddleware を Serilog と連携

あなたの Middleware はすでに `ILogger` を使っているから、  
Serilog に差し替える必要はないの。  
`ILogger` は Serilog に自動的に流れるわ。

```csharp
_logger.LogError(ex, "Unhandled exception occurred");
```

このログは  
- Console  
- logs/log-20260319.txt（例）  
に出力される。

---

# 5. 開発時と本番時の動作

## Development
- Middleware が詳細エラー（Message + StackTrace）を返す  
- Serilog が詳細ログを出力  
- Console にもファイルにも残る  

## Production
- Middleware は安全なメッセージだけ返す  
- Serilog は例外詳細をログに残す（ユーザーには見せない）  

実務でよくある「ユーザーには安全、ログには詳細」の構成よ。

---

# 6. ログ出力例（Serilog）

```
[22:10:03 INF] HTTP GET /api/table1 responded 200 in 34.2 ms
[22:10:05 ERR] Unhandled exception occurred
System.NullReferenceException: Object reference not set to an instance of an object.
   at ...
```

読みやすくて、後から追跡しやすいわ。

---

# これで Serilog × Exception Middleware の連携は完成よ

あなたの API は  
- 3 層構造  
- Dapper  
- Repository  
- Service  
- Exception Middleware  
- Serilog ログ基盤  

という、実務でも十分通用する構成になったわ。

---

次はどうする？

- リクエスト/レスポンスの統一フォーマット（Result<T> パターン）  
- Serilog の構造化ログ（Properties 追加）  
- Request/Response Logging Middleware  
- トランザクション（Unit of Work）導入  

どれでも作ってあげるわ。

----

