---
title: "Rustでチャットサーバーを作る #7 — asyncとFutureの正体を理解する"
emoji: "⚡"
type: "tech"
topics: ["rust", "tokio"]
published: false
---

# Rustでチャットサーバーを作る #7 — asyncとFutureの正体を理解する

Phase 2 でスレッドによるマルチスレッドチャットサーバーを完成させた後、いよいよ非同期処理へ。
「async/await って名前は聞いたことあるけど、内側で何が起きているの？」という疑問を、今回じっくり解消しました。

---

## 「同期」と「非同期」を喫茶店で理解した

最初にCopilotから「日常生活の例えで説明してみて」と言われて、こんな例えを思いつきました。

> 同期は注文してからカウンターで待ち続けること。
> 非同期は注文したら席に座って本を読みながら待つこと。

これがほぼ正解で、整理するとこうなります。

**同期（sync）**
- 「コーヒーください」→ カウンターでじっと待つ → 受け取る
- 待っている間、他のことは何もできない

**非同期（async）**
- 「コーヒーとケーキください」→ 席に座って本を読む
- 「コーヒーできました〜」→ 受け取る、また本を読む
- 待っている間も他のことができる

---

## Futureとは「注文票」だった

Rust では、非同期処理の単位を **`Future`** と呼びます。

`async fn` を呼び出すと、処理そのものは実行されず、「将来実行される予定の計算」を表す **`Future` オブジェクトが返ってくるだけ**です。

```rust
async fn hello() -> String {
    String::from("Hello, async world!")
}

let future = hello(); // ← この時点では何も実行されていない！
                      //   注文票ができただけ
```

実際に試したところ、`future` を `println!("{}", future)` しようとすると：

```
error[E0277]: `impl Future<Output = String>` doesn't implement `std::fmt::Display`
```

「String じゃなくて Future が返ってきたから表示できない」というエラーでした。
`async fn` が `Future` を返すというのが、エラーで一発で確認できました。

---

## 「誰が」Futureを実行するのか

`Future` は注文票なので、誰かが「これ処理してください」と厨房（executor）に渡す必要があります。

| 役割 | 喫茶店の例え | Rustでの正体 |
|---|---|---|
| 仕組みのルール | 番号札システム | `Future` trait（Rust標準） |
| 処理する係 | 厨房スタッフ | executor |
| 具体的な実装 | 実際の喫茶店 | **Tokio**（外部クレート） |

ここで重要な発見がありました。**Rust 標準ライブラリは `Future` の仕組みを定義するだけで、実行する executor は持っていない**のです。

```
Rust 標準ライブラリ
  └─ Future trait を定義（ルールだけ）
  └─ async/await 構文を提供

Tokio（外部クレート）
  └─ executor を持つ
  └─ Future を poll し続ける
  └─ 完了したら次の処理へ進める
```

---

## .awaitの正体

`.await` は「executor よ、この Future が終わるまで待ってくれ」という合図です。

```rust
let coffee = make_coffee().await;
// ↑ executor に「これを poll してくれ。終わったら戻ってきて」という指示
```

`.await` なしで実行しようとするとエラーになります：

```rust
fn main() {
    let result = hello().await; // ← エラー！
}
```

```
error[E0728]: `await` is only allowed inside `async` functions and blocks
this is not `async`
```

`.await` は `async` 関数の中でしか使えません。
なぜなら、`.await` で待っている間に自分自身も一時停止して他のタスクに切り替わる必要があるからです。一時停止できるのは `async` 関数だけです。

---

## #[tokio::main] でTokioを起動する

`Cargo.toml` に Tokio を追加します。

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

TOML のインラインテーブルは1行で書く必要があるので注意（複数行は構文エラー）。

`#[tokio::main]` アトリビュートを使うと、Tokio の executor を起動した上で `async fn main` を実行してくれます：

```rust
#[tokio::main]
async fn main() {
    let result = hello().await; // ← これで動く！
    println!("{}", result);
}

async fn hello() -> String {
    String::from("Hello, async world!")
}
```

実行結果：
```
Hello, async world!
```

---

## thread::sleep vs tokio::time::sleep

2秒待つタスクを2つ動かして、かかる時間を比べました。

**thread::sleep（同期）**

```rust
thread::sleep(Duration::from_secs(2)); // スレッドをブロック
println!("タスク1完了");
thread::sleep(Duration::from_secs(2));
println!("タスク2完了");
// → 合計4秒かかる
```

**tokio::time::sleep（非同期）**

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    tokio::join!(task1(), task2()); // 2つを並行実行
}

async fn task1() {
    sleep(Duration::from_secs(2)).await; // 待つ間、他のタスクへ切り替え
    println!("タスク1完了");
}

async fn task2() {
    sleep(Duration::from_secs(2)).await;
    println!("タスク2完了");
}
// → 合計2秒で終わる！
```

| | `thread::sleep` | `tokio::time::sleep` |
|---|---|---|
| 待ち方 | スレッドをブロック | executor に制御を返す |
| 待っている間 | 他のタスクは動けない | 他のタスクが動く |
| 2タスクの合計時間 | 4秒 | 2秒 |

### 並列と並行の違いも学んだ

- **並列（parallel）**: 複数スレッドが同時に動く（Phase 2 の `thread::spawn`）
- **並行（concurrent）**: 1スレッドで切り替えながら進む（Tokio の `async/await`）

Tokio はデフォルトで1スレッド上でタスクを切り替えています。スレッドを増やさなくても、多くのタスクをさばけるのが非同期の強みです。

---

## まとめ

- `Future` は「将来実行される予定の計算」を表すオブジェクト（注文票）
- `async fn` を呼び出しただけでは何も実行されない
- executor（Tokio）が `Future` を poll して初めて動く
- `.await` は executor に「これを処理してくれ」と渡す合図
- `#[tokio::main]` = Tokio executor を起動する便利な記法
- `tokio::time::sleep` はスレッドをブロックせず、待つ間に他のタスクが動ける

次の記事では、いよいよ同期版チャットサーバーを Tokio を使った非同期版へ書き換えていきます。
