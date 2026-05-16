---
title: "Rustでチャットサーバーを作る #10 — Rust のモジュールシステムとファイル分割"
emoji: "📦"
type: "tech"
topics: ["rust"]
published: false
---

# Rustでチャットサーバーを作る #10 — Rust のモジュールシステムとファイル分割

コードが1ファイルに集中してきたので、責務ごとに分割することにしました。「Rust のモジュールシステムって複雑そう...」と思っていましたが、やってみると思ったよりシンプルでした。

---

## クレートとモジュールの違い

最初に「`.rs` ファイルがクレートになるの？」と思っていましたが、少し違います。

- **クレート** = コンパイルの単位。`Cargo.toml` で管理するプロジェクト全体（または外部ライブラリ1つ）
- **モジュール** = クレート内の名前空間の区切り。`.rs` ファイルや `mod {}` ブロックが対応

```
chat-server クレート（全体）
├── src/main.rs      → ルートモジュール
├── src/message.rs   → message モジュール
└── src/server.rs    → server モジュール
```

`tokio` や `std` は**外部クレート**です。`crate::` と書いたときの `crate` は「今コンパイルしている自分自身のクレート全体のルート」を指します。

---

## mod 宣言 — Rust にファイルの存在を教える

分割した `.rs` ファイルは、自動的には使えません。ルートモジュール（`main.rs`）で **`mod`** 宣言が必要です：

```rust
// main.rs
mod message;  // src/message.rs を読み込む
mod server;   // src/server.rs を読み込む
```

`mod message;` と書くと、Rust は `src/message.rs` を探して読み込みます。セミコロンを忘れると構文エラーになります。

---

## pub と use — 公開と利用

モジュール内の定義は**デフォルトで非公開**です。他のモジュールから使うには `pub` を付けます：

```rust
// message.rs
pub enum Message { ... }          // pub を付けて公開
pub fn parse_message(...) { ... } // これも pub
```

呼び出し側では `use` でパスを指定します：

```rust
// server.rs
use crate::message::Message;         // crate:: = 自分のクレートのルートから
use crate::message::parse_message;
```

`crate::message::` の部分が「このクレートの message モジュールの中の」という意味です。

---

## 最終的なファイル構成

| ファイル | 役割 |
|---|---|
| `main.rs` | サーバー起動・accept ループ・シャットダウン処理 |
| `message.rs` | `Message` 型の定義と `parse_message` 関数 |
| `server.rs` | `handle_client` 関数（1クライアントの処理） |

`main.rs` は `use crate::server::handle_client;` で関数を取り込み、`tokio::spawn` 内で呼び出すだけになりました：

```rust
mod message;
mod server;

use crate::server::handle_client;

// ...

tokio::spawn(async move {
    handle_client(stream, addr, tx).await;
});
```

---

## まとめ

- Rust の**クレート** = `Cargo.toml` 単位のコンパイル単位
- **モジュール** = クレート内の名前空間（`.rs` ファイル = 1モジュール）
- `mod ファイル名;` でルートモジュールにファイルを追加
- `pub` を付けないと他のモジュールから見えない
- `use crate::モジュール名::定義名;` で取り込む

次の記事では、`println!` に代わる本格的なロギングの仕組み、`tracing` クレートを導入します。
