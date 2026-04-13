# Playing画面全体の流れとUI

ここではPlugin、一時停止ボタン、再開ボタン、スコアテキスト、リザルト画面への遷移を実装します。

まず、`playing/mod.rs`に`PlayingPlugin`を作成します。
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

また、`playing/mod.rs`に`playing/`内にあるモジュールを書き込みます
```rust
pub mod bullet;
pub mod enemy;
pub mod hp;
pub mod player;
pub mod utils;
```
また、`GameState`も使うので`use`に入れておきます
```rust
use crate::state;
```
ここで、`crate`はルートを指す、つまり`main.rs`にあるものを指し、`main.rs`に展開されている`state`を使うということです。

ではUIを作っていきます。
`playing/mod.rs`内に`UI`,`PauseButton`,`ResumeButton`,`TimeUI`,`KillUI`をコンポーネントとして作ります。

`InGameState`というステートも作ることでゲームが動いているか止まっているかを管理します。

`setup_ui`関数でUI全体がどういう構成かを書いてバンドルとして返します。(ここでhp::HpUIというまだ作ってないコンポーネントが出てくるので`playing/hp.rs`で定義だけしといてください。)
```rust
#[derive(Component)]
struct UI;

#[derive(Component)]
struct PauseButton;

#[derive(Component)]
struct ResumeButton;

#[derive(Component)]
struct TimeUI;

#[derive(Component)]
struct KillUI;

#[derive(Clone, Copy, PartialEq, Eq, Hash, Debug, Default, States)]
pub enum InGameState {
    #[default]
    Running,
    Paused,
}

fn setup_ui(asset_server: &AssetServer) -> impl Bundle {
    (
        // 親となるUI全体
        UI,
        DespawnOnExit(state::GameState::Playing),
        Node {
            width: percent(100),
            height: percent(100),
            align_items: AlignItems::FlexEnd,
            justify_content: JustifyContent::FlexStart,
            flex_direction: FlexDirection::Column,
            row_gap: px(10.0),
            ..default()
        },
        children![
            (   
                // ResumeButton
                Button,
                ResumeButton,
                Node {
                    width: percent(20),
                    height: percent(10),
                    justify_content: JustifyContent::Center,
                    align_items: AlignItems::Center,
                    border_radius: BorderRadius::MAX,
                    ..default()
                },
                BorderColor::all(Color::WHITE),
                BackgroundColor(Color::WHITE),
                children![(
                    Text::new("Resume"),
                    TextFont {
                        font: asset_server.load(
                            "embedded://invader_tutorial/fonts/NotoSansJP-Bold.ttf"
                        ),
                        font_size: 40.0,
                        ..default()
                    },
                    TextLayout::new_with_justify(Justify::Center),
                    TextColor::BLACK,
                )]
            ),
            (
                // PauseButton
                Button,
                PauseButton,
                Node {
                    width: percent(20),
                    height: percent(10),
                    justify_content: JustifyContent::Center,
                    align_items: AlignItems::Center,
                    border_radius: BorderRadius::MAX,
                    ..default()
                },
                BorderColor::all(Color::WHITE),
                BackgroundColor(Color::WHITE),
                children![(
                    Text::new("Pause"),
                    TextFont {
                        font: asset_server.load(
                            "embedded://invader_tutorial/fonts/NotoSansJP-Bold.ttf"
                        ),
                        font_size: 40.0,
                        ..default()
                    },
                    TextLayout::new_with_justify(Justify::Center),
                    TextColor::BLACK,
                )]
            ),
            (
                // TimeUI
                Text::new(""),
                TimeUI,
                TextFont {
                    font: asset_server
                        .load("embedded://invader_tutorial/fonts/NotoSansJP-Bold.ttf"),
                    font_size: 40.0,
                    ..default()
                },
                TextLayout::new_with_justify(Justify::Center),
                TextColor::WHITE,
            ),
            (
                // KillUI
                Text::new(""),
                KillUI,
                TextFont {
                    font: asset_server
                        .load("embedded://invader_tutorial/fonts/NotoSansJP-Bold.ttf"),
                    font_size: 40.0,
                    ..default()
                },
                TextLayout::new_with_justify(Justify::Center),
                TextColor::WHITE,
            ),
            (   
                // HpUI
                Text::new(""),
                hp::HpUI,
                TextFont {
                    font: asset_server
                        .load("embedded://invader_tutorial/fonts/NotoSansJP-Bold.ttf"),
                    font_size: 40.0,
                    ..default()
                },
                TextLayout::new_with_justify(Justify::Center),
                TextColor::WHITE,
            ),
        ],
    )
}
```

では次に現在のスコアと過去のスコアのリストのリソースを定義します(`playing/mod.rs`)。リソースとはアプリ全体に一つだけ存在し、どのシステムからでもアクセスできるシングルトン的なデータストレージです。

```rust
// スコアの構造を定義
// キル数と生存時間から算出する
#[derive(Default, Clone, Copy)]
pub struct Score {
    pub kill: i32,
    pub survival_time: f32,
}
// スコアの計算
impl Score {
    pub fn score(&self) -> f32 {
        (self.kill as f32 * self.survival_time.min(100.0)).sqrt()
    }
}
// スコアを表示するために`score.to_string()`でできる文字列を作る
impl std::fmt::Display for Score {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        f.write_str(&format!(
            "Kill: {}\nSurvival Time: {:.2}\nScore: {:.2}",
            self.kill,
            self.survival_time,
            self.score()
        ))
    }
}

// 現在のスコア
#[derive(Resource)]
pub struct CurrentScore(Score);
// スコアの初期化
fn reset_current_score(current_score: &mut CurrentScore) {
    current_score.0 = Score::default();
}
// 過去のスコアのリスト
#[derive(Resource)]
pub struct ScoreList(pub Vec<Score>);
// 現在のスコアをリストに追加する
fn push_score_list(mut score_list: ResMut<ScoreList>, current_score: Res<CurrentScore>) {
    score_list.0.push(current_score.0);
}
```

時間を管理するためのストップウォッチのリソースを定義します(`playing/mod.rs`)。

```rust
#[derive(Resource, Default)]
pub struct StopWatch {
    time: f32,
    is_running: bool,
}

impl StopWatch {
    pub fn new(run: bool) -> Self {
        Self {
            time: 0.0,
            is_running: run,
        }
    }
    pub fn now(&self) -> f32 {
        self.time
    }
    pub fn start(&mut self) {
        self.is_running = true;
    }
    pub fn pause(&mut self) {
        self.is_running = false;
    }
    pub fn reset(&mut self) {
        self.time = 0.0;
    }
    pub fn is_running(&self) -> bool {
        self.is_running
    }
}

fn update_stopwatch(time: Res<Time>, mut stopwatch: ResMut<StopWatch>) {
    if stopwatch.is_running() {
        stopwatch.time += time.delta_secs();
    }
}

fn start_stopwatch_res(mut stopwatch: ResMut<StopWatch>) {
    stopwatch.reset();
    stopwatch.start();
}
```

`TimeUI`や`KillUI`を更新するシステムと一時停止ボタンと再開ボタンを動作させるシステムを作ります(`playing/mod.rs`)。

