# Home画面を作る

まずUIで使うテキストのためのフォントを準備してください。ここでは`fonts/NotoSansJP-Bold.ttf`を`src/`の直下に置きます。次に`main.rs`の`main`関数の中を以下のように変更します。

```rust
let mut app = App::new();
bevy::asset::embedded_asset!(app, "fonts/NotoSansJP-Bold.ttf");
app.add_plugins(DefaultPlugins)
    .add_plugins(state::GameStatePlugin)
    .run();
```
これでフォントがassetとして埋め込まれました。

次に`home/mod.rs`を作成します。また、`main.rs`に`mod home;`を追加します。

```rust
// src/home/mod.rs
use crate::state;
use bevy::prelude::*;

pub struct HomePlugin;

impl Plugin for HomePlugin {
    fn build(&self, app: &mut App) {
        
    }
}
```
`HomePlugin`を`main.rs`の`app`に追加します。

```rust
app.add_plugins(home::HomePlugin);
```