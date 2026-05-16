---
title: "Rustのトレイトとジェネリクスを手を動かして理解する"
emoji: "🔧"
type: "tech"
topics: ["rust"]
published: true
---

# Rustのトレイトとジェネリクスを手を動かして理解する

「トレイト」と「ジェネリクス」はRustの型システムの核心です。他言語の「インターフェース」や「テンプレート」に近いですが、Rustならではの特徴があります。

---

## トレイト — 「できること」を定義する

トレイトは「この型はこういう操作ができる」というルールを定義します。

```rust
trait Greet {
    fn hello(&self) -> String;
}
```

これだけでは何もできません。具体的な型に**実装**します：

```rust
struct User {
    name: String,
}

impl Greet for User {
    fn hello(&self) -> String {
        format!("こんにちは、{}！", self.name)
    }
}

let user = User { name: "Taro".to_string() };
println!("{}", user.hello()); // "こんにちは、Taro！"
```

### デフォルト実装

トレイトにデフォルトの実装を持たせることができます：

```rust
trait Greet {
    fn name(&self) -> &str;

    // デフォルト実装（オーバーライド可能）
    fn hello(&self) -> String {
        format!("こんにちは、{}！", self.name())
    }
}

struct User { name: String }

impl Greet for User {
    fn name(&self) -> &str {
        &self.name
    }
    // hello() はデフォルト実装をそのまま使う
}
```

### よく使う標準トレイト

Rustには便利な組み込みトレイトが多数あります：

| トレイト | 意味 | derive で自動実装 |
|---|---|---|
| `Debug` | `{:?}` でデバッグ表示 | ✅ |
| `Clone` | `.clone()` でコピー | ✅ |
| `PartialEq` | `==` で比較 | ✅ |
| `Display` | `{}` で表示 | ❌（手動実装） |
| `Iterator` | イテレータ | ❌（手動実装） |

#### `Debug` トレイトの内部

`Debug` トレイトの実体は `std::fmt` モジュールにあるこのトレイトです：

```rust
// 標準ライブラリの定義（抜粋）
pub trait Debug {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result;
}
```

`{:?}` でフォーマットするとき、Rustは内部でこの `fmt` を呼び出しています：

```rust
let s = format!("{:?}", value);
// ↑ 内部では ↓ と等価
let s = {
    let mut buf = String::new();
    value.fmt(&mut std::fmt::Formatter::new(&mut buf)); // fmt() を呼ぶだけ
    buf
};
```

`#[derive(Debug)]` を付けると、コンパイラが自動で `impl Debug` を展開します。
手書きすると次のようになります：

```rust
use std::fmt;

struct User {
    name: String,
    age: u32,
}

// #[derive(Debug)] が生成するコードのイメージ
impl fmt::Debug for User {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        // "User { name: \"Taro\", age: 30 }" という文字列を組み立てる
        f.debug_struct("User")          // 構造体名
            .field("name", &self.name)  // フィールド名と値
            .field("age", &self.age)
            .finish()
    }
}

fn main() {
    let user = User { name: "Taro".to_string(), age: 30 };
    println!("{:?}", user);
    // → User { name: "Taro", age: 30 }

    println!("{:#?}", user); // 整形表示（#付き）
    // → User {
    //       name: "Taro",
    //       age: 30,
    //   }
}
```

`Display` トレイトは自分で好きな文字列を組み立てる手動実装のみですが、構造は同じです：

```rust
impl fmt::Display for User {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        // "{}" で表示したい文字列を write! で書く
        write!(f, "{}({}歳)", self.name, self.age)
    }
}

println!("{}", user);  // → Taro(30歳)
println!("{:?}", user); // → User { name: "Taro", age: 30 }（Debug のまま）
```

まとめると `{:?}` や `{}` でフォーマットするとき、Rustは **対応するトレイトの `fmt` メソッドを呼ぶだけ**です。`#[derive]` はその `fmt` の実装をコンパイラが自動生成するショートカットです。

---

## ジェネリクス — 「型を後で決める」

同じロジックを複数の型に使いたいとき、**ジェネリクス**を使います。

```rust
// i32 専用
fn largest_i32(list: &[i32]) -> i32 {
    let mut largest = list[0];
    for &item in list {
        if item > largest { largest = item; }
    }
    largest
}

// f64 専用
fn largest_f64(list: &[f64]) -> f64 { ... }
```

これをジェネリクスで1つにまとめると：

```rust
fn largest<T: PartialOrd>(list: &[T]) -> &T {
    let mut largest = &list[0];
    for item in list {
        if item > largest { largest = item; }
    }
    largest
}
```

`T: PartialOrd` は「`T` は `>` で比較できる型に限る」というトレイト境界です。どんな型でも受け付けるわけにはいかない（比較できない型に `>` は使えない）ため、制約を付けます。

### 構造体のジェネリクス

```rust
struct Point<T> {
    x: T,
    y: T,
}

let int_point = Point { x: 5, y: 10 };
let float_point = Point { x: 1.0, y: 4.0 };
```

---

## トレイト境界 — ジェネリクスに制約を付ける

ジェネリクスの型に「〜ができること」を要求するのがトレイト境界です：

```rust
// T は Display トレイトを実装している型に限る
fn print_item<T: std::fmt::Display>(item: T) {
    println!("{}", item);
}
```

複数のトレイトを要求するときは `+` で繋げます：

```rust
fn print_debug<T: std::fmt::Display + std::fmt::Debug>(item: T) {
    println!("Display: {}", item);
    println!("Debug: {:?}", item);
}
```

