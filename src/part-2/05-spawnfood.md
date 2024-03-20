# Gerador de Comidas

Nosso próximo passo é começarmos um sistema que gere comidas de forma aleatória pela grade. O primeiro passo é definir qual sera a cor da comida. Como pretendemos fazer um jogo multiplayer, não faz sentido termos comidas coloridas, já que estas serão dos jogadores, sendo assim podemos criar um módulo chamado `food` e adicionar a constante `const FOOD_COLOR: Color = Color::rgb(1.0, 1.0, 1.0)`. Próximo passo é criamos um componente chamado `Food` para representar a comida:

```rs
// food.rs
#[derive(Component)]
pub struct Food;
```

Próximo passo é criarmos um sistema que gera uma comida em um local aleatório da grade. Como este sistema utiliza aleatoriedade, podemos utilizar uma biblioteca de `property testing` semelhante a *proptest* do python, a *propcheck* do Elixir e a *quickcheck* do Haskell, chamada `proptest` para gerar centenas de cenários de teste. Para isso, adicionamos `proptest = "1.4.0"` como uma `dev-dependencies` no Cargo.toml ou `cargo add proptest@1.4.0 --dev` na linha de comando e para utilizarmos basta utilizar a macro `proptest!` e determinar os valores a serem executados (ou quantidade de cenários) como argumento da função de teste como em `_execution in 0u16..1000`:

```rust
#[cfg(test)]
mod test {
    use crate::components::Position;

    use super::*;
    use proptest::prelude::*;

    proptest!{
        #[test]
        fn spawns_food_inplace(_execution in 0u16..1000) {
            // Setup app
            let mut app = App::new();

            // Add startup system
            app.add_startup_system(spawn_system);

            // Run systems
            app.update();

            let mut query = app.world.query_filtered::<&Position, With<Food>>();
            assert_eq!(query.iter(&app.world).count(), 1);
            query.iter(&app.world).for_each(|position| {
                let x = position.x;
                let y = position.y;

                assert!(x >= 0 && x as i16 <= (GRID_WIDTH -1) as i16);
                assert!(y >= 0 && y as i16 <= (GRID_HEIGHT -1) as i16);
            })
        }
    }
}
```

Para usarmos nosso `GRID_WIDTH` e `GRID_HEIGHT` no nosso `food.rs` vamos precisar marcar nossa constante como uma crate publica pra isso precisamos fazer a modificação abaixo:
```rust
// grid.rs
pub(crate) const GRID_WIDTH: u16 = 10;
pub(crate) const GRID_HEIGHT: u16 = 10;
```

A vantagem de um proptest é que ele permite executar diversos cenários e podemos definir regras de limite para falha, executando centenas de cenários em poucos segundos. Para este teste passar, precisamos implementar a função `spawn_system` para o módulo `food`:

```rust
// food.rs
use bevy::prelude::*;
use rand::random;

use crate::{
    components::{Position, Size},
    grid::{GRID_HEIGHT, GRID_WIDTH},
};

const FOOD_COLOR: Color = Color::rgb(1.0, 1.0, 1.0);

#[derive(Component)]
pub struct Food;

pub fn spawn_system(mut commands: Commands) {
    commands
        .spawn(SpriteBundle {
            sprite: Sprite {
                color: FOOD_COLOR,
                ..default()
            },
            ..default()
        })
        .insert(Food)
        .insert(Position {
            x: (random::<u16>() % GRID_WIDTH) as i16,
            y: (random::<u16>() % GRID_HEIGHT) as i16,
        })
        .insert(Size::square(0.8));
}

#[cfg(test)]
mod test {
    use crate::components::Position;

    use super::*;
    use proptest::prelude::*;

    proptest! {
        #[test]
        fn spawns_food_inplace(_execution in 0u16..1000) {
            // Setup app
            let mut app = App::new();

            // Add startup system
            app.add_systems(Startup, spawn_system);

            // Run systems
            app.update();

            let mut query = app.world.query_filtered::<&Position, With<Food>>();
            assert_eq!(query.iter(&app.world).count(), 1);
            query.iter(&app.world).for_each(|position| {
                let x = position.x;
                let y = position.y;

                assert!(x >= 0 && x <= (GRID_WIDTH -1) as i16);
                assert!(y >= 0 && y <= (GRID_HEIGHT -1) as i16);
            })
        }
    }
}
```

O próximo passo é adicionar o sistema a `App` na função `main`, porém este sistema tem uma pegadinha. Como não queremos que o sistema gere uma nova comida para cada frame, precisamos definir um tempo de intervalo para as comidas serem geradas. Como este cenário de executar uma função somente a cada x segundos é muito comum no desenvolvimento de jogos a Bevy nos disponibiliza a struct `FixedTimestep` que nos permite definir um passo (`step`) em segundos, que será usada com a função `with_run_criteria`:

```rust
use std::time::Duration; // <-- Import

use bevy::{prelude::*, time::common_conditions::on_timer}; // <-- Import de on_timer

pub mod components;
pub mod food;
pub mod grid;
mod snake;

fn main() {
    App::new()
        .add_systems(Startup, setup_camera)
        .add_systems(Startup, snake::spawn_system)
        .insert_resource(ClearColor(Color::rgb(0.04, 0.04, 0.04)))
        .add_plugins(
            DefaultPlugins
                .set(WindowPlugin {
                    primary_window: Some(Window {
                        resolution: (500.0, 500.0).into(),
                        title: "Snake".into(),
                        resizable: false,
                        ..default()
                    }),
                    ..default()
                })
                .build(),
        )
        .add_systems(PostUpdate, (grid::position_translation, grid::size_scaling))
        .add_systems(Update, snake::movement_system)
        .add_systems( // <--
            Update, // <-- Schedule
            food::spawn_system // <-- Sistema
                .run_if(on_timer(Duration::from_secs_f32(1.0))), //<-- Pegadinha
        )
        .add_systems(PostUpdate, (grid::position_translation, grid::size_scaling))
        .run();
}

// código
```

Próximo passo será melhorar o movimento da cabeça da cobra, tornando ele mais lento e cadenciado.
