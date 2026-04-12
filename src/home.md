# Home画面を作る

Home画面は、ゲームが始まったときに開かれる画面で、`Start`ボタンが配置されており、押すと`Playing`画面に遷移します。

まずUIで使うテキストのためのフォントを準備してください。ここでは`fonts/NotoSansJP-Bold.ttf`を`src/`の直下に置きます。次に`main.rs`の`main`関数の中を以下のように変更します。

```rust
let mut app = App::new();
app.add_plugins(DefaultPlugins); // DefaultPluginsを先に足す
bevy::asset::embedded_asset!(app, "fonts/NotoSansJP-Bold.ttf"); // フォントをassetとして埋め込む
app.add_plugins(state::GameStatePlugin)
    .add_plugins(home::HomePlugin)
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

では、Home画面のUI(Startボタン)を実装していきます。

まず`StartButton`という名前のコンポーネントを作ります。
```rust
#[derive(Component)]
pub struct StartButton;
```
また、UIのバンドルを作るための関数を作ります。
```rust
fn start_button_bundle(assets_server: Res<AssetServer>) -> impl Bundle {
    (
        DespawnOnExit(state::GameState::Home), // Home画面から抜けたときにDespawnする
        Node {
            width: percent(100),
            height: percent(100),
            align_items: AlignItems::Center,
            justify_content: JustifyContent::Center,
            ..default()
        },
        // StartButtonはNodeのchildrenとして作る
        children![(
            Button,
            StartButton,
            Node {
                width: percent(20),
                height: percent(10),
                justify_content: JustifyContent::Center,
                align_items: AlignItems::Center,
                border_radius: BorderRadius::MAX,
                ..default()
            },
            BorderColor::all(Color::WHITE),
            BackgroundColor(Color::BLACK),
            children![(
                Text::new("Start"),
                TextFont {
                    font: assets_server
                        .load("embedded://invader_tutorial/fonts/NotoSansJP-Bold.ttf"),
                    font_size: 40.0,
                    ..default()
                },
                TextLayout::new_with_justify(Justify::Center),
                TextColor::BLACK,
            )]
        )],
    )
}
```
これをHome画面に来たときにspawnするようにします。
```rust
fn setup_home_screen(mut commands: Commands, assets_server: Res<AssetServer>) {
    // spawn a camera
    commands.spawn((
        Camera3d::default(),
        Transform::from_xyz(0.0, 10.0, 0.0).looking_at(Vec3::ZERO, Vec3::Y),
        DespawnOnExit(state::GameState::Home),
    ));
    // spawn UI
    commands.spawn(start_button_bundle(assets_server));
}
```
`setup_home_screen`システムを`HomePlugin`の`build`メソッドに追加します。
```rust
impl Plugin for HomePlugin {
    fn build(&self, app: &mut App) {
        app.add_systems(OnEnter(state::GameState::Home), setup_home_screen);
    }
}
```

`StartButton`が表示されるようになりましたが、押しても何も起こりません。そこで、`StartButton`が押されたときに`Playing`画面に遷移するようにします。
```rust
type StartButtonInputs = (Changed<Interaction>, With<StartButton>);
fn update_start_button(
    mut query: Query<(&Interaction, &mut BackgroundColor), StartButtonInputs>,
    mut game_state: ResMut<NextState<state::GameState>>,
) {
    for (interaction, mut background_color) in query.iter_mut() {
        match interaction {
            Interaction::Pressed => {
                background_color.0 = Color::srgb(0.5, 0.5, 0.5);
                game_state.set(state::GameState::Playing);
            }
            Interaction::Hovered => {
                background_color.0 = Color::srgb(0.7, 0.7, 0.7);
            }
            Interaction::None => {
                background_color.0 = Color::srgb(0.9, 0.9, 0.9);
            }
        }
    }
}
```
`update_start_button`システムを`HomePlugin`の`build`メソッドに追加します。
```rust
impl Plugin for HomePlugin {
    fn build(&self, app: &mut App) {
        app.add_systems(OnEnter(state::GameState::Home), setup_home_screen)
            .add_systems(
                Update,
                update_start_button.run_if(in_state(state::GameState::Home)),
            );
    }
}
```
これでHome画面が完成しました。