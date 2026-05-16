---
title: "Rustでチャットサーバーを作る #2 — TCPサーバーの基礎"
emoji: "🌐"
type: "tech"
topics: ["rust", "tcp"]
published: false
---

# Rustでチャットサーバーを作る #2 — TCPサーバーの基礎

## TcpListener でサーバーを立てる

チャットサーバーには TCP を使います。Copilotに「TCPは『電話』、UDPは『手紙』のイメージ」と教わり、なるほどと思いました。電話は接続を確立してから双方向に通信し、手紙は届いたかどうか確認しません。

メッセージが**確実に・順番通りに届く**必要があるチャットアプリには TCP 一择だと納得しました。

### コード

```rust
use std::net::TcpListener;

fn main() {
    let listener = TcpListener::bind("127.0.0.1:8080").unwrap();
    println!("サーバー起動: 127.0.0.1:8080");

    let (stream, addr) = listener.accept().unwrap();
    println!("接続あり: {}", addr);
}
```

---

## unwrap() とは

`bind()` や `accept()` は `Result<T, E>` を返します。

```rust
enum Result<T, E> {
    Ok(T),   // 成功
    Err(E),  // 失敗
}
```

`.unwrap()` は `Ok` なら中の値を取り出し、`Err` なら**パニック（即クラッシュ）**します。  
Copilotに「学習中は使っていいけど、本番コードでは `match` や `?` 演算子で丁寧に扱うんだよ」と教わりました。今はまず動かすことを優先して `.unwrap()` で進めます。

```rust
// unwrap を使わない書き方
let listener = match TcpListener::bind("127.0.0.1:8080") {
    Ok(l)  => l,
    Err(e) => { eprintln!("起動失敗: {}", e); return; }
};
```

---

## listener と stream の役割分担

```
listener（受付係）
  → ポートを予約してOSに登録する
  → 接続が来るのをひたすら待つ
  → 来たら stream を作って返す（何度でも繰り返せる）

stream（専用回線）
  → 特定の1クライアントとの通信路
  → 切断したら消える
```

`listener.accept()` を呼ぶと、接続が来るまで**処理がそこで止まります**（ブロッキング）。  
最初は「プログラムが止まる」とまたバグかと思いましたが、「接続を待っている状態」なんだとわかって納得しました。接続が来た瞬間に OS が新しい `TcpStream` を作り、`(stream, addr)` として返してきます。

### 所有権の観点

```rust
let (stream, addr) = listener.accept().unwrap();
```

`accept()` は `listener` を**借用（`&self`）**して呼ばれます。所有権は移りません。  
`stream` は `accept()` の呼び出し時に新しく生まれた値です。

---

## ブロッキングの問題

`accept()` は接続が来るまで待ち続けます。また `stream` で読み書きしている間も次の接続を受け付けられません。

```
1人目が接続 → accept() が返る → stream で読み書き中
                                        ↑
                               この間 accept() は呼ばれない
                               → 2人目はOSのキューで待たされる
```

これを解決するのが **Phase 2 のスレッド**です。

---

## 動作確認

```bash
# サーバー起動
cargo run

# 別ターミナルで接続（nc = netcat）
nc 127.0.0.1 8080
```

`nc`（netcat）は TCP 接続を張るシンプルなツールで、専用クライアントがなくても動作確認ができます。

---

## まとめ

- `TcpListener::bind` でポートを予約して待ち受け開始
- `accept()` は接続が来るまでブロック（待ち続ける）
- `listener` は何度でも `accept()` できる受付係
- `stream` は1接続専用の通信路で、`accept()` のたびに新しく生まれる
- 1対1なのでマルチクライアントには別の仕組みが必要（→ Phase 2）

次の記事では `stream` を使って実際にデータを読み書きします。
