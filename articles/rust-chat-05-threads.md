---
title: "Rustでチャットサーバーを作る #5 — thread::spawn と move クロージャ"
emoji: "🧵"
type: "tech"
topics: ["rust"]
published: false
---

# Rustでチャットサーバーを作る #5 — thread::spawn と move クロージャ

## 「2人目が繋がらない」問題

Phase 1 で完成させた 1対1 チャットサーバー。実は大きな問題を抱えていました。

```rust
let (stream, addr) = listener.accept().unwrap(); // 1回だけ
for line in reader.lines() { ... }               // ここでずっと止まっている
```

2人目が接続しようとしても、`accept()` を呼ぶ前に `for` ループで詰まったまま。「ドアの受付係がお客さんの対応中で、次のお客さんを迎えに行けない」状態です。

解決策として `thread::spawn` を使ってマルチスレッド化することにしました。

---

## スレッドが必要な理由

「接続ごとにスレッドを生やして、そこに処理を任せる」という設計です。

```
メインスレッド:  accept → accept → accept → ...（ずっと待ち受け）
                    ↓         ↓
スレッド1:      1人目の処理...
スレッド2:              2人目の処理...
```

メインスレッドはすぐに次の `accept()` に戻れる。他の言語でも同じ発想ですが、Rustには独特の書き方があります。

---

## thread::spawn の基本形

```rust
use std::thread;

thread::spawn(move || {
    // ここに新しいスレッドでやりたい処理
});
```

### クロージャとは

`|| { }` と書く「名前のない関数」です。普通の関数と違って、**周りの変数を取り込める（キャプチャする）** のが特徴です。

```rust
let name = String::from("太郎");

// クロージャは外の変数をそのまま使える
let greet = || println!("こんにちは、{}さん", name);
greet(); // → "こんにちは、太郎さん"
```

### move が必要な理由

ここで最初につまずきました。「代入したら所有権が移るはずなのに、なぜ `move` が必要なの？」と。

クロージャはデフォルトで変数を**借用**しようとします。でも、スレッドはいつ終わるかわからないので「借用元が先に消えるかも」とコンパイラが判断して弾きます。

```rust
// moveなし → コンパイルエラー
thread::spawn(|| {
    let reader = BufReader::new(stream); // streamを借用しようとする
    // でもスレッドの寿命が不明 → 危険！
});

// move あり → OK
thread::spawn(move || {
    let reader = BufReader::new(stream); // streamの所有権ごと受け取る
});
```

`move` は「指示書に変数を貼り付けて担当者（スレッド）に渡す」イメージです。

---

## 実際のコード変更

`loop` の中で接続を受け、処理をスレッドに渡す形に書き換えました。

```rust
loop {
    // ① メインスレッドで接続を受け付ける
    let (stream, addr) = listener.accept().unwrap();
    println!("接続あり: {}", addr);

    // ② 処理をスレッドに渡す（streamの所有権ごと）
    thread::spawn(move || {
        let mut writer = stream.try_clone().unwrap();
        let reader = BufReader::new(stream); // streamをここで使う

        for line in reader.lines() {
            // ... Phase 1 と同じ処理
        }
    });
    // ③ スレッドに任せてすぐここに戻る → accept()を待てる
}
```

### ポイント: try_clone と clone の違い

`stream.try_clone()` と `Arc::clone()` の違いも今回理解しました。

| | `clone()` | `try_clone()` |
|---|---|---|
| 戻り値 | `T` | `Result<T, Error>` |
| 失敗の可能性 | なし | あり（OS制限） |

`TcpStream` は OS が管理するリソースなので、複製が失敗することがある。だから `Result` を返す `try_clone()` を使います。

---

## まとめ

- `loop` + `accept()` で複数接続を受け付けられる
- `thread::spawn` でクライアントごとの処理をスレッドに分離
- クロージャはデフォルトで借用 → スレッドに渡すときは `move` で所有権ごと渡す
- `try_clone()` は OS リソースの複製なので失敗しうる → `Result` を返す

次の記事では、スレッド間でクライアント一覧を共有するための `Arc<Mutex<HashMap>>` と、ブロードキャストのための `mpsc::channel` について書きます。
