# 敵

`Enemy`コンポーネントと定数を作ります。`playing/enemy.rs`
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
敵をスポーンする関数を作ります。
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
また、ランダムな位置にスポーンする関数と初期配置をするシステムを作ります。ランダムな位置へのスポーンは敵が倒されたり範囲外に出てリスポーンするときに使います。乱数を使いたいので`rand`というライブラリクレートを使います。
```toml
rand = "0.10.1"
```
をCargo.tomlに追加

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

範囲外に出た敵をデスポーンして新たな敵をスポーンします。
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

また、敵にも弾丸を発射するシステムを作ります。
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

これらのシステムをこれまで同様にプラグインに追加します。
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