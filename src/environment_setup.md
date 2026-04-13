# 環境構築

以下の手順に従って、開発環境を構築します。検証済み環境以外でも、同様の手順で構築可能です。

**検証済み環境**

- Windows 11
- Rust 1.92.0
- Bevy 0.18.1

## Rustのインストール

[公式サイト](https://www.rust-lang.org/tools/install)の手順に従ってインストールしてください。インストールの詳細については、[The Rust Programming Language 日本語版](https://doc.rust-jp.rs/book-ja/ch01-01-installation.html#windows%E3%81%A7rustup%E3%82%92%E3%82%A4%E3%83%B3%E3%82%B9%E3%83%88%E3%83%BC%E3%83%AB%E3%81%99%E3%82%8B)が参考になります。

## Rust Analyzerのインストール

VSCodeを使用している場合は、拡張機能から「Rust Analyzer」をインストールしてください。コード補完やリアルタイムのエラーチェックが可能になるため、導入を強く推奨します。

## Bevyのインストール

後ほど `Cargo.toml` に Bevy を追記するため、この段階での個別のインストール作業は不要です。

ターミナル（またはコマンドプロンプト）で以下のコマンドを実行し、プロジェクトを作成します。

```bash
cargo new invader_tutorial
cd invader_tutorial
```

作成されたフォルダを、VSCodeなどの任意のテキストエディタで開いてください。

## Bevyの追加

`Cargo.toml` に以下を追記します。

```toml
[dependencies]
bevy = "0.18.1"
```

次に、`.cargo/config.toml` （フォルダがない場合は作成してください）に以下を追記します。

```toml
[target.x86_64-pc-windows-msvc]
linker = "rust-lld.exe"
```
また、`Cargo.toml` に以下のプロファイル設定を追加します。
```toml
[profile.dev]
opt-level = 1
[profile.dev.package."*"]
opt-level = 3
[features]
dynamic_linking = ["bevy/dynamic_linking"]
```
これは、ビルド速度を向上させるための設定です。

以上で環境構築は完了です。
