---
title: "RustのエラーハンドリングをResult/Option/thiserrorで理解する"
emoji: "🚨"
type: "tech"
topics: ["rust"]
published: true
---

# RustのエラーハンドリングをResult/Option/thiserrorで理解する

:::message
**2026/05/17 修正**
`?` 演算子の説明を修正しました（詳細は各箇所を参照）。
:::

Rustには例外（Exception）がありません。その代わりに、エラーは**戻り値**として明示的に扱います。最初は「毎回エラーを処理するのが面倒」と感じますが、慣れると「エラーが起きうる箇所が一目で分かる」という安心感に変わります。

---

## Option — 値がないかもしれない

`Option<T>` は「値があるかもしれないし、ないかもしれない」を表します。他言語の `null` に相当しますが、型で管理するため **ヌルポインタ例外が起きません**。

```rust
enum Option<T> {
    Some(T), // 値がある
    None,    // 値がない
}
```

実際の使い方：

```rust
let v = vec![1, 2, 3];

// インデックスが範囲外かもしれない → Option を返す
let first = v.get(0);   // Some(1)
let tenth = v.get(10);  // None

// match で処理
match v.get(0) {
    Some(val) => println!("値: {}", val),
    None => println!("存在しない"),
}
```

### 便利メソッド

```rust
let opt: Option<i32> = Some(42);

opt.unwrap()              // Some なら値を取り出す、None ならパニック
opt.unwrap_or(0)          // None のとき 0 を使う
opt.unwrap_or_else(|| 0)  // None のとき クロージャを実行
opt.map(|v| v * 2)        // Some の中身を変換 → Some(84)
opt.is_some()             // true
opt.is_none()             // false
```

---

## Result — 成功か失敗か

`Result<T, E>` は「成功なら T、失敗なら E」を表します。

```rust
enum Result<T, E> {
    Ok(T),  // 成功
    Err(E), // 失敗
}
```

ファイルを開く例：

```rust
use std::fs::File;

match File::open("hello.txt") {
    Ok(file) => println!("開けた: {:?}", file),
    Err(e) => println!("エラー: {}", e),
}
```

### ? 演算子 — エラーを伝播する

エラーが起きたら呼び出し元にそのまま返す、というパターンを `?` で簡潔に書けます：

```rust
use std::fs;
use std::io;

fn read_username() -> Result<String, io::Error> {
    let content = fs::read_to_string("username.txt")?; // エラーなら即 return
    Ok(content.trim().to_string())
}
```

`?` は「Ok なら中身を取り出す、Err なら即 return」という糖衣構文です。内部ではこう展開されています：

```rust
// ? を使った書き方
let content = fs::read_to_string("username.txt")?;

// コンパイラが展開するイメージ
let content = match fs::read_to_string("username.txt") {
    Ok(val) => val,                        // 成功 → val を content に束縛して続行
    Err(e)  => return Err(From::from(e)), // 失敗 → From で変換して即 return
};
```

:::message
**2026/05/17 修正**：展開コードに `From::from(e)` を明示しました
:::

つまり `username.txt` が読み込めなかった場合、`read_username` 自体の戻り値が `Err` になります。呼び出し元はこうなります：

```rust
match read_username() {
    Ok(name) => println!("ユーザー名: {}", name),
    Err(e)   => println!("読み込み失敗: {}", e), // ← ファイルが開けなかった場合
}
```

ネストが深くなるのを防げるだけでなく、「`?` が付いている行はエラーが起きうる」とひと目で分かります。

なお `?` は `Option<T>` にも使えます。`None` のとき即 `return None` になるため、`Result` 専用の演算子ではありません。

---

## 独自エラー型と thiserror

### なぜ複数のエラーを1つの型にまとめるのか

`?` は内部で `From::from(e)` を呼んでエラー型を変換します。つまり **発生したエラー型から戻り値のエラー型への `From` 実装が必要**です。

:::message
**2026/05/17 修正**：「型が一致しないといけない」→「`From` 実装が必要」に修正しました
:::

```rust
// ❌ From 実装がなく ? が使えない
fn load_config() -> Result<i32, io::Error> {  // エラー型は io::Error と宣言
    let content = fs::read_to_string("config.txt")?; // io::Error → OK（同じ型なので From は恒等変換）
    let value: i32 = content.trim().parse()?;         // ParseIntError → io::Error への From 実装がない！
    Ok(value)
}
```

ファイル読み込みは `io::Error`、文字列パースは `ParseIntError` と種類が違うため、両方に `?` を使うには全エラーをまとめた1つの型が必要です：

```rust
#[derive(Debug)]
enum AppError {
    Io(std::io::Error),
    Parse(std::num::ParseIntError),
    NotFound(String),
}
```

### `From` トレイトの手書きが面倒

`?` は `From::from(e)` で自動変換しますが、**`From` 実装がなければ自動変換できません**。`.map_err(...)` で手動変換する方法もありますが、エラーの種類が増えるたびに書く必要があります：

```rust
// .map_err() で手動変換する方法
fn load_config() -> Result<i32, AppError> {
    let content = fs::read_to_string("config.txt")
        .map_err(|e| AppError::Io(e))?;    // io::Error → AppError::Io に変換してから ?
    let value: i32 = content.trim().parse()
        .map_err(|e| AppError::Parse(e))?; // ParseIntError → AppError::Parse に変換してから ?
    Ok(value)
}
```

`From` 実装がなくてもこれで動きますが、呼び出し箇所が増えるたびに同じ変換コードが散らばります。`From` を実装する方法の場合も同様で、エラーの種類ごとに手書きが必要です：

```rust
// エラーの種類ごとに From を手書きする必要がある（面倒）
impl From<std::io::Error> for AppError {
    fn from(e: std::io::Error) -> Self { AppError::Io(e) }
}
impl From<std::num::ParseIntError> for AppError {
    fn from(e: std::num::ParseIntError) -> Self { AppError::Parse(e) }
}
// エラーの種類が増えるたびに1つずつ追加しなければならない…
```

そこで **`thiserror`** クレートを使います：

```toml
[dependencies]
thiserror = "2"
```

```rust
use thiserror::Error;

#[derive(Debug, Error)]
enum AppError {
    #[error("I/O エラー: {0}")]
    Io(#[from] std::io::Error),

    #[error("パースエラー: {0}")]
    Parse(#[from] std::num::ParseIntError),

    #[error("見つかりません: {0}")]
    NotFound(String),
}
```

### `#[error]` と `#[from]` が生成するもの

**`#[from]`** は `From` トレイトの実装を自動生成します（上の手書きコードが不要になる）。

**`#[error("...")]`** はそのエラーを `{}` で表示したときのメッセージを定義します。内部では `Display` トレイトの実装に展開されます：

```rust
// #[error("I/O エラー: {0}")] が生成するコードのイメージ
impl std::fmt::Display for AppError {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        match self {
            AppError::Io(e)       => write!(f, "I/O エラー: {}", e),
            AppError::Parse(e)    => write!(f, "パースエラー: {}", e),
            AppError::NotFound(s) => write!(f, "見つかりません: {}", s),
        }
    }
}
```

`{0}` はタプル構造体の1番目の中身を、名前付きフィールドはフィールド名で指定します：

```rust
#[error("範囲外: 最大={max}, 実際={actual}")]
OutOfRange { max: i32, actual: i32 },

// 実際の表示
let err = AppError::NotFound("config.txt".to_string());
println!("{}", err); // → 見つかりません: config.txt
```

これで `?` がそのまま使えます：

```rust
fn load_config() -> Result<i32, AppError> {
    let content = std::fs::read_to_string("config.txt")?; // io::Error → AppError::Io
    let value: i32 = content.trim().parse()?;             // ParseIntError → AppError::Parse
    Ok(value)
}
```

---

## anyhow — プロトタイプや小規模アプリ向け

エラー型を細かく定義するのが面倒な場合は **`anyhow`** が便利です：

```toml
[dependencies]
anyhow = "1"
```

```rust
use anyhow::Result;

fn main() -> Result<()> {
    let content = std::fs::read_to_string("config.txt")?;
    let value: i32 = content.trim().parse()?;
    println!("設定値: {}", value);
    Ok(())
}
```

`anyhow::Result` は全てのエラーを受け取れる汎用型です。エラーの種類を区別しなくていいので、スクリプト的な用途に向いています。

---

## thiserror vs anyhow の使い分け

| | `thiserror` | `anyhow` |
|---|---|---|
| 用途 | ライブラリ・エラーを種類ごとに扱いたい | アプリ・エラーを一括で扱いたい |
| 型 | 独自の `enum` エラー型 | `anyhow::Error`（何でも受け取る） |
| 呼び出し元 | エラーの種類に応じた処理ができる | エラーは表示するだけが多い |

---

## まとめ

- `Option<T>` — 値がないかもしれない（`null` の代わり）
- `Result<T, E>` — 成功か失敗か（例外の代わり）
- `?` — エラーを呼び出し元に伝播する糖衣構文
- `thiserror` — 独自エラー型を簡単に定義する（ライブラリ向け）
- `anyhow` — 型を気にせず手軽にエラーハンドリング（アプリ向け）

Rustのエラーハンドリングは「エラーが起きうる箇所が型に現れる」ため、ドキュメントを読まなくても関数の動作が分かりやすくなります。
