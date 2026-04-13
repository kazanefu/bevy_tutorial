# プレイヤー

プレイヤーが操作するものを作っていきます。

まず、`playing/utils.rs`に`Character`、`HP`と`Control`というコンポーネントを定義します。`Control`で速度を計算&保持して、それに基づいて`Transform`を更新します。できれば既存のライブラリを使いたかったのですがこれを書いてた当時はBevyのバージョンが上がったばかりでrapierという物理エンジンのライブラリが追いついていなかったので簡単なものを自分で作ることにしました。

```rust
use bevy::{math::NormedVectorSpace, prelude::*};

pub struct UtilPlugin;

impl Plugin for UtilPlugin {
    fn build(&self, app: &mut App) {
        app.add_systems(
            Update,
            (update_velocity).run_if(
                in_state(crate::state::GameState::Playing)
                    .and(in_state(super::InGameState::Running)),
            ),
        );
    }
}

#[derive(Component, PartialEq, Eq, Clone, Copy)]
pub enum Character {
    Player,
    Enemy,
}

#[derive(Component)]
pub struct HP(pub f32);

#[derive(Component, Default)]
pub struct Control {
    pub mass: f32,
    pub force: Vec3,
    pub acceleration: Vec3,
    pub velocity: Vec3,
    pub speed_limit: f32,
}

impl Control {
    pub fn add_force(&mut self, force: Vec3) {
        self.force = force;
    }
    pub fn speed(&self) -> f32 {
        self.velocity.norm()
    }
    pub fn calculate_velocity(&mut self, delta_time: f32) {
        self.acceleration = self.force / self.mass;
        self.velocity += self.acceleration * delta_time;
        if self.speed() >= self.speed_limit {
            self.velocity = self.velocity.normalize() * self.speed_limit;
        }
        self.force = Vec3::ZERO;
    }
}

fn update_velocity(query: Query<(&mut Control, &mut Transform)>, time: Res<Time>) {
    for (mut control, mut transform) in query {
        control.calculate_velocity(time.delta_secs());
        transform.translation += control.velocity * time.delta_secs();
        transform.translation.x = transform.translation.x.clamp(-20.0, 20.0); // x limit
    }
}
```
`UtilPlugin`を`playing/mod.rs`のappに追加します。

`playing/player.rs`を編集していきます。まずは定数を定義していきます。
```rust
const PLAYER_FORCE: f32 = 3.0;
const PLAYER_SPEED_LIMIT: f32 = 10.0;
const PLAYER_MAX_HP: f32 = 100.0;
const PLAYER_START_X: f32 = 0.0;
const PLAYER_START_Y: f32 = 0.0;
const PLAYER_START_Z: f32 = -8.0;
```
次に`Player`コンポーネントとプレイヤーをスポーンするシステムを作ります。
```rust
use bevy::prelude::*;

// superは一つ上の階層を指す。つまりここでは`playing/`。(正確には`playing/.mod.rs`)
use super::bullet;
use super::utils::*;

#[derive(Component)]
pub struct Player;

fn spawn_player(
    mut commands: Commands,
    mut meshes: ResMut<Assets<Mesh>>,
    mut materials: ResMut<Assets<StandardMaterial>>,
) {
    commands.spawn((
        DespawnOnExit(crate::state::GameState::Playing),
        Character::Player,
        Player,
        HP(PLAYER_MAX_HP),
        Transform::from_xyz(PLAYER_START_X, PLAYER_START_Y, PLAYER_START_Z).looking_at(Vec3::ZERO, Vec3::Y),
        Mesh3d(meshes.add(Cuboid::new(1.0, 1.0, 1.0))),
        MeshMaterial3d(materials.add(StandardMaterial {
            base_color: Color::BLACK,
            emissive: Color::srgb(0.1, 0.1, 1.0).to_linear(),
            ..default()
        })),
        Control {
            speed_limit: PLAYER_SPEED_LIMIT,
            mass: 1.0,
            ..default()
        },
    ));
}
```
次に入力を受け取ってPlayerを動かすシステムを作ります。
```rust
fn move_player(mut query: Query<&mut Control, With<Player>>, keyboard: Res<ButtonInput<KeyCode>>) {
    let mut control = match query.single_mut() {
        Ok(control) => control,
        Err(_) => {
            warn!("Expected exactly one Player entity, but found none or multiple.");
            return;
        }
    };
    if keyboard.pressed(KeyCode::ArrowLeft) || keyboard.pressed(KeyCode::KeyA) {
        control.add_force(Vec3 {
            x: PLAYER_FORCE,
            y: 0.0,
            z: 0.0,
        });
    }
    if keyboard.pressed(KeyCode::ArrowRight) || keyboard.pressed(KeyCode::KeyD) {
        control.add_force(Vec3 {
            x: -PLAYER_FORCE,
            y: 0.0,
            z: 0.0,
        });
    }
}
```

これらのシステムを追加したプラグインを作ります。
```rust
pub struct PlayerPlugin;

impl Plugin for PlayerPlugin {
    fn build(&self, app: &mut App) {
        app.add_systems(OnEnter(crate::state::GameState::Playing), spawn_player)
            .add_systems(
                Update,
                (move_player).run_if(
                    in_state(crate::state::GameState::Playing)
                        .and(in_state(super::InGameState::Running)),
                ),
            );
    }
}
```
`playing/mod.rs`の方にこのプラグインを追加するのを忘れないでください。

これでプレイヤーを動かせるようになりました。