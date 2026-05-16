---
title: "Rustでチャットサーバーを作る #6 — Arc・Mutex・mpsc でブロードキャスト"
emoji: "📡"
type: "tech"
topics: ["rust"]
published: false
---

# Rustでチャットサーバーを作る #6 — Arc・Mutex・mpsc でブロードキャスト

## スレッドを分けたら、次の問題が出た

前回でマルチスレッド化できました。でも気づいてしまいました。「AさんのメッセージをどうやってBさんとCさんに届けるの？」と。

各スレッドは独立していて、お互いのことを知りません。「全員のリスト」をどこかで共有する必要があります。

---

## Arc — 「共有の棚」

複数スレッドで同じデータを持つために `Arc`（Atomic Reference Counted）を使います。

```
スレッド1 ─┐
スレッド2 ─┼─→ Arc<データ>  （全員が同じ棚を指している）
スレッド3 ─┘
```

`Arc::clone()` はデータをコピーするのではなく、**参照カウントを増やす**だけです。全員が同じ1つのデータを参照しています。

```rust
let data = Arc::new(vec![1, 2, 3]);

let data2 = Arc::clone(&data); // カウント: 2
let data3 = Arc::clone(&data); // カウント: 3
// 全員が同じ vec を指している
```

---

## Mutex — 「トイレの鍵」

`Arc` で共有できても、複数スレッドが同時に書き込んだらデータが壊れます。`Mutex` はその衝突を防ぐ「鍵付きの箱」です。

```
スレッド1が lock() → 🔒 書き込み中
スレッド2は待機    → ⏳
スレッド1が解放    → 🔓
スレッド2が lock() → 🔒 書き込み中
```

使い方は `lock()` で `MutexGuard` を取り出すだけ：

```rust
let guard = mutex.lock().unwrap(); // 🔒 鍵を取る
guard.insert(...);                 // データを操作
// ← guard がスコープを抜けると自動で 🔓 解放（RAII）
```

`unlock()` を明示的に呼ぶ必要がない点が Rust らしいと感じました。

### lock() が Err になるケース

「他のスレッドがロック中のとき失敗するのでは？」と思いましたが、それは**待機**です。  
`Err` になるのは「鍵を持ったまま panic したスレッドがある」（Poison）場合だけです。

---

## Arc\<Mutex\<HashMap\>\> の完成形

2つを組み合わせると、スレッド間で安全に共有できる HashMap ができます：

```rust
let clients: Arc<Mutex<HashMap<SocketAddr, mpsc::Sender<String>>>>
    = Arc::new(Mutex::new(HashMap::new()));
//  ^^^^       ^^^^^       ^^^^^^^^^^^^^^^
//  共有       排他制御    実際のデータ（クライアント一覧）
```

接続が来るたびに `Arc::clone()` してスレッドに渡します：

```rust
loop {
    let (stream, addr) = listener.accept().unwrap();
    let (tx, rx) = mpsc::channel::<String>(); // ← 各クライアントのチャンネル

    clients.lock().unwrap().insert(addr, tx); // ← txをHashMapに登録
    let clients = clients.clone();            // ← cloneしてスレッドに渡す

    thread::spawn(move || {
        // clients, rx, stream を使って処理
    });
}
```

---

## mpsc — ブロードキャストの仕組み

`mpsc`（multiple producer, single consumer）でクライアントごとにチャンネルを1本用意します。

```
HashMap の中身:
  Aのアドレス → tx_a  →  rx_a → Aのストリームに書く
  Bのアドレス → tx_b  →  rx_b → Bのストリームに書く
  Cのアドレス → tx_c  →  rx_c → Cのストリームに書く
```

Aさんがメッセージを送ったとき、HashMap の**全員の `tx`** に `send()` します。

```rust
Message::Chat(text) => {
    let clients = clients.lock().unwrap();
    for (_, (tx, _)) in clients.iter() {
        tx.send(text.clone()).unwrap(); // 全員に届ける
    }
}
```

### rx を別スレッドで処理する理由

`rx.recv()` はメッセージを待ってブロックします。socket の読み込みもブロックします。両方を1スレッドでやるとどちらかが止まってしまうので、**rx 専用のスレッドを内側に作りました**。

```rust
thread::spawn(move || {
    let mut writer_for_rx = stream.try_clone().unwrap(); // rx 用
    let mut writer = stream.try_clone().unwrap();        // コマンド応答用
    let reader = BufReader::new(stream);

    // rx を受け取って書き出す専用スレッド
    thread::spawn(move || {
        for msg in rx {
            writeln!(writer_for_rx, "{}", msg).unwrap();
        }
    });

    // socket から読んで全員に send する
    for line in reader.lines() { ... }
});
```

---

## デッドロック対策：スコープで MutexGuard を解放

ここで詰まりました。入室通知で `lock()` した MutexGuard が解放されないまま次の `lock()` を呼んでいたのです。

```rust
// NG: MutexGuard が生きたまま次の lock() を呼ぶ → デッドロック！
let clients = clients.lock().unwrap();
for (_, (tx, _)) in clients.iter() { ... }
// ← ここで解放されると思っていたが、スコープがまだ続いている

clients.lock().unwrap().insert(...); // デッドロック！
```

`{ }` で明示的にスコープを区切ることで解決しました：

```rust
// OK: { } を抜けた瞬間に MutexGuard が drop されてロック解放
{
    let clients = clients.lock().unwrap();
    for (_, (tx, _)) in clients.iter() { ... }
} // ← ここで解放

clients.lock().unwrap().insert(...); // 問題なし
```

「スコープを抜けたら自動解放」という RAII が、デッドロック防止にも関わってくるとはこの時初めて気づきました。

---

## コマンド実装：enum に List を追加

`/nick`・`/quit` は Phase 1 からありましたが、`/list` を追加しました。`enum Message` に新しいバリアントを追加するだけです。

```rust
enum Message {
    Chat(String),
    Nick(String),
    List,   // ← 追加（引数なし）
    Quit,
}
```

`parse_message` にも1行追加：

```rust
} else if line == "/list" {
    Message::List
}
```

### /list の実装：values() と map() と collect()

HashMap から全員のニックネームを取り出して表示します。

```rust
Message::List => {
    let clients = clients.lock().unwrap();
    let names: Vec<String> = clients.values()
        .map(|(_, nick)| {
            nick.clone().unwrap_or_else(|| "名無し".to_string())
        })
        .collect();
    writeln!(writer, "接続中: {}", names.join(", ")).unwrap();
}
```

- `.values()` → HashMap のバリューだけを取り出すイテレータ
- `.map(|(_, nick)| ...)` → タプルからニックネームだけを取り出して変換
- `.collect()` → イテレータを `Vec<String>` にまとめる

「タプルの `tx` はなぜ Vec に入らないの？」と思いましたが、`.map()` で `nick` だけを返しているからです。`_` にバインドした値はその場で捨てられます。

### ニックネームの保存：HashMap の型変更と Option

ニックネームは接続直後はありません。「まだ設定していない」状態を表すために `Option<String>` を使いました。

```rust
// 変更前
HashMap<SocketAddr, mpsc::Sender<String>>

// 変更後（ニックネームをタプルで持つ）
HashMap<SocketAddr, (mpsc::Sender<String>, Option<String>)>
//                                         ^^^^^^^^^^^^^^^ Some("太郎") か None
```

`/nick` コマンドでニックネームを更新するには `get_mut()` で可変参照を取ります：

```rust
Message::Nick(name) => {
    writeln!(writer, "名前を {} に設定しました", name).unwrap();
    if let Some(entry) = clients.lock().unwrap().get_mut(&addr) {
        entry.1 = Some(name); // タプルの2番目を更新
    }
}
```

---

## 入退室通知

接続時と切断時に全員へブロードキャストします。Chat と全く同じ「全員の `tx` に `send()`」パターンです。

```rust
// 入室通知（for ループの前）
{
    let msg = format!("{} が入室しました", addr);
    let clients = clients.lock().unwrap();
    for (_, (tx, _)) in clients.iter() {
        tx.send(msg.clone()).unwrap();
    }
} // ← ここで MutexGuard が drop

// 退室通知（for ループの後）
{
    let msg = format!("{} が退出しました", addr);
    let mut clients = clients.lock().unwrap();
    for (_, (tx, _)) in clients.iter() {
        tx.send(msg.clone()).unwrap();
    }
    clients.remove(&addr); // HashMap からも削除
}
```

`{ }` で囲んでいるのはデッドロック対策です（前の章で説明済み）。

---

## LAN 公開：0.0.0.0 に変更

最後は1行の変更でした。

```rust
// 変更前（自分自身からしか接続できない）
TcpListener::bind("127.0.0.1:8080")

// 変更後（LAN 内の全デバイスから接続可能）
TcpListener::bind("0.0.0.0:8080")
```

| アドレス | 意味 | アクセス元 |
|---|---|---|
| `127.0.0.1` | ループバック | 同じ PC 内だけ |
| `0.0.0.0` | 全インターフェース | LAN 内の全デバイス |

`0.0.0.0` は「どの IP アドレスからの接続も受け付ける」という意味です。

---

## 完成した機能

- 複数クライアントのブロードキャストチャット
- `/nick <名前>` でニックネーム設定
- `/list` で接続中ユーザー一覧表示
- `/quit` で切断
- 入退室通知
- `0.0.0.0:8080` で LAN 公開

---

## まとめ

- `Arc` は「参照カウントで管理する共有所有権」。データはコピーされない
- `Mutex` は「1スレッドしか入れないトイレの鍵」。`MutexGuard` は RAII で自動解放
- `mpsc::channel` でクライアントごとに専用チャンネルを用意してブロードキャスト
- `{ }` でスコープを明示することでデッドロックを防げる
- `tx` が `drop` されると `rx` のループが自動で終わる（チャンネルのクローズ）

次の記事では、同期版から非同期版（Tokio）への書き換えに備えて `async/await` と `Future` の概念を整理します。
