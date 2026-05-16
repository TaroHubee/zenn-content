---
title: "Rustでチャットサーバーを作る #8 — 同期版を Tokio 非同期版へ書き換える"
emoji: "🔄"
type: "tech"
topics: ["rust", "tokio", "tcp"]
published: false
---

# Rustでチャットサーバーを作る #8 — 同期版を Tokio 非同期版へ書き換える

Phase 3 で `Future` と `async/await` の概念を理解したところで、いよいよ実際のコードを Tokio 版に書き換えていきます。「概念は分かった、でも実際の API はどう使うの？」という疑問が一気に解消されました。

---

## tokio::net::TcpListener — .await を忘れないで

同期版では `TcpListener::bind("0.0.0.0:8080").unwrap()` と書いていましたが、Tokio 版はこうなります：

```rust
use tokio::net::TcpListener;

let listener = TcpListener::bind("0.0.0.0:8080").await.unwrap();
```

`.await` が必要なのがポイントです。`bind` 自体はほぼ瞬時に終わりますが、Tokio の世界では I/O 操作は全て `async fn` になっているため `.await` が必須です。

接続受付も同様です：

```rust
let (stream, addr) = listener.accept().await.unwrap();
```

同期版の `listener.accept()` に `.await` を足すだけ、と考えると覚えやすいです。

---

## tokio::spawn — スレッドからタスクへ

同期版では `thread::spawn` でクライアントごとに OS スレッドを起動していました。Tokio 版では：

```rust
tokio::spawn(async move {
    // クライアント処理
});
```

インターフェースはよく似ていますが、**OS スレッドではなく軽量な非同期タスクを起動する**という違いがあります。

| | `thread::spawn` | `tokio::spawn` |
|---|---|---|
| 起動するもの | OS スレッド | 非同期タスク |
| コスト | 比較的重い（MB オーダーのスタック） | 軽い（数 KB から） |
| 数の目安 | 数百〜数千 | 数十万も可能 |
| 待ち方 | ブロック | `.await` で他に譲る |

---

## broadcast::channel — 全員に届けるには

同期版では `std::sync::mpsc` を使っていましたが、`mpsc`（multi-producer single-consumer）は受信者が1人だけです。全クライアントへの同報には **`tokio::sync::broadcast`** が適しています。

```rust
use tokio::sync::broadcast;

let (tx, _rx) = broadcast::channel::<String>(32); // バッファサイズ 32
let tx = Arc::new(tx); // 複数タスクで共有

// タスク側で受信者を作る
let mut rx = tx.subscribe();
```

`tx`（送信者）は `Arc` で包んで複数タスクで共有します。`rx`（受信者）は `subscribe()` で各タスクが自分用に作ります。

| | `mpsc` | `broadcast` |
|---|---|---|
| 受信者 | 1人 | 複数可 |
| 主な用途 | タスク間の1対1通信 | 全員への通知 |

なお、`broadcast::Sender` は `send` が `&self`（不変参照）で呼べるため、**`Mutex` 不要**です。`Arc` で包むだけで複数タスクから送信できます。

---

## into_split() — TcpStream を読み書きに分けるには

クライアント処理では「受信しながらブロードキャストを書き込む」必要があります。1つの `TcpStream` を読み用・書き用に分けるために `into_split()` を使います：

```rust
let (reader, mut writer) = stream.into_split();
let mut reader = BufReader::new(reader);
```

最初は同期版のように `try_clone()` を試みましたが、Tokio の `TcpStream` は **Reactor（I/O 通知システム）に登録されている**ため、ファイルディスクリプタをコピーしても正しく動きません。`into_split()` は同じ Reactor 登録を共有しながら読み権・書き権に分割するための正しい方法です。

---

## AsyncBufReadExt / AsyncWriteExt — 非同期版の読み書き

Tokio の非同期ストリームを読み書きするには、対応するトレイトを `use` で取り込む必要があります：

```rust
use tokio::io::{AsyncBufReadExt, AsyncWriteExt, BufReader};

// 読む
let mut line = String::new();
reader.read_line(&mut line).await.unwrap();

// 書く
writer.write_all("こんにちは\n".as_bytes()).await.unwrap();
```

注意点として、`b"こんにちは\n"` のようなバイトリテラルには**日本語が使えません**。文字列に `.as_bytes()` を呼ぶのが正しい方法です。

---

## まとめ

- `tokio::net::TcpListener::bind().await` でリスナー作成
- `listener.accept().await` で接続受付（同期版に `.await` を足すだけ）
- `tokio::spawn` で軽量な非同期タスクを起動
- `broadcast::channel` で複数受信者へのブロードキャスト（`Mutex` 不要）
- `into_split()` で TcpStream を読み権・書き権に分割（`try_clone()` は使えない）
- `AsyncBufReadExt` / `AsyncWriteExt` で非同期読み書き

次の記事では、受信と切断検知を**同時に**待つ `tokio::select!` マクロと、Ctrl+C でのグレースフルシャットダウンを実装します。
