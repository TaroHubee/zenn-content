---
title: "Rustでチャットサーバーを作る #12 — #[tokio::test] で非同期テストを書く"
emoji: "✅"
type: "tech"
topics: ["rust", "tokio"]
published: false
---

# Rustでチャットサーバーを作る #12 — #[tokio::test] で非同期テストを書く

全機能が実装できたので、テストを書いて仕上げます。「テストって難しそう」と思っていましたが、Rust のテストは関数に属性を付けるだけで始められます。

---

## Rust のテストの基本形

テストはソースファイルの末尾に直接書けます：

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_something() {
        assert_eq!(actual, expected);
    }
}
```

2つのキーワードの意味：

- **`#[cfg(test)]`** — `cargo test` のときだけコンパイルされる。`cargo build` には含まれない
- **`use super::*`** — `tests` モジュールは `message` モジュールの中に入れ子になっているため、`super::` で1つ上（`message` モジュール）の定義を全部取り込む

---

## #[test] と #[tokio::test] の違い

| | `#[test]` | `#[tokio::test]` |
|---|---|---|
| 対象 | 同期関数 | `async fn` |
| executor | 不要 | Tokio が自動起動 |
| 用途 | 純粋な計算のテスト | `.await` を使うテスト |

`#[tokio::main]` が main 関数用の executor 起動だったように、`#[tokio::test]` はテスト関数用の executor 起動です。`parse_message` は同期関数なので `#[test]` で十分です。

---

## parse_message のテスト

`assert_eq!` で比較するには、`Message` に `PartialEq` が必要です。`derive` に追加します：

```rust
#[derive(Debug, PartialEq)]
pub enum Message { ... }
```

テストは4ケース：

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_parse_message_chat() {
        assert_eq!(parse_message("Hello"), Message::Chat("Hello".to_string()));
    }

    #[test]
    fn test_parse_message_list() {
        assert_eq!(parse_message("/list"), Message::List);
    }

    #[test]
    fn test_parse_message_quit() {
        assert_eq!(parse_message("/quit"), Message::Quit);
    }

    #[test]
    fn test_parse_message_nick() {
        assert_eq!(parse_message("/nick Taro"), Message::Nick("Taro".to_string()));
    }
}
```

実行結果：

```
running 4 tests
test message::tests::test_parse_message_chat ... ok
test message::tests::test_parse_message_list ... ok
test message::tests::test_parse_message_nick ... ok
test message::tests::test_parse_message_quit ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

4つ全部パスしました。

---

## まとめ

- `#[test]` を付けるだけで `cargo test` の対象になる
- `#[cfg(test)]` でテストコードは本番バイナリに含まれない
- `use super::*` でテスト対象モジュールの定義を取り込む
- `assert_eq!` で比較するには `#[derive(PartialEq)]` が必要
- `async fn` のテストには `#[tokio::test]` を使う

これでチャットサーバーの全実装が完成しました。
