# Stateの追加

ゲーム内の画面遷移（タイトル画面、プレイ中、結果表示など）を管理するために、`src/state.rs` を作成して以下の通り記述します。

```rust
use bevy::prelude::*;

#[derive(Clone, Copy, PartialEq, Eq, Hash, Debug, Default, States)]
pub enum GameState {
    /// タイトル画面
    #[default]
    Home,
    /// ゲームプレイ中
    Playing,
    /// 結果表示画面
    Result,
}

pub struct GameStatePlugin;

impl Plugin for GameStatePlugin {
    fn build(&self, app: &mut App) {
        // AppにGameStateを登録
        app.init_state::<GameState>();
    }
}
```

`GameState`はゲームの現在の状態を表す列挙型です。
`#[derive(Clone, Copy, PartialEq, Eq, Hash, Debug, Default, States)]`は、`GameState`が`Clone`、`Copy`、`PartialEq`、`Eq`、`Hash`、`Debug`、`Default`、`States`トレイトを実装するためのものです。
`States`は、`GameState`がゲームの状態であることを示します。
`#[default]`は、`GameState`のデフォルト値を`Home`に設定するためのものです。

`GameStatePlugin`は`GameState`を初期化するためのプラグインです。プラグインを作るには`Plugin`トレイトを実装する必要があります。また、`build`メソッドを定義する必要があります。`build`メソッドの引数`app`は`App`のインスタンスです。この`app`に`GameState`を追加します。

`init_state::<GameState>()`は、`GameState`を初期化するためのメソッドです。

次に`main.rs`の`app`に`GameStatePlugin`を追加します。

```rust
mod state;
```
によって`state.rs`を読み込みます。

また、`app`に`GameStatePlugin`を追加します。

```rust
App::new()
    .add_plugins(DefaultPlugins)
    .add_plugins(state::GameStatePlugin) // ← これを追加
    .run();
```

これでGameStateが使えるようになりました。