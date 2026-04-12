# Playing画面全体の流れとUI

ここではPlugin、一時停止ボタン、再開ボタン、スコアテキスト、リザルト画面への遷移を実装します。

まず、`PlayingPlugin`を作成します。
```rust
use bevy::prelude::*;

pub struct PlayingPlugin;

impl Plugin for PlayingPlugin {
    fn build(&self, app: &mut App) {
        
    }
}
```

次に、`PlayingPlugin`を`main.rs`の`app`に追加します。
```rust
app.add_plugins(playing::PlayingPlugin);
```
