---
title: "Rustのモジュールシステムとクレート管理を手を動かして理解する"
emoji: "📦"
type: "tech"
topics: ["rust"]
published: true
---

# Rustのモジュールシステムとクレート管理を手を動かして理解する

Rustでコードが大きくなってきたとき、ファイルを分けて整理したくなります。「モジュール」「クレート」「パス」という概念を正しく理解すれば、Rustの構造化は思ったよりシンプルです。

---

## クレートとパッケージ

まず用語を整理します：

| 用語 | 意味 |
|---|---|
| **クレート** | コンパイルの単位。バイナリクレート（実行可能ファイル）またはライブラリクレート |
| **パッケージ** | `Cargo.toml` を持つプロジェクト。1つ以上のクレートを含む |

```
my-project/          ← パッケージ
  Cargo.toml
  src/
    main.rs          ← バイナリクレートのルート
    lib.rs           ← ライブラリクレートのルート（任意）
```

外部クレート（`tokio`, `serde` など）は `Cargo.toml` の `[dependencies]` に書きます。

---

## モジュール — コード内の名前空間

モジュールはクレート内のコードを整理する仕組みです。

### インラインモジュール

```rust
mod math {
    pub fn add(a: i32, b: i32) -> i32 {
        a + b
    }

    // プライベート（外から見えない）
    fn internal() {}
}

fn main() {
    println!("{}", math::add(1, 2)); // 3
}
```

### ファイルモジュール

モジュールをファイルに分けるには `mod` 宣言をルートに書くだけです：

```
src/
  main.rs      ← mod math; と書く
  math.rs      ← math モジュールの内容
```

```rust
// main.rs
mod math;  // src/math.rs を読み込む

fn main() {
    println!("{}", math::add(1, 2));
}
```

```rust
// math.rs
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

### サブディレクトリのモジュール

さらに深い階層は `mod.rs` または同名ファイルで表現します：

```
src/
  main.rs
  utils/
    mod.rs     ← utils モジュール（または utils.rs でも可）
    math.rs    ← utils::math サブモジュール
```

---

## pub — 公開範囲の制御

デフォルトは非公開です。外から使えるようにするには `pub` を付けます：

```rust
pub struct User {           // pub: 外から見える
    pub name: String,       // pub: フィールドも外から見える
    password: String,       // 非公開
}

pub fn create_user(name: &str) -> User {
    User {
        name: name.to_string(),
        password: "secret".to_string(), // 同じモジュール内なのでアクセス可
    }
}
```

`pub(crate)` で「このクレート内だけ公開」、`pub(super)` で「親モジュールだけ公開」なども使えます。

---

## use — パスを省略する

長いパスを毎回書かなくて済むように `use` でインポートします：

```rust
// 毎回フルパスで書く（冗長）
let v = std::collections::HashMap::new();

// use でインポート
use std::collections::HashMap;
let v = HashMap::new();

// 複数まとめて
use std::collections::{HashMap, HashSet, BTreeMap};

// 全部
use std::collections::*;
```

同じクレート内を参照するときは `crate::` を使います：

```rust
// server.rs から message.rs の定義を使う
use crate::message::parse_message;
```

---

## Cargo.toml — 依存関係の管理

外部クレートは `Cargo.toml` に追加します：

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
tracing = "0.1"
```

`cargo add` コマンドで自動追加できます：

```bash
cargo add tokio --features full
cargo add serde --features derive
```

### バージョン指定の書き方

| 書き方 | 意味 |
|---|---|
| `"1"` | 1.x.x の最新（後方互換あり） |
| `"1.2"` | 1.2.x の最新 |
| `"=1.2.3"` | 1.2.3 ピン留め |
| `">=1.0, <2.0"` | 範囲指定 |

Rustのクレートは**セマンティックバージョニング**に従い、メジャーバージョンが変わると破壊的変更が入ります。

---

## よく使うCargo コマンド

```bash
cargo new my-project      # 新規プロジェクト作成
cargo build               # デバッグビルド
cargo build --release     # リリースビルド（最適化あり）
cargo run                 # ビルド＆実行
cargo test                # テスト実行
cargo check               # コンパイルチェック（ビルドより速い）
cargo add tokio           # 依存クレートを追加
cargo update              # Cargo.lock を更新
cargo doc --open          # ドキュメント生成＆ブラウザで開く
```

---

## まとめ

- **クレート** = コンパイル単位（`Cargo.toml` のプロジェクト全体）
- **モジュール** = コード内の名前空間（`.rs` ファイル = 1モジュール）
- `mod name;` でファイルをモジュールとして読み込む
- デフォルト非公開。`pub` を付けると外部からアクセス可
- `use crate::` で同じクレート内のパスを短縮
- `Cargo.toml` で外部クレートの依存を管理

モジュールシステムを正しく使うことで、大きなプロジェクトでも各ファイルの責務が明確になります。
