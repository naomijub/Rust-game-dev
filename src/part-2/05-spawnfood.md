# Gerador de Comidas

Nosso próximo passo é começarmos um sistema que gere comidas de forma aleatória pela grade. O primeiro passo é definir qual sera a cor da comida. Como pretendemos fazer um jogo multiplayer, não faz sentido termos comidas coloridas, já que estas serão dos jogadores, sendo assim podemos criar um módulo chamado `food` e adicionar a constante `const FOOD_COLOR: Color = Color::rgb(1.0, 1.0, 1.0)`. Próximo passo é criamos um componente chamado `Food` para representar a comida:

```rs
// food.rs
#[derive(Component)]
pub struct Food;
```

Próximo passo é criarmos um sistema que gera uma comida em um local aleatório da grade. Como este sistema utiliza aleatoriedade, podemos utilizar uma biblioteca de `property testing` semelhante a *proptest* do python, a *propcheck* do Elixir e a *quickcheck* do Haskell, chamada `proptest` para gerar centenas de cenários de teste. Para isso, adicionamos `proptest = "1.0.0"` como uma `dev-dependencies` no Cargo.toml e para utilizarmos basta utilizar a macro `proptest!` e determinar os valores a serem executados (ou quantidade de cenários) como argumento da função de teste como em `_execution in 0u32..1000`:

```rs
#[cfg(test)]
mod test {
    use crate::components::Position;

    use super::*;
    use proptest::prelude::*;

    proptest!{
        #[test]
        fn spawns_food_inplace(_execution in 0u32..1000) {
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

                assert!(x >= 0 && x as i32 <= (GRID_WIDTH -1) as i32);
                assert!(y >= 0 && y as i32 <= (GRID_HEIGHT -1) as i32);
            })
        }
    }
}
```

A vantagem de um proptest é que ele permite executar diversos cenários e podemos definir regras de limite para falha, executando centenas de cenários em poucos segundos. Para este teste passar, precisamos implementar a função `spawn_system` para o módulo `food`:

```rs
// food.rs
pub fn spawn_system(mut commands: Commands) {
    commands
        .spawn_bundle(SpriteBundle {
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
```

O próximo passo é adicionar o sistema a `App` na função `main`, porém este sistema tem uma pegadinha. Como não queremos que o sistema gere uma nova comdia para cada frame, precisamos definir um tempo de intervalo para as comidas serem geradas. Como este cenário de executar uma função somente a cada x segundos é muito comum no desenvolvimento de jogos a Bevy nos disponibiliza a struct `FixedTimestep` que nos permite definir um passo (`step`) em segundos, que será usada com a função `with_run_criteria`:

```rs
// main.rs
pub mod food;

fn main() {
    App::new()
        .insert_resource(WindowDescriptor {
            title: "Snake Game".to_string(),
            width: 500.0,
            height: 500.0,
            ..default()
        }) // <--
        .add_startup_system(setup_camera)
        .add_startup_system(snake::spawn_system)
        .add_plugins(DefaultPlugins)
        .add_system(snake::movement_system)
        .add_system_set(
            SystemSet::new()
                .with_run_criteria(FixedTimestep::step(1.0)) // <-- Pegadinha
                .with_system(food::spawn_system), // <-- Sistema
        ) // <--
        .add_system_set_to_stage(
            CoreStage::PostUpdate,
            SystemSet::new()
                .with_system(grid::position_translation)
                .with_system(grid::size_scaling),
        )
        .run();
}
```

Próximo passo será melhorar o movimento da cabeça da cobra, tornando ele mais lento e cadenciado.