---
title: "Rustでチャットサーバーを作る #11 — tracing でログを本格化する"
emoji: "📝"
type: "tech"
topics: ["rust"]
published: false
---

# Rustでチャットサーバーを作る #11 — tracing でログを本格化する

`println!` でログを出していましたが、本番を意識した `tracing` クレートに切り替えました。「タイムスタンプを付けたい」「環境によってログ量を変えたい」という要望が一気に解決しました。

---

## println! の何が問題だったのか

```
接続あり: 192.168.1.5:12345
切断: 192.168.1.5:12345
```

これだけだと困ることが3つあります：

- **いつ起きたか分からない**（タイムスタンプなし）
- **重要度が分からない**（通常の接続もエラーも同じ形式）
- **本番では止められない**（消すにはコードを変えるしかない）

---

## tracing と tracing-subscriber — 2つに分かれている理由

`tracing` クレートは「ログを発行するルール」だけを定義します。「どこに出すか」は知りません。

実際に出力するのは `tracing-subscriber` の役割です。

```
info!("接続あり")          ← tracing の仕事（叫ぶだけ）
      ↓
[env-filter で判断]        ← tracing-subscriber の仕事
      ↓
ターミナルに表示            ← tracing-subscriber の仕事
```

出力先をファイルや JSON に変えたくなっても、`info!` と書いたコードは一切変えなくて済みます。これは `Future` trait（ルール）と Tokio（実装）の分離と同じ考え方です。

---

## Cargo.toml への追加

```bash
cargo add tracing
cargo add tracing-subscriber --features env-filter
```

`tracing` にバージョン `1` はなく、`0.1` 系です。`cargo add` を使うと自動で最新安定版を追加してくれます。

`env-filter` feature を付けることで `RUST_LOG` 環境変数でログレベルを切り替えられるようになります。

---

## features とは

`features` はクレートの「オプション機能」を選んでコンパイルする仕組みです：

```toml
# 最小構成
tracing-subscriber = "0.3"

# env-filter 機能を追加
tracing-subscriber = { version = "0.3", features = ["env-filter"] }

# tokio のように全機能まとめた "full" があるクレートも
tokio = { version = "1", features = ["full"] }
```

使わない機能をコンパイルしないので、ビルド時間とバイナリサイズを抑えられます。

---

## 初期化と使い方

`main` の最初に subscriber を初期化します：

```rust
tracing_subscriber::fmt::init();
```

あとは `println!` を `info!` などに置き換えるだけです：

```rust
use tracing::info;

info!("サーバー起動: 0.0.0.0:8080");
info!("接続あり: {}", addr);
info!("切断: {}", addr);
```

---

## ログレベルで出力を切り替える

ログレベルには順番があります：

```
error > warn > info > debug > trace
```

`RUST_LOG` 環境変数で実行時に切り替えられます：

```bash
RUST_LOG=info cargo run    # info 以上を表示
RUST_LOG=warn cargo run    # warn・error だけ表示
RUST_LOG=debug cargo run   # 詳細ログも表示
cargo run                  # RUST_LOG なし → デフォルト（error のみ）
```

実際に試すとこんなログが出ました：

```
2026-05-16T09:01:47.960140Z  INFO chat_server: サーバー起動: 0.0.0.0:8080
2026-05-16T09:01:55.467216Z  INFO chat_server: 接続あり: 127.0.0.1:49986
```

タイムスタンプ・ログレベル・クレート名が自動で付きます。

`RUST_LOG=warn cargo run` にすると `info!` のログが一切出なくなることも確認しました。

---

## まとめ

- `tracing` = ログを発行するルール（`info!`, `warn!` マクロ）
- `tracing-subscriber` = 実際に出力する実装
- 2つに分かれているのは「出力先を自由に差し替えるため」
- `features` = クレートのオプション機能を選んでコンパイルする仕組み
- `RUST_LOG` 環境変数でログレベルを実行時に切り替えられる
- コードを変えずに本番・開発でログ量を調整できる

次の記事では、`#[tokio::test]` を使った非同期テストを書きます。
