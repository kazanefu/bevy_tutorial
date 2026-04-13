# Home画面（タイトル画面）の作成

Home画面は、アプリ起動時に最初に表示される画面です。中央に配置された `Start` ボタンをクリックすることで、ゲーム本編（Playing 画面）へと遷移します。

まずは、UIで使用するフォントの準備が必要です。ここでは、`fonts/NotoSansJP-Bold.ttf` というファイルを `src/` 直下に配置して使用します。次に、`main.rs` の `main` 関数を以下のように書き換えて、フォントアセットをアプリに埋め込みます。

```rust
let mut app = App::new();
app.add_plugins(DefaultPlugins); 

// フォントファイルをバイナリに埋め込み、アセットとして利用可能にする
bevy::asset::embedded_asset!(app, "fonts/NotoSansJP-Bold.ttf"); 

app.add_plugins(state::GameStatePlugin)
    .add_plugins(home::HomePlugin)
    .run();
```
`bevy::asset::embedded_asset!` マクロを使用することで、外部ファイルに依存せず実行ファイル単体でフォントを表示できるようになります。

続いて、`src/home/mod.rs` を作成します。同時に、`main.rs` でこのモジュールを扱えるよう `mod home;` を追記してください。

```rust
// src/home/mod.rs
use crate::state;
use bevy::prelude::*;

pub struct HomePlugin;

impl Plugin for HomePlugin {
    fn build(&self, app: &mut App) {
        // ここにシステムを登録していきます
    }
}
```
`HomePlugin`を`main.rs`の`app`に追加します。

```rust
app.add_plugins(home::HomePlugin);
```

では、Home画面のUI(Startボタン)を実装していきます。

まずは、UI要素を識別するための `StartButton` コンポーネントと、ボタンの見た目を定義する関数を作成します。

```rust
#[derive(Component)]
pub struct StartButton;
```

UIの構成（バンドル）を定義する関数を作成します。
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
このUIレイアウトを、Home画面に遷移したタイミングで生成（スポーン）するように設定します。
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

これでStartボタンが表示されるようになりましたが、まだクリックしても反応がありません。そこで、ボタンが押されたことを検知して `Playing` 状態へ遷移するシステムを実装します。
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