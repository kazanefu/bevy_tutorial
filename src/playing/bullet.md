# 弾丸の実装

`playing/bullet.rs` を作成し、`Bullet` コンポーネントと関連する定数を定義します。
```rust
use bevy::prelude::*;

use super::utils::*;
use crate::state;

#[derive(Component)]
pub struct Bullet {
    owner: Character,
    velocity: Vec3,
    damage: f32,
}

const BULLET_SPEED: f32 = 30.0;
const PLAYER_BULLET_DAMAGE: f32 = 30.0;
const ENEMY_BULLET_DAMAGE: f32 = 3.0;
const BULLET_COLLISION_RADIUS: f32 = 1.0;
const BULLET_LIFE_TIME: f32 = 0.7;
```
弾丸を移動させるためのシステムを作成します。
```rust
fn move_bullet(query: Query<(&mut Transform, &Bullet)>, time: Res<Time>) {
    for (mut transform, bullet) in query {
        transform.translation += bullet.velocity * time.delta_secs();
    }
}
```
弾丸は発射から一定時間後に消滅させる必要があるため、時間を計測するためのコンポーネントを `playing/utils.rs` に追加します。
```rust
#[derive(Component)]
pub struct Interval {
    pub time: f32,
    pub interval: f32,
}

impl Interval {
    pub fn tick(&mut self, delta_time: f32) {
        self.time += delta_time;
    }
    pub fn reset(&mut self) {
        self.time = 0.0;
    }
    pub fn is_ready(&self) -> bool {
        self.time >= self.interval
    }
}

fn tick_interval(time: Res<Time>, query: Query<&mut Interval>) {
    for mut interval in query {
        interval.tick(time.delta_secs());
    }
}
```
作成した `tick_interval` システムを、`UtilPlugin` の `build` メソッド内で App に追加しておきます。
```rust
pub struct UtilPlugin;

impl Plugin for UtilPlugin {
    fn build(&self, app: &mut App) {
        app.add_systems(
            Update,
            (tick_interval, update_velocity).run_if(
                in_state(crate::state::GameState::Playing)
                    .and(in_state(super::InGameState::Running)),
            ),
        );
    }
}
```


続いて、`playing/bullet.rs` に弾丸を生成（スポーン）するための関数を定義します。
```rust
pub fn spawn_bullet(
    commands: &mut Commands,
    owner: Character,
    translation: Vec3,
    forward: Dir3,
    meshes: &mut Assets<Mesh>,
    materials: &mut Assets<StandardMaterial>,
) {
    let (color, damage) = match owner {
        Character::Player => (Color::srgb(0.0, 1.0, 1.0), PLAYER_BULLET_DAMAGE),
        Character::Enemy => (Color::srgb(1.0, 0.0, 1.0), ENEMY_BULLET_DAMAGE),
    };
    commands.spawn((
        Bullet {
            owner,
            velocity: forward.normalize() * BULLET_SPEED,
            damage,
        },
        super::utils::Interval {
            time: 0.0,
            interval: BULLET_LIFE_TIME,
        },
        DespawnOnExit(state::GameState::Playing),
        Transform::from_translation(translation).looking_to(forward, Vec3::Y),
        Mesh3d(meshes.add(Cuboid::new(0.2, 0.2, 1.0))),
        MeshMaterial3d(materials.add(StandardMaterial {
            base_color: Color::BLACK,
            emissive: color.to_linear(),
            ..default()
        })),
    ));
}
```

弾丸とキャラクターの衝突判定を行うシステムを作成します。
```rust
fn bullet_collision(
    mut commands: Commands,
    bullet_query: Query<(Entity, &Transform, &Bullet)>,
    mut character_query: Query<(&Transform, &Character, &mut HP)>,
) {
    for (bullet_entity, bullet_transform, bullet) in bullet_query {
        for (character_transform, character, mut hp) in character_query.iter_mut() {
            if *character == bullet.owner {
                continue;
            }
            let distance_sq = bullet_transform
                .translation
                .distance_squared(character_transform.translation);
            if distance_sq <= BULLET_COLLISION_RADIUS * BULLET_COLLISION_RADIUS {
                hp.0 -= bullet.damage;
                commands.entity(bullet_entity).despawn();
                break ;
            }
        }
    }
}
```

生存期間（Interval）が終了した弾丸を自動的に削除するシステムを作成します。

```rust
fn remove_time_out_bullet(
    mut commands: Commands,
    bullet_query: Query<(Entity, &super::utils::Interval), With<Bullet>>,
) {
    for (entity, interval) in bullet_query {
        if interval.is_ready() {
            commands.entity(entity).despawn();
        }
    }
}
```

`BulletPlugin` を作成し、これまで定義したシステムを登録します。
```rust
pub struct BulletPlugin;

impl Plugin for BulletPlugin {
    fn build(&self, app: &mut App) {
        app.add_systems(
            Update,
            (bullet_collision, move_bullet, remove_time_out_bullet).run_if(
                in_state(crate::state::GameState::Playing)
                    .and(in_state(super::InGameState::Running)),
            ),
        );
    }
}
```
作成した `BulletPlugin` を `playing/mod.rs` に追加するのを忘れないようにしてください。

最後に、プレイヤーがスペースキーを押した時に弾丸を発射できるよう、`playing/player.rs` に発射システムを追加します。
```rust
fn shoot(
    mut commands: Commands,
    query: Query<(&Transform, &Character), With<Player>>,
    keyboard: Res<ButtonInput<KeyCode>>,
    mut meshes: ResMut<Assets<Mesh>>,
    mut materials: ResMut<Assets<StandardMaterial>>,
) {
    for (transform, &owner) in query {
        if keyboard.just_pressed(KeyCode::Space) {
            bullet::spawn_bullet(
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
作成した `shoot` システムを `PlayerPlugin` にも登録しましょう。
```rust
pub struct PlayerPlugin;

impl Plugin for PlayerPlugin {
    fn build(&self, app: &mut App) {
        app.add_systems(OnEnter(crate::state::GameState::Playing), spawn_player)
            .add_systems(
                Update,
                (move_player, shoot).run_if(
                    in_state(crate::state::GameState::Playing)
                        .and(in_state(super::InGameState::Running)),
                ),
            );
    }
}
```

これで弾丸の発射までできるようになりました。
