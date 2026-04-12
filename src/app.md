# Appの作成

main.rsを以下のように書き換えます。

```rust
#![cfg_attr(not(debug_assertions), windows_subsystem = "windows")]
use bevy::prelude::*;

fn main() {
    App::new().add_plugins(DefaultPlugins).run();
}
```
一行目はリリースビルドの時にコンソールウィンドウを表示しないための設定です。

二行目は bevyクレートのネームスペースを使うためのものです。

`main`関数はプログラムのエントリーポイントです。ここでは`App::new()`でAppのインスタンスを作成し、`run()`で実行しています。

`add_plugins(DefaultPlugins)`は、ウィンドウの表示やキーボード入力、マウス入力など、ゲームに必要な基本的な機能をまとめて追加するためのものです。

これで`cargo run`を実行すると、ウィンドウが表示されます。(`cargo run`は`cargo build`してから`target/debug/invader_tutorial.exe`を実行するのと同じです。)

![app](./img/app.png)