`where` 句を使うと長い境界を読みやすく書けます：

```rust
fn compare_and_print<T>(a: T, b: T)
where
    T: PartialOrd + std::fmt::Display,
{
    if a > b {
        println!("{} > {}", a, b);
    } else {
        println!("{} <= {}", a, b);
    }
}
```

---

## impl Trait — 引数・戻り値にトレイトを使う

関数の引数や戻り値にトレイトを直接指定できます：

```rust
// 引数: Display を実装している何らかの型
fn print_it(item: &impl std::fmt::Display) {
    println!("{}", item);
}

// 戻り値: Iterator を実装している何らかの型
fn make_counter() -> impl Iterator<Item = i32> {
    (0..10)
}
```

### トレイト境界 `<T: Trait>` との使い分け

引数位置では、`impl Trait` とトレイト境界はほぼ同じ意味です。ただし**複数の引数を同じ型に縛りたい**ときはトレイト境界が必要です：

```rust
// ❌ impl Trait: a が i32、b が f64 でもコンパイルが通ってしまう
fn compare(a: impl PartialOrd, b: impl PartialOrd) { ... }

// ✅ トレイト境界: T で縛るので a と b は必ず同じ型
fn compare<T: PartialOrd>(a: T, b: T) { ... }
```

| 場面 | 使うもの |
|---|---|
| 引数1つ、型名が長い | `impl Trait`（簡潔） |
| 複数引数を同じ型に縛りたい | `<T: Trait>` |

### 戻り値で `impl Trait` が必要な理由

「型名の省略記法」ではなく、**そもそも型名が書けない・非現実的なケース**があるためです。

**クロージャには型名がない** — クロージャはコンパイラが内部で生成する「名前のない型」です。外の変数をキャプチャすると、その変数をフィールドに持つ無名構造体として扱われます：

```rust
fn make_adder(n: i32) -> impl Fn(i32) -> i32 {
    |x| x + n  // n をキャプチャ → 型名が存在しないので impl Trait しか書けない
}

// コンパイラ内部のイメージ
struct Closure { n: i32 }  // キャプチャした変数をフィールドに持つ
impl Fn(i32) -> i32 for Closure {
    fn call(&self, x: i32) -> i32 { x + self.n }
}
```

**イテレータの連鎖は型名が非現実的なほど長い**：

```rust
// impl Trait なら短く書ける
fn double_values(v: &[i32]) -> impl Iterator<Item = i32> + '_ {
    v.iter().map(|x| x * 2)
}

// 実際の型名を書くと…（読む気が失せる）
fn double_values(v: &[i32]) -> std::iter::Map<std::slice::Iter<'_, i32>, fn(&i32) -> i32> {
    v.iter().map(|x| x * 2)
}
// filter や take が増えるとさらに何十文字にも膨れ上がる
```

`impl Trait` は「コンパイル時に具体的な型が決まる」ため、動的ディスパッチより高速です。

---

## dyn Trait — 動的ディスパッチ

実行時まで型が決まらない場合は `dyn Trait` を使います：

```rust
trait Animal {
    fn sound(&self) -> &str;
}

struct Dog;
struct Cat;

impl Animal for Dog { fn sound(&self) -> &str { "ワン" } }
impl Animal for Cat { fn sound(&self) -> &str { "ニャー" } }

// Vec に異なる型を混在させる
let animals: Vec<Box<dyn Animal>> = vec![
    Box::new(Dog),
    Box::new(Cat),
];

for animal in &animals {
    println!("{}", animal.sound());
}
```

`Box<dyn Animal>` は「Animal トレイトを実装した何らかの型のポインタ」です。型はヒープ上に置かれ、実行時に解決されます。

### `impl Trait` vs `Box<dyn Trait>` の選び方

```rust
// ✅ impl Trait: 条件分岐なし・型が1種類・速度優先
fn make_counter() -> impl Iterator<Item = i32> {
    (0..10)
}

// ✅ Box<dyn Trait>: 条件によって返す型が変わる
fn make_animal(kind: &str) -> Box<dyn Animal> {
    if kind == "dog" { Box::new(Dog) } else { Box::new(Cat) }
    // Dog と Cat はサイズが違う可能性があり、コンパイル時にサイズが決まらない
    // → ポインタ（固定8バイト）をスタックに、実データをヒープに置く
}

// ❌ impl Trait では条件分岐できない
fn make_animal(kind: &str) -> impl Animal {  // コンパイルエラー
    if kind == "dog" { Dog } else { Cat }     // 型が一致しない
}
```

| | `impl Trait` | `Box<dyn Trait>` |
|---|---|---|
| 型の解決 | コンパイル時 | 実行時（vtable） |
| ヒープ | 不要 | 必要 |
| 速度 | 速い（直接呼び出し） | やや遅い（間接呼び出し） |
| 条件分岐で型を切り替え | ❌ | ✅ |
| Vec などに異なる型を混在 | ❌ | ✅ |

---

## まとめ

| 機能 | 用途 |
|---|---|
| `trait` | 「できること」の定義 |
| `impl Trait for Type` | 具体的な実装 |
| `#[derive]` | 標準トレイトの自動実装 |
| ジェネリクス `<T>` | 型を後で決める汎用コード |
| トレイト境界 `T: Trait` | ジェネリクスに制約を付ける |
| `impl Trait` | 静的ディスパッチ（高速） |
| `dyn Trait` | 動的ディスパッチ（柔軟） |

トレイトとジェネリクスを組み合わせることで、重複のない汎用的なコードを型安全に書けます。