```rust
fn update_time_ui(stopwatch: Res<StopWatch>, mut time_ui_query: Query<&mut Text, With<TimeUI>>) {
    for mut time_ui in &mut time_ui_query {
        let current_time = stopwatch.now();
        **time_ui = format!("Time: {:.2}s / 100s", current_time);
    }
}

fn update_kill_ui(
    current_score: ResMut<CurrentScore>,
    mut kill_ui_query: Query<&mut Text, With<KillUI>>,
) {
    for mut kill_ui in &mut kill_ui_query {
        **kill_ui = format!("Kill: {}", current_score.0.kill);
    }
}

type ResumeButtonInputs = (Changed<Interaction>, With<ResumeButton>);
fn update_resume_button(
    mut query: Query<(&Interaction, &mut BackgroundColor), ResumeButtonInputs>,
    mut stopwatch: ResMut<StopWatch>,
    mut game_state: ResMut<NextState<InGameState>>,
) {
    for (interaction, mut background_color) in query.iter_mut() {
        match interaction {
            Interaction::Pressed => {
                background_color.0 = Color::srgb(0.5, 0.5, 0.5);
                stopwatch.start();
                game_state.set(InGameState::Running);
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

type PauseButtonInputs = (Changed<Interaction>, With<PauseButton>);
fn update_pause_button(
    mut query: Query<(&Interaction, &mut BackgroundColor), PauseButtonInputs>,
    mut stopwatch: ResMut<StopWatch>,
    mut game_state: ResMut<NextState<InGameState>>,
) {
    for (interaction, mut background_color) in query.iter_mut() {
        match interaction {
            Interaction::Pressed => {
                background_color.0 = Color::srgb(0.5, 0.5, 0.5);
                stopwatch.pause();
                game_state.set(InGameState::Paused);
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
また、時間制限を超えたらリザルト画面に行くシステムも作ります。
```rust
const TIME_LIMIT: f32 = 100.0;
fn check_time_limit(
    stopwatch: Res<StopWatch>,
    mut current_score: ResMut<CurrentScore>,
    mut game_state: ResMut<NextState<crate::state::GameState>>,
) {
    let current_time = stopwatch.now();
    current_score.0.survival_time = current_time;
    if current_time >= TIME_LIMIT {
        game_state.set(state::GameState::Result);
    }
}
```
次は背景を用意します。適当な画像を用意して`src/img/invader_background.png`にでもおいておきます。`main.rs`につぎのを追加します。
```rust
bevy::asset::embedded_asset!(app, "img/invader_background.png");
```

Playing画面の初期配置をするシステムを作ります。
```rust
fn setup_playing(
    mut commands: Commands,
    asset_server: Res<AssetServer>,
    mut current_score: ResMut<CurrentScore>,
    mut meshes: ResMut<Assets<Mesh>>,
    mut materials: ResMut<Assets<StandardMaterial>>,
) {
    reset_current_score(&mut current_score);
    // spawn a camera
    commands.spawn((
        Camera3d::default(),
        Transform::from_xyz(0.0, 30.0, 0.0).looking_at(Vec3::ZERO, Vec3::Z),
        DespawnOnExit(state::GameState::Playing),
    ));
    // spawn ui
    commands.spawn(setup_ui(&asset_server));
    // spawn background
    commands.spawn((
        Mesh3d(meshes.add(Plane3d::default().mesh().size(50.0, 50.0))),
        MeshMaterial3d(materials.add(
            StandardMaterial {
                base_color_texture:
                    Some(
                        asset_server.load(
                            "embedded://invader_tutorial/img/invader_background.png",
                        ),
                    ),
                unlit: true,
                ..default()
            },
        )),
        Transform::from_xyz(0.0, 0.0, -10.0),
        DespawnOnExit(state::GameState::Playing),
    ));
}
```
ではステート、リソース、システムを`app`に追加していきます。
```rust
impl Plugin for PlayingPlugin {
    fn build(&self, app: &mut App) {
        app.insert_resource(StopWatch::new(false))
            .insert_resource(CurrentScore(Score::default()))
            .insert_resource(ScoreList(Vec::new()))
            .init_state::<InGameState>()
            .add_systems(
                OnEnter(state::GameState::Playing),
                (setup_playing, start_stopwatch_res),
            )
            .add_systems(
                Update,
                (
                    update_stopwatch,
                    update_time_ui,
                    update_kill_ui,
                    check_time_limit,
                    update_pause_button,
                    update_resume_button,
                )
                    .run_if(in_state(state::GameState::Playing)),
            )
            .add_systems(OnExit(state::GameState::Playing), push_score_list);
    }
}

```
