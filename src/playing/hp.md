# HP

HpUIの更新とHPが0になったときのシステムを書きます。

HpUIの更新とプレイヤーのHPが0になったときの処理をするシステムを作る。
```rust
use crate::playing::CurrentScore;

use super::utils::*;
use bevy::prelude::*;

#[derive(Component)]
pub struct HpUI;

pub fn update_player_hp(
    mut commands: Commands,
    mut ui_query: Query<&mut Text, With<HpUI>>,
    player: Query<(Entity, &HP), With<super::player::Player>>,
    mut game_state: ResMut<NextState<crate::state::GameState>>,
) {
    let mut ui = match ui_query.single_mut() {
        Ok(ui) => ui,
        Err(_) => {
            warn!("Expected exactly one HP UI entity, but found none or multiple.");
            return;
        }
    };
    let (player_entity, hp) = match player.single() {
        Ok(p) => p,
        Err(_) => {
            warn!("Expected exactly one Player entity, but found none or multiple.");
            return;
        }
    };
    **ui = format!("HP: {}", hp.0);
    if hp.0 <= 0.0 {
        commands.entity(player_entity).despawn();
        game_state.set(crate::state::GameState::Result);
    }
}
```

敵のHPが0になったときに敵をデスポーンしKillをカウントアップするシステムを作る。
```rust
pub fn handle_enemy_death(
    mut commands: Commands,
    query: Query<(Entity, &HP, &Character)>,
    mut materials: ResMut<Assets<StandardMaterial>>,
    mut meshes: ResMut<Assets<Mesh>>,
    mut current_score: ResMut<CurrentScore>,
) {
    for (entity, hp, &character) in query {
        if character == Character::Player {
            continue;
        }
        if hp.0 <= 0.0 {
            commands.entity(entity).despawn();
            if character == Character::Enemy {
                current_score.0.kill += 1;
                super::enemy::spawn_random_enemy(&mut commands, &mut meshes, &mut materials);
            }
        }
    }
}
```

これらのシステムを`HpPlugin`に追加する。
```rust
use crate::state;

pub struct HpPlugin;

impl Plugin for HpPlugin {
    fn build(&self, app: &mut App) {
        app.add_systems(
            Update,
            (update_player_hp, handle_enemy_death).chain().run_if(
                in_state(state::GameState::Playing).and(in_state(super::InGameState::Running)),
            ),
        );
    }
}
```
また、このプラグインも`playing/mod.rs`の方に追加するのを忘れないように。
