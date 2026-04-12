# Playing画面の作成

Playing画面は、ゲームがプレイされている画面です。長くなるので、Playing画面全体の流れ、プレイヤー、敵、球、HP、これらで共通して扱うもの、と分けて説明しますが、横断することもあるのでとりあえず以下のファイルを作ってください。
```
src/playing/mod.rs
src/playing/player.rs
src/playing/enemy.rs
src/playing/bullet.rs
src/playing/hp.rs
src/playing/utils.rs
```

また、`main.rs`に`mod playing;`を追加します。
```rust
mod state;
mod home;
mod playing;
```
