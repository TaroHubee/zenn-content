---
title: "RustのスマートポインタBox/Rc/Arcを手を動かして理解する"
emoji: "📦"
type: "tech"
topics: ["rust"]
published: true
---

# RustのスマートポインタBox/Rc/Arcを手を動かして理解する

Rustには「スマートポインタ」と呼ばれる型が複数あります。`&` 参照と違い、所有権を持ちながら追加の機能を提供します。最初は「どれを使えばいいか分からない」状態になりますが、用途を整理すれば迷わなくなります。

---

## Box\<T\> — ヒープに値を置く

`Box<T>` は値をヒープ上に置くためのシンプルなポインタです。

```rust
let b = Box::new(5); // 5 をヒープに置く
println!("{}", b);   // 5（自動的に参照が外れる）
```

### いつ使うか

**①コンパイル時にサイズが決まらない型（再帰的な型）**

```rust
// NG: サイズが不明でコンパイルエラー
enum List {
    Cons(i32, List), // List のサイズが無限になる
    Nil,
}

// OK: Box でヒープに置けばポインタサイズ（固定）になる
enum List {
    Cons(i32, Box<List>),
    Nil,
}
```

**②大きな値を「移動」でなく参照で扱いたい**

```rust
let large_data = Box::new([0u8; 1_000_000]); // ヒープに1MB確保
// スタックで1MBを移動コピーするより効率的
```

---

## Rc\<T\> — 複数の所有者（シングルスレッド）

Rustの通常の所有権では「持ち主は1人」ですが、`Rc<T>`（Reference Counted）を使うと**複数の所有者**を持てます。

```rust
use std::rc::Rc;

let a = Rc::new(String::from("hello"));
let b = Rc::clone(&a); // 参照カウントを増やす（データはコピーしない）
let c = Rc::clone(&a);

println!("参照カウント: {}", Rc::strong_count(&a)); // 3
println!("{}", a); // "hello"
println!("{}", b); // "hello"
// a, b, c が全て drop されるとデータが解放される
```

`Rc::clone` は参照カウントを増やすだけで、データ自体はコピーしません。

### 制約：シングルスレッドのみ

`Rc` は**スレッド間で共有できません**。マルチスレッドには次の `Arc` を使います。

---

## Arc\<T\> — 複数の所有者（マルチスレッド）

`Arc<T>`（Atomically Reference Counted）は `Rc` のスレッドセーフ版です：

```rust
use std::sync::Arc;
use std::thread;

let data = Arc::new(vec![1, 2, 3]);

let data2 = Arc::clone(&data);
let handle = thread::spawn(move || {
    println!("スレッド内: {:?}", data2);
});

handle.join().unwrap();
println!("メイン: {:?}", data);
```

`Arc` は参照カウントをアトミック操作で更新するため、複数スレッドから同時にクローンしても安全です。ただしその分 `Rc` より少しコストがかかります。

---

## Arc\<Mutex\<T\>\> — 複数スレッドから変更可能なデータ

`Arc` は共有できますが、中の値の変更には `Mutex`（排他ロック）が必要です：

```rust
use std::sync::{Arc, Mutex};
use std::thread;

let counter = Arc::new(Mutex::new(0));
let mut handles = vec![];

for _ in 0..10 {
    let counter = Arc::clone(&counter);
    let handle = thread::spawn(move || {
        let mut num = counter.lock().unwrap(); // ロック取得
        *num += 1;
    }); // ロック解放（num が drop される）
    handles.push(handle);
}

for handle in handles { handle.join().unwrap(); }
println!("結果: {}", *counter.lock().unwrap()); // 10
```

---

## RefCell\<T\> — 実行時の借用チェック

コンパイル時ではなく**実行時**に借用ルールをチェックしたい場合に使います：

```rust
use std::cell::RefCell;

let data = RefCell::new(vec![1, 2, 3]);

// 複数の不変借用は OK
let r1 = data.borrow();
let r2 = data.borrow();
println!("{:?} {:?}", r1, r2);
drop(r1); drop(r2);

// 可変借用
data.borrow_mut().push(4);
println!("{:?}", data.borrow()); // [1, 2, 3, 4]
```

`borrow()` や `borrow_mut()` が実行時に借用ルールを確認し、違反すればパニックします。`Rc<RefCell<T>>` の組み合わせで「複数の所有者から変更可能」を実現できます。

---

## どれを使うか

| 状況 | 使う型 |
|---|---|
| ヒープに置きたい / 再帰的な型 | `Box<T>` |
| 複数所有者・シングルスレッド・読み取り専用 | `Rc<T>` |
| 複数所有者・マルチスレッド・読み取り専用 | `Arc<T>` |
| 複数所有者・マルチスレッド・変更あり | `Arc<Mutex<T>>` |
| 複数所有者・シングルスレッド・変更あり | `Rc<RefCell<T>>` |

---

## まとめ

- `Box<T>` — ヒープに置く。サイズ不明な型・大きなデータに
- `Rc<T>` — 参照カウントで複数所有者（シングルスレッドのみ）
- `Arc<T>` — `Rc` のスレッドセーフ版
- `Mutex<T>` — 排他ロックで複数スレッドから変更可能に
- `RefCell<T>` — 実行時の借用チェック（コンパイル時を実行時に先延ばし）

実際のコードでは `Arc<Mutex<T>>` と `Arc<broadcast::Sender<T>>` が非同期プログラミングでよく登場します。
