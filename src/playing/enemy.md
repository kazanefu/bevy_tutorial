# 敵キャラクターの実装

`playing/enemy.rs` を作成し、`Enemy` コンポーネントと関連する定数を定義します。
```rust
#[derive(Component)]
pub struct Enemy;

const SHOOT_RATE: i32 = 3;
const SHOOT_CHANCE_MAX: i32 = 10;
const SHOOT_INTERVAL: f32 = 0.3;
const ENEMY_MAX_HP: f32 = 100.0;

const ENEMY_SPAWN_X_RANGE: std::ops::RangeInclusive<i32> = -9..=9;
const ENEMY_SPAWN_Y: f32 = 0.0;
const ENEMY_SPAWN_Z: f32 = 10.0;

const ENEMY_SPEED_LIMIT: f32 = 2.0;
```
敵を生成するための `spawn_enemy` 関数を定義します。
```rust
fn spawn_enemy(
    commands: &mut Commands,
    meshes: &mut Assets<Mesh>,
    materials: &mut Assets<StandardMaterial>,
    translation: Vec3,
) {
    let mut rng = rand::rng();
    commands.spawn((
        DespawnOnExit(crate::state::GameState::Playing),
        Character::Enemy,
        Enemy,
        super::utils::Interval {
            time: 0.0,
            interval: SHOOT_INTERVAL,
        },
        HP(ENEMY_MAX_HP),
        Transform::from_translation(translation).looking_to(-Vec3::Z, Vec3::Y),
        Mesh3d(meshes.add(Cuboid::new(1.0, 1.0, 1.0))),
        MeshMaterial3d(materials.add(StandardMaterial {
            base_color: Color::BLACK,
            emissive: Color::srgb(1.0, 0.1, 0.1).to_linear(),
            ..default()
        })),
        Control {
            speed_limit: ENEMY_SPEED_LIMIT,
            mass: 1.0,
            velocity: Vec3 {
                x: rng.random_range(-1.0..1.0),
                y: 0.0,
                z: rng.random_range(-1.0..0.0),
            },
            ..default()
        },
    ));
}
```
また、ランダムな位置へのスポーン関数と、ゲーム開始時の初期配置を行うシステムを定義します。
ランダムスポーンは、敵が撃破された際や画面外へ消えた際のリスポーンに使用します。

乱数生成を行うために、`rand` クレートを追加しましょう。`Cargo.toml` の `[dependencies]` に以下を追記します。

```toml
rand = "0.10.1"
```

```rust
use rand::prelude::*;
pub fn spawn_random_enemy(
    commands: &mut Commands,
    meshes: &mut Assets<Mesh>,
    materials: &mut Assets<StandardMaterial>,
) {
    let mut rng = rand::rng();
    let x = rng.random_range(ENEMY_SPAWN_X_RANGE);
    spawn_enemy(
        commands,
        meshes,
        materials,
        Vec3 {
            x: x as f32,
            y: ENEMY_SPAWN_Y,
            z: ENEMY_SPAWN_Z,
        },
    );
}

fn setup_enemies(
    mut commands: Commands,
    mut materials: ResMut<Assets<StandardMaterial>>,
    mut meshes: ResMut<Assets<Mesh>>,
) {
    for i in -3..=3 {
        spawn_enemy(
            &mut commands,
            &mut meshes,
            &mut materials,
            Vec3 {
                x: (i * 3) as f32,
                y: ENEMY_SPAWN_Y,
                z: ENEMY_SPAWN_Z,
            },
        );
    }
}
```

画面外（移動範囲外）に出た敵を削除し、代わりに新しい敵をランダムな位置にスポーンさせます。
```rust
const ENEMY_X_LIMIT: f32 = 19.0;
const ENEMY_NEG_X_LIMIT: f32 = -19.0;
const ENEMY_Z_LIMIT: f32 = 15.0;
const ENEMY_NEG_Z_LIMIT: f32 = -6.0;

fn delete_out_of_range_enemy(
    query: Query<(&Transform, Entity), With<Enemy>>,
    mut commands: Commands,
    mut materials: ResMut<Assets<StandardMaterial>>,
    mut meshes: ResMut<Assets<Mesh>>,
) {
    for (translation, entity) in query
        .iter()
        .map(|(transform, entity)| (transform.translation, entity))
    {
        let x = translation.x;
        let z = translation.z;
        if !(ENEMY_NEG_X_LIMIT..=ENEMY_X_LIMIT).contains(&x)
            || !(ENEMY_NEG_Z_LIMIT..=ENEMY_Z_LIMIT).contains(&z)
        {
            commands.entity(entity).despawn();
            spawn_random_enemy(&mut commands, &mut meshes, &mut materials);
        }
    }
}
```

敵が一定間隔で一定確率で弾丸を発射するシステムを作成します。
```rust
fn enemy_shoot(
    mut commands: Commands,
    query: Query<(&Transform, &Character, &mut super::utils::Interval), With<Enemy>>,
    mut meshes: ResMut<Assets<Mesh>>,
    mut materials: ResMut<Assets<StandardMaterial>>,
) {
    for (transform, &owner, mut interval) in query {
        let mut rng = rand::rng();
        let random_val = rng.random_range(0..SHOOT_CHANCE_MAX);
        let is_ready = interval.is_ready();
        if is_ready {
            interval.reset();
        }
        if is_ready && random_val <= SHOOT_RATE {
            super::bullet::spawn_bullet(
                &mut commands,
                owner,
                transform.translation,
                transform.forward(),
                &mut meshes,
                &mut materials,
            );
        }
    }
}
```

作成したシステムを `EnemyPlugin` にまとめ、アプリケーションに登録します。
```rust
use super::utils::*;
use bevy::prelude::*;
use rand::prelude::*;
pub struct EnemyPlugin;

impl Plugin for EnemyPlugin {
    fn build(&self, app: &mut App) {
        app.add_systems(OnEnter(crate::state::GameState::Playing), setup_enemies)
            .add_systems(
                Update,
                (enemy_shoot, delete_out_of_range_enemy).run_if(
                    in_state(crate::state::GameState::Playing)
                        .and(in_state(super::InGameState::Running)),
                ),
            );
    }
}
```