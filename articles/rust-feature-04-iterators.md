---
title: "Rustのイテレータとクロージャを手を動かして理解する"
emoji: "🔁"
type: "tech"
topics: ["rust"]
published: true
---

# Rustのイテレータとクロージャを手を動かして理解する

Rustのイテレータは非常に強力で、`for` ループよりも表現力が高く、かつパフォーマンスも同等かそれ以上です。クロージャと組み合わせることで、簡潔で読みやすいコードが書けます。

---

## クロージャ — 変数に入れられる関数

クロージャは「その場で定義できる小さな関数」です。変数に代入したり、引数として渡したりできます：

```rust
let add = |x, y| x + y;
println!("{}", add(2, 3)); // 5
```

通常の関数との違いは**周囲の変数を「キャプチャ」できる**点です。

**キャプチャとは**、クロージャが自分の外にある変数を取り込んで使えることです。通常の関数では外側の変数は見えませんが、クロージャは見えます：

```rust
let base = 10;

// ❌ 通常の関数: 外側の変数は見えない
fn add_base(x: i32) -> i32 {
    x + base  // エラー: base が見えない
}

// ✅ クロージャ: base をキャプチャして使える
let add_base = |x| x + base;
println!("{}", add_base(5)); // 15
```

これが機能する理由は、クロージャがコンパイラによって**キャプチャした変数をフィールドに持つ構造体**として扱われるからです：

```rust
// |x| x + base  のコンパイラ内部のイメージ
struct Closure {
    base: i32,  // キャプチャした変数を保持
}
// add_base(5) を呼ぶと self.base + 5 が実行される
```

### クロージャのキャプチャ方法

デフォルトでは**借用**でキャプチャするため、所有権は元の変数のままです。`move` を明示したときだけ所有権がクロージャに移ります：

```rust
let s = String::from("hello");

// ① 借用キャプチャ（デフォルト）— 所有権は s のまま
let print_s = || println!("{}", s);
print_s();
println!("{}", s); // ✅ s はまだ使える

// ② move キャプチャ — 所有権がクロージャに移る
let print_s = move || println!("{}", s);
print_s();
println!("{}", s); // ❌ s はもう使えない（所有権がクロージャに移った）
```

借用キャプチャはさらに2種類あり、コンパイラがクロージャの中身を見て**最小限の借用方法を自動選択**します：

```rust
let mut count = 0;

// 不変借用（&）— 読むだけ
let read = || println!("{}", count);
read();

// 可変借用（&mut）— 変更する場合
let mut increment = || count += 1;
increment();
println!("{}", count); // → 1（所有権はまだ count にある）
```

| 書き方 | キャプチャ方法 | 所有権の変化 |
|---|---|---|
| 普通のクロージャ（読むだけ） | 不変借用 `&` | なし |
| 普通のクロージャ（変更あり） | 可変借用 `&mut` | なし |
| `move` クロージャ | 所有権ごと移動 | クロージャに移る |

`move` が必要な典型例はスレッドです。スレッドは元の変数より長生きするかもしれないため、借用では安全が保証できません：

```rust
let msg = String::from("hello");
std::thread::spawn(move || {
    println!("{}", msg); // msg の所有権をスレッドに移動
});
```

---

## イテレータ — 要素を1つずつ処理する

`Vec`・配列・範囲・文字列・`HashMap` など様々な型でイテレータが使えますが、内部では2種類のトレイトが関係しています。

**`Iterator` トレイト** — `next()` メソッドを持ち、要素を1つずつ返せる型。`map` / `filter` などのメソッドはこのトレイトから来ています。

**`IntoIterator` トレイト** — 「`Iterator` に変換できる」型。`for` ループはこのトレイトを使っています。

型によって、どちらを実装しているかが異なります：

| 型 | `Iterator` | `IntoIterator` | 備考 |
|---|---|---|---|
| `Range`（`1..=5`） | ✅ 直接 | ✅ | そのまま `.map()` 等が使える |
| `Vec<T>` | ❌ | ✅ | `.iter()` で `Iter<T>` に変換 |
| 配列 `[T; N]` | ❌ | ✅ | `.iter()` で `Iter<T>` に変換 |
| `HashMap<K,V>` | ❌ | ✅ | `.iter()` で `(&K, &V)` のイテレータに変換 |
| `String` / `&str` | ❌ | ❌ | `.chars()` / `.bytes()` が必要 |

`Vec` を `for x in &v` と書けるのも、`&Vec<T>` が `IntoIterator` を実装しているからです。`.iter()` を呼ぶと内部で `IntoIterator` が呼ばれ、`Iterator` を実装した専用の型（`std::slice::Iter`）に変換されます。

### よく使うコレクションでの使い方

