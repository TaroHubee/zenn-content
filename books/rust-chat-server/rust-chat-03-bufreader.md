---
title: "Rustでチャットサーバーを作る #3 — BufReaderで行単位の読み書き"
emoji: "📖"
type: "tech"
topics: ["rust", "tcp"]
published: true
---

# Rustでチャットサーバーを作る #3 — BufReaderで行単位の読み書き

## TCPはバイトの「流れ」

TCP でデータを受け取るとき、送信側が1つのメッセージとして送っても、受信側には**バラバラのバイト列**として届くことがあります。最初「え？送ったとおりに来ないの？」と驚いたですが、ネットワーク層の仕様上そうなんだと納得しました。

```
送信: "Hello\n"
受信: "He" → "llo" → "\n"  （ネットワークの都合で分割される）
```

チャットアプリでは**改行（`\n`）を1メッセージの区切り**に使うのが自然だと教わりました。それを効率よく扱うのが `BufReader` です。

---

## BufReader とは

内部にバッファを持ち、**改行が来るまでデータをためて、1行まとめて返す**ラッパーです。「ラッパー」という言葉が最初ぴんときませんでしたが、自分では何もしないラッパーをみたいな存在いわしるイメージで理解しました。

```
生の TcpStream: バイトを1個ずつ読む（不便）
BufReader:      改行まで読んで1行として返す（便利）
```

---

## コード

```rust
use std::io::{BufRead, BufReader, Write};
use std::net::TcpListener;

fn main() {
    let listener = TcpListener::bind("127.0.0.1:8080").unwrap();
    println!("サーバー起動: 127.0.0.1:8080");

    let (stream, addr) = listener.accept().unwrap();
    println!("接続あり: {}", addr);

    // stream を複製して「読む用」と「書く用」に分ける
    let mut writer = stream.try_clone().unwrap();
    let reader = BufReader::new(stream);  // stream の所有権が移る

    for line in reader.lines() {
        let line = line.unwrap();
        println!("受信: {}", line);
        writeln!(writer, "Echo: {}", line).unwrap();
    }
}
```

---

## なぜ try_clone() が必要か

```rust
let reader = BufReader::new(stream); // stream の所有権が BufReader に移る
writeln!(stream, ...);               // ← stream はもう使えない！
```

`BufReader::new(stream)` で所有権が移ってしまうため、**先に複製を作っておく**必要があります。

```rust
let mut writer = stream.try_clone().unwrap(); // 先に複製（書く用）
let reader = BufReader::new(stream);          // 元は読む用に渡す
```

これも所有権の制約から来る設計です。

---

## for ループはいつ終わるか

`reader.lines()` はイテレータです。次の行を要求したとき：

- 次の `\n` が来た → 1行返す（ループ1周）
- まだ来ていない → **ここで待つ（ブロック）**
- 接続が切れた → `None` を返す → ループ終了

つまり**クライアントが切断するまでループし続けます**。

---

## &str と String の違い

この辺で混乱したのが `&str` と `String` の使い分けです。最初は「文字列なのになんで型が2種類もあるの？」と戸惑いました。

| | &str | String |
|---|---|---|
| 置き場所 | スタック（参照） | ヒープ |
| 所有権 | なし（借用） | あり |
| 変更 | 不可 | 可能 |
| 用途 | 読むだけ | 変更・所有したい |

```rust
let s: &str = "hello";              // 文字列リテラル・読み取り専用
let s: String = String::from("hello"); // ヒープに確保・変更可能
```

### 関数の引数は &str が基本

```rust
fn do_something(s: &str) { ... }   // 読むだけ → &str
fn do_something(s: String) { ... } // 所有して変更 → String
```

Copilotに「読むだけなら `&str` にすると、`String` も文字列リテラルも両方渡せる汎用性が生まれる」と教わりました。それからは意識して使えるようになりました。

```rust
do_something("hello");              // &str → OK
do_something(&my_string);          // &String → &str として扱える → OK
```

### to_string() は &str → String の変換

```rust
let s: &str = "hello";
let owned: String = s.to_string(); // ヒープにコピーして String を作る
```

---

## まとめ

- `BufReader` は改行区切りで1行ずつ読む便利なラッパー
- `try_clone()` は所有権の制約を回避するために必要
- `lines()` のループはクライアントが切断するまで続く
- 関数の引数は「読むだけなら `&str`」が基本
- `to_string()` で `&str` → `String` に変換できる

次の記事では `enum` と `match` でコマンド処理を実装します。
