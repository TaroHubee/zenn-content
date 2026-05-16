---
title: "Rustでチャットサーバーを作る #9 — tokio::select! で複数の Future を同時に待つ"
emoji: "🎯"
type: "tech"
topics: ["rust", "tokio"]
published: true
---

# Rustでチャットサーバーを作る #9 — tokio::select! で複数の Future を同時に待つ

Tokio 版に書き換えたところで、ある問題に気づきました。「チャットメッセージが片方にしか届かない」という現象です。原因を調べる中で `tokio::select!` というとても便利なマクロに出会いました。

---

## なぜメッセージが届かなかったのか

当初のコードはこんな流れでした：

```rust
loop {
    reader.read_line(&mut line).await; // ← ここで止まる
    // ブロードキャスト受信
    rx.recv().await;
}
```

`read_line` は次の行が来るまで**ずっと待ち続けます**。その間、`rx.recv()` は呼ばれません。クライアントが何も書かない間は、ブロードキャストが届いても受け取れないのです。

「二つの Future を同時に待ちたい」——これが `tokio::select!` の出番です。

---

## tokio::select! — 複数の Future を競わせる

```rust
tokio::select! {
    // 分岐①: クライアントから1行来た
    result = reader.read_line(&mut line) => {
        // line の処理
    }

    // 分岐②: broadcast からメッセージが届いた
    result = rx.recv() => {
        match result {
            Ok(msg) => {
                let msg = format!("{}\n", msg);
                writer.write_all(msg.as_bytes()).await.unwrap();
            }
            Err(_) => break, // チャンネルが閉じた
        }
    }
}
```

`tokio::select!` は複数の Future を**同時に**進め、**最初に完了したもの**の腕を実行します。もう片方はキャンセルされます。

これで「クライアントから入力がなくても、ブロードキャストが届いた瞬間に受け取れる」ようになりました。

---

## グレースフルシャットダウン — Ctrl+C で全員に通知

サーバーを突然落とすのではなく、接続中のクライアントに「サーバーが落ちます」と伝えてから終了したい。これが**グレースフルシャットダウン**です。

`tokio::signal::ctrl_c()` は Ctrl+C を非同期で待つ Future です。accept ループでも `select!` と組み合わせることができます：

```rust
loop {
    tokio::select! {
        result = listener.accept() => {
            // 新しい接続の処理
        }

        _ = tokio::signal::ctrl_c() => {
            tx.send("サーバーがシャットダウンします".to_string()).ok();
            tokio::time::sleep(std::time::Duration::from_millis(200)).await;
            break;
        }
    }
}
```

ここで `sleep(200ms)` が必要な理由に少し詰まりました。`send` した直後に `break` すると、各タスクがメッセージを受け取る前にプロセスが終了してしまうためです。少し待つことで全タスクがメッセージを処理する時間を確保しています。

---

## まとめ

- `tokio::select!` は複数の Future を同時に待ち、最初に完了した腕を実行する
- `read_line` だけをループすると、待っている間ブロードキャストを受け取れない
- `select!` で `read_line` と `rx.recv()` を同時に待つことで解決
- `tokio::signal::ctrl_c()` で Ctrl+C を非同期で待てる
- シャットダウン前に `sleep` でメッセージ配信の時間を確保する

次の記事では、大きくなったコードをファイルに分割し、Rust のモジュールシステムを学びます。