```rust
// Vec
let names = vec!["Alice", "Bob", "Charlie"];
let upper: Vec<String> = names.iter().map(|s| s.to_uppercase()).collect();
// ["ALICE", "BOB", "CHARLIE"]

// 配列（固定長）
let arr = [10, 20, 30, 40, 50];
let sum: i32 = arr.iter().sum(); // 150

// 範囲（1〜10 の偶数だけ集める）
let evens: Vec<i32> = (1..=10).filter(|x| x % 2 == 0).collect();
// [2, 4, 6, 8, 10]

// 文字列の各文字（母音を除く）
let no_vowels: String = "hello world"
    .chars()
    .filter(|c| !"aeiou".contains(*c))
    .collect(); // "hll wrld"
```

`HashMap` も同様にイテレートできます：

```rust
use std::collections::HashMap;

let mut scores: HashMap<&str, i32> = HashMap::new();
scores.insert("Alice", 90);
scores.insert("Bob", 75);
scores.insert("Carol", 88);

// 80点以上の名前だけ抽出
let high: Vec<&&str> = scores.iter()
    .filter(|(_, score)| **score >= 80)
    .map(|(name, _)| name)
    .collect();
```

### イテレータアダプタ

イテレータには変換・フィルターメソッドが多数あります：

```rust
let v = vec![1, 2, 3, 4, 5];

// map: 各要素を変換
let doubled: Vec<i32> = v.iter().map(|x| x * 2).collect();
// [2, 4, 6, 8, 10]

// filter: 条件に合う要素だけ残す
let evens: Vec<&i32> = v.iter().filter(|x| *x % 2 == 0).collect();
// [2, 4]

// map + filter を組み合わせる
let result: Vec<i32> = v.iter()
    .filter(|x| *x % 2 == 0)
    .map(|x| x * 10)
    .collect();
// [20, 40]
```

### よく使うメソッド

```rust
let v = vec![1, 2, 3, 4, 5];

v.iter().sum::<i32>()        // 15（合計）
v.iter().count()             // 5（個数）
v.iter().max()               // Some(5)
v.iter().min()               // Some(1)
v.iter().any(|x| *x > 3)    // true（1つでも条件を満たすか）
v.iter().all(|x| *x > 0)    // true（全て条件を満たすか）
v.iter().find(|x| **x == 3) // Some(3)
v.iter().position(|x| *x == 3) // Some(2)（インデックス）

// fold: 累積計算（sum の一般化）
let product = v.iter().fold(1, |acc, x| acc * x); // 120（積）
```

---

## iter() vs iter_mut() vs into_iter()

| メソッド | 要素の型 | 所有権 |
|---|---|---|
| `iter()` | `&T` | 借用（元のコレクションはそのまま） |
| `iter_mut()` | `&mut T` | 可変借用（要素を変更できる） |
| `into_iter()` | `T` | 所有権を取る（元のコレクションは消費） |

```rust
let mut v = vec![1, 2, 3];

// iter_mut: 要素を直接変更
for x in v.iter_mut() {
    *x *= 2;
}
// v = [2, 4, 6]

// into_iter: 所有権ごと消費
let doubled: Vec<i32> = v.into_iter().map(|x| x * 2).collect();
// v はもう使えない
```

---

## チェーンの遅延評価

イテレータアダプタは**遅延評価**されます。`.collect()` や `.sum()` などの「消費メソッド」を呼ぶまで実際の処理は行われません：

```rust
let v = vec![1, 2, 3, 4, 5];

// この時点では何も処理されない
let iter = v.iter()
    .filter(|x| *x % 2 == 0)
    .map(|x| x * 10);

// collect() を呼んで初めて処理が走る
let result: Vec<i32> = iter.collect(); // [20, 40]
```

これにより、`for` ループで中間コレクションを作る必要がなく、メモリ効率が良くなります。

---

## for ループ vs イテレータ

同じ処理を両方で書いてみます：

```rust
let v = vec![1, 2, 3, 4, 5];

// for ループ版
let mut result = Vec::new();
for x in &v {
    if x % 2 == 0 {
        result.push(x * 10);
    }
}

// イテレータ版
let result: Vec<i32> = v.iter()
    .filter(|x| *x % 2 == 0)
    .map(|x| x * 10)
    .collect();
```

どちらのパフォーマンスも同等です（コンパイラが最適化します）。イテレータ版は意図が明確に読み取れるため、Rustでは慣用的にこちらが使われます。

---

## まとめ

- クロージャ = 変数に代入・引数に渡せる関数
- クロージャは外側の変数をキャプチャできる（`move` で所有権ごと移動）
- `iter()` → 各要素を借用して順番に処理
- `map` / `filter` / `fold` などのアダプタを繋げられる
- イテレータは遅延評価（消費メソッドを呼ぶまで処理されない）
- `for` ループと同等のパフォーマンスで、より表現力が高い
