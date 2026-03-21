# C# と Python の文法比較表（移行者向け）

要点のまとめ：  
**C# → Python へ移行するときに最もつまずきやすいのは「型」「命名規則」「クラス構造」「例外処理」「コレクション操作」あたり。**  

---

## 1. 基本文法の比較

| 項目 | C# | Python |
|------|------|--------|
| 変数宣言 | `int x = 10;` | `x = 10`（型なし） |
| 型 | 静的型付け | 動的型付け |
| 行末 | `;` 必須 | なし |
| ブロック | `{ }` | インデント |
| コメント | `//` / `/* */` | `#` / `""" """` |
| 文字列 | `"text"` | `"text"` or `'text'` |
| 文字 | `'a'`（char 型） | `'a'`（文字列扱い） |

---

## 2. 関数・メソッド

| 項目 | C# | Python |
|------|------|--------|
| 定義 | `void Foo(int x)` | `def foo(x):` |
| 戻り値 | 型必須 | 何でも返せる |
| オーバーロード | 可能 | 不可（デフォルト引数で代替） |
| アクセス修飾子 | `public` など必須 | なし（慣習で `_name`） |

---

## 3. クラス・オブジェクト

| 項目 | C# | Python |
|------|------|--------|
| クラス定義 | `class MyClass {}` | `class MyClass:` |
| コンストラクタ | `MyClass()` | `__init__` |
| フィールド | 型必須 | 何でも追加できる |
| プロパティ | `get; set;` | `@property` |
| インターフェース | `interface` | なし（プロトコル的に実装） |
| 継承 | `:` | `(BaseClass)` |

---

## 4. コレクション

| 種類 | C# | Python |
|------|------|--------|
| 配列 | `int[] arr = {1,2,3};` | `[1, 2, 3]` |
| リスト | `List<int>` | `list` |
| 辞書 | `Dictionary<string,int>` | `dict` |
| セット | `HashSet<int>` | `set` |
| LINQ | 豊富 | `list comprehension` や `map/filter` |

---

## 5. 例外処理

| C# | Python |
|----|--------|
| `try { ... } catch (Exception e) {}` | `try: ... except Exception as e:` |
| `finally` | `finally` |

---

## 6. 命名規則（重要）

| 種類 | C# | Python |
|------|------|--------|
| 変数 | camelCase | snake_case |
| 関数 | PascalCase / camelCase | snake_case |
| クラス | PascalCase | PascalCase |
| 定数 | PascalCase or UPPER_CASE | UPPER_CASE |

---

# C# → Python 移行時の注意点（ここが一番重要）

## 1. **型がないのでバグが潜みやすい**
C# のように型チェックがないため、  
**実行して初めてエラーに気づく** ことが多い。

→ 対策：  
- `mypy` で型チェック  
- `typing` モジュールを使う（`list[int]` など）

---

## 2. **インデントが文法になる**
C# の `{}` の代わりにインデントが必須。

```python
if x > 0:
    print("OK")
```

インデントミスは即エラー。

---

## 3. **オーバーロードがない**
C# のように同名メソッドを複数作れない。

→ 対策：  
- デフォルト引数  
- 可変長引数 `*args`  
- 型チェックで分岐

---

## 4. **アクセス修飾子がない**
`public/private` がないので、慣習で管理。

- `_name` → 内部用  
- `__name` → 名前マングリング（疑似 private）

---

## 5. **プロパティの書き方が違う**
C# の `get; set;` は Python ではデコレータで書く。

```python
class A:
    @property
    def value(self):
        return self._value
```

---

## 6. **例外が多用される文化**
Python は「例外で制御する」文化が強い。

例：ファイル存在チェック

C#  
```csharp
if (File.Exists(path)) { ... }
```

Python  
```python
try:
    open(path)
except FileNotFoundError:
    ...
```

---

## 7. **クラスにフィールドを後付けできてしまう**
Python は柔軟すぎるため、意図しないバグの原因に。

```python
obj.new_field = 123  # 追加できてしまう
```

→ dataclass を使うと安全性が上がる。

---

# まとめ（あなた向けの最適化版）

- **Python は “読みやすさ最優先” の文化**  
- 型がないので **mypy** を使うと C# っぽい安心感が出る  
- 命名規則は **snake_case** が基本  
- クラス構造はシンプルで柔軟だが、逆にバグも入りやすい  
- LINQ の代わりに **リスト内包表記** を使うと気持ちよく書ける  

---
