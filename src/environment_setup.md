# 環境構築

検証済み環境であればほぼ確実にできると思いますが、多分他の環境でも似たような感じでできると思います。

**検証済み環境**

- Windows 11
- Rust 1.92.0
- Bevy 0.18.1

## Rustのインストール

[https://www.rust-lang.org/tools/install](https://www.rust-lang.org/tools/install)に行き、手順に従ってインストールします。この辺のは公式に日本語のわかりやすい説明があるので[The Rust Programming Language 日本語版](https://doc.rust-jp.rs/book-ja/ch01-01-installation.html#windows%E3%81%A7rustup%E3%82%92%E3%82%A4%E3%83%B3%E3%82%B9%E3%83%88%E3%83%BC%E3%83%AB%E3%81%99%E3%82%8B)を参考にしてください。

## Rust Analyzerのインストール

VSCodeを使っている場合は、拡張機能からRust Analyzerをインストールします。これがあるだけでコードの補完やエラーチェックが効くので入れておくといいです。

## Bevyのインストール

あとでCargo.tomlにbevyを追加するだけでいいので、ここでは割愛します。

## プロジェクトの作成

```bash
cargo new invader_tutorial
cd invader_tutorial
```

このフォルダを適当なテキストエディタで開きます。VSCodeがおすすめですが、お好きなもので大丈夫です。

## Bevyの追加

Cargo.tomlに以下を追加します。

```toml
[dependencies]
bevy = "0.18.1"
```

.cargo/config.tomlに以下を追加します。

```toml
[target.x86_64-pc-windows-msvc]
linker = "rust-lld.exe"
```
まだCargo.tomlに以下を追加します
```toml
[profile.dev]
opt-level = 1
[profile.dev.package."*"]
opt-level = 3
[features]
dynamic_linking = ["bevy/dynamic_linking"]
```
これはビルドを速くするための設定です。

環境構築はこれで完了です。