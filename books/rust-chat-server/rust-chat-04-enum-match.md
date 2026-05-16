---
title: "Rustでチャットサーバーを作る #4 — enumとmatchでコマンド処理"
emoji: "🔀"
type: "tech"
topics: ["rust"]
published: false
---

# Rustでチャットサーバーを作る #4 — enumとmatchでコマンド処理

## チャットコマンドの設計

チャットサーバーでは、受け取った文字列を種類ごとに処理したいです。

```
"/nick Alice" → ユーザー名設定
"/quit"       → 切断
"Hello"       → チャットメッセージ
```

Rust の `enum` を使ったとき、「え、これだけで表現できるの？」と驚いたので記録します。

---

## Rust の enum は「値を持てる」

他の言語の enum との最大の違いがこれです。

```typescript
// TypeScript の enum（名前付き定数にすぎない）
enum MessageType { Chat, Nick, Quit }
// 値は別途管理が必要
```

```rust
// Rust の enum（各バリアントがデータを持てる）
enum Message {
    Chat(String),  // チャット文字列を持つ
    Nick(String),  // 新しいユーザー名を持つ
    Quit,          // データなし
}
```

「メッセージの種類」と「その内容」を**1つの型で表現できる**ことに気づいたとき、TypeScriptやJavaの `enum` とは全然別物だと感じました。

---

## 文字列 → enum に変換する関数

```rust
fn parse_message(line: &str) -> Message {
    if line.starts_with("/nick ") {
        Message::Nick(line.strip_prefix("/nick ").unwrap_or("").to_string())
    } else if line == "/quit" {
        Message::Quit
    } else {
        Message::Chat(line.to_string())
    }
}
```

### ポイント1: 引数は &str

「読むだけ」なので所有権を受け取る必要はありません。`&str` にすることで文字列リテラルも `String` の参照も両方受け取れます。

### ポイント2: セミコロンなしが戻り値

```rust
if ... {
    Message::Nick(...)   // ← ; なし → これが戻り値
} else if ... {
    Message::Quit        // ← ; なし → これが戻り値
} else {
    Message::Chat(...)   // ← ; なし → これが戻り値
}
```

Rust では**ブロックの最後の式（`;` なし）がそのブロックの値**になります。

### ポイント3: strip_prefix

```rust
"/nick Alice".strip_prefix("/nick ")  // → Some("Alice")
"hello".strip_prefix("/nick ")        // → None
```

`Option<&str>` が返るので `.unwrap_or("")` でデフォルト値を指定しています。

---

## match で種類ごとに処理

```rust
for line in reader.lines() {
    let line = line.unwrap();
    let msg = parse_message(&line);

    match msg {
        Message::Chat(text) => {
            println!("チャット: {}", text);
            writeln!(writer, "{}", text).unwrap();
        }
        Message::Nick(name) => {
            println!("名前変更: {}", name);
            writeln!(writer, "名前を {} に設定しました", name).unwrap();
        }
        Message::Quit => {
            println!("切断");
            break;
        }
    }
}
```

### match の強力な点

**全ケースを書かないとコンパイルエラーになります。**

```rust
match msg {
    Message::Chat(text) => { ... }
    Message::Nick(name) => { ... }
    // Quit を書き忘れると →
    // error[E0004]: non-exhaustive patterns: `Quit` not covered
}
```

実際に一度書き忘れてコンパイルエラーで指摘されたとき、「TypeScript の `switch` や Java の `if-else` なら黙って通るのに」と思いました。Rust の `match` は**処理漏れをコンパイル時に防ぐ**強力な仕組みだと体感しました。

---

## Phase 1 完成コード

```rust
use std::io::{BufRead, BufReader, Write};
use std::net::TcpListener;

enum Message {
    Chat(String),
    Nick(String),
    Quit,
}

fn parse_message(line: &str) -> Message {
    if line.starts_with("/nick ") {
        Message::Nick(line.strip_prefix("/nick ").unwrap_or("").to_string())
    } else if line == "/quit" {
        Message::Quit
    } else {
        Message::Chat(line.to_string())
    }
}

fn main() {
    let listener = TcpListener::bind("127.0.0.1:8080").unwrap();
    println!("サーバー起動: 127.0.0.1:8080");

    let (stream, addr) = listener.accept().unwrap();
    println!("接続あり: {}", addr);

    let mut writer = stream.try_clone().unwrap();
    let reader = BufReader::new(stream);

    for line in reader.lines() {
        let line = line.unwrap();
        let msg = parse_message(&line);

        match msg {
            Message::Chat(text) => {
                println!("チャット: {}", text);
                writeln!(writer, "{}", text).unwrap();
            }
            Message::Nick(name) => {
                println!("名前変更: {}", name);
                writeln!(writer, "名前を {} に設定しました", name).unwrap();
            }
            Message::Quit => {
                println!("切断");
                break;
            }
        }
    }
}
```

### 動作確認

```bash
cargo run

# 別ターミナル
nc 127.0.0.1 8080
/nick Taro        # → 「名前を Taro に設定しました」と返ってくる
こんにちは        # → チャットメッセージがエコーされる
/quit             # → サーバーとの接続が切れる
```

---

## まとめ

- Rust の `enum` は各バリアントがデータを持てる（他言語より強力）
- セミコロンなしの最後の式がブロックの戻り値になる
- `match` は全ケース網羅を強制 → 処理漏れをコンパイル時に防ぐ
- `&str` は「読むだけ」の引数に使う基本パターン

**現時点の限界**: 1対1専用サーバーで、1人が接続中は2人目が待たされます。  
次の Phase 2 では `thread::spawn` と `Arc<Mutex<>>` を使ってマルチクライアント対応にします。
