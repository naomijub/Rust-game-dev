# Migrando versões da Bevy

A equipe da Bevy fez um trabalho sensacional auxiliando equipes que usam a engine a manter seus códigos atulizados com as novas versões e guias futuros podem ser encontrados em https://bevyengine.org/learn/book/migration-guides/. Neste momento a última versão é a 0.9, portanto migraremos para versão 0.8 e depois para 0.9.

## Migrando para versão 0.8

Para iniciarmos a migração basta mudarmos a versão da `bevy` no Cargo.toml:

```toml
[dependencies]
bevy = { version = "0.8", features = ["dynamic"] }
rand = "0.7"
```

Depois ao executar um `cargo check` veremos que já no arquivo `main.rs` há dois erros:

1. `OrthographicCameraBundle` é uma struct não declarada. Basta substituir `OrthographicCameraBundle::new_2d()` por `Camera2dBundle::default()`.
2. `FixedTimestep` não foi encontrado no módulo `core`. Isso acontece pois `FixedTimestep` e todas as coisas relacionadas a tempo foram movidas para o módulo time, assim mude o import para `use bevy::{time::FixedTimestep, prelude::*};`.

Migração pronta!

## Migrando para versão 0.9

A migração para versão 0.9 é um pouco mais trabalhosa, pois algumas inferfaces da API mudaram.

### Spawn

Agora para utilizar a função `Commands.spawn` precisamos enviar uma tupla com quais serão os componentes spawnados, como era feito em  `spawn_bundle`, ou utilizar `spawn_empty` para criar uma entidade vazia. Além disso, `spawn_bundle` passa a ser deprecada. Assim a mudança fica:

```rs
// Antigo
commands.spawn().insert_bundle((A, B, C));
// Novo
commands.spawn((A, B, C));
```

No nosso código, essa mudança reflete nos módulos `game.rs`:

```rs
pub fn game_over_system(mut commands: Commands, mut reader: EventReader<GameEndEvent>) {
    if reader.iter().next().is_some() {
        commands.spawn_empty().insert(GameEndEvent::GameOver);
        println!("{}", GameEndEvent::GameOver);
    }
}
```

E mudamos `spawn_bundle` por `spawn` em `food.rs`, assim como em vários lugares de `snake.rs`:

```rs
pub fn spawn_system(mut commands: Commands, positions: Query<&Position>) {
    let positions_set: HashSet<&Position> = positions.iter().collect();

    if let Some(position) = (0..(GRID_WIDTH * GRID_HEIGHT))
        .map(|_| Position {
            x: if cfg!(test) {
                3
            } else {
                (random::<u16>() % GRID_WIDTH) as i16
            },
            y: if cfg!(test) {
                5
            } else {
                (random::<u16>() % GRID_HEIGHT) as i16
            },
        })
        .find(|position| !positions_set.contains(position))
    {
        commands
            .spawn(SpriteBundle {
                sprite: Sprite {
                    color: FOOD_COLOR,
                    ..default()
                },
                ..default()
            })
            .insert(Food)
            .insert(position)
            .insert(Size::square(0.65));
    }
}
```

Outra possível otimização é:

```rs
commands
    .spawn((SpriteBundle {
        sprite: Sprite {
            color: FOOD_COLOR,
            ..default()
        },
        ..default()
    }, Food, Size::square(0.65)))
    .insert(position);
```

### Resources

Uma mudança importante para resources é que agora todo resource precisa implementar a trait `Resource` através da macro derive, então no  módulo `snake.rs` precisamos adicionar:

```rs
#[derive(Default, Deref, DerefMut, Resource)] // <-- Resource
pub struct Segments(Vec<Entity>);

#[derive(Default, Resource)] // <-- Resource
pub struct LastTailPosition(Option<Position>);
```

Além disso, o jeito de adicionar `WindowDescriptor` ao `App` mudou, pois agora o `WindowDescriptor` é parte do `WindowPlugin`, que deve ser configurado em `DefaultPlugins`:

```rs
fn main() {
    App::new()
        .add_plugins(DefaultPlugins.set(WindowPlugin {
        window: WindowDescriptor {
            title: "Snake Game".to_string(),
            width: 500.0,
            height: 500.0,
            ..default()
        },
        ..default()
      }))
        // REMOVER
        // .insert_resource(WindowDescriptor {
        //     title: "Snake Game".to_string(),
        //     width: 500.0,
        //     height: 500.0,
        //     ..default()
        // })
        // .add_plugins(DefaultPlugins)
        .insert_resource(snake::Segments::default())
        .insert_resource(snake::LastTailPosition::default())
        .add_event::<GrowthEvent>()
        .add_event::<GameEndEvent>()
        .add_startup_system(setup_camera)
        .add_startup_system(snake::spawn_system)
        // ...
}     
```

Last thing on handling `Window` resource, the `Window::new` function now receives an Optional `raw_handle`, então nos testes em `grid.rs` devemos remover `let raw_window_handle = Some(RawWindowHandle::Web(WebHandle::empty()));` e modificar Window para:

```rs
 let window = Window::new(
    WindowId::new(),
    &descriptor, 400, 400, 1., None,
    None, // <--
);
```

## Migrando para versão 0.10

Primeiro passo eh utilizarmos `cargo outdated -R` para identificarmos quais bibliotecas podem ser atualizadas. O resultado eh:

```sh
$ cargo outdated -R
warning: Feature dynamic of package bevy has been obsolete in version 0.10.0
Name               Project  Compat  Latest  Kind         Platform
----               -------  ------  ------  ----         --------
bevy               0.9.1    ---     0.10.0  Normal       ---
proptest           1.0.0    1.1.0   1.1.0   Development  ---
rand               0.7.3    ---     0.8.5   Normal       ---
raw-window-handle  0.4.3    ---     0.5.1   Development  ---
```

Assim, podemos iniciar subindo as versoes das biblitoecas que nao sao a Bevy. Iniciamos por `proptest`, `rand` e `raw-window-handle`, que ao subirmos para as versoes `1.1.0`, `0.8.5` e `remover`, respectivamente. Como `proptest` nao trouxe nenhuma quebra de compatibildiade e estamos utilizando apenas a API mais simples de `rand`, nao observamos nenhum problema de compatibilidade. 

Proximo passo eh seguir os passos do tutorial de [migracao 0.9->0.10](https://bevyengine.org/learn/migration-guides/0.9-0.10/) e atualizarmos a versao da `Bevy` para `0.10`. Ao fazermos essa atualizacao, a primeira grande mudanca eh a feature `dynamic`, que agora se chama `dynamic_linking`:

```toml
[dependencies]
bevy = { version = "0.10", features = ["dynamic_linking"] }
rand = "0.8.5"
``` 

Com o upgrade de versao para a `0.10`, percebemos que os modulos `grid`, `main` e `snake`  contem erros. Comecaremos pelo modulo `grid` que trata dos erros relacionados a `Window`, j'a que agora `Windows` passou a ser uma entidade e sua construcao ficou simplificada. Assim, nos testes `translate_position_to_window` e `transform_has_correct_scale_for_window` podemos simplificar a criacao de ` Window` com (e removendo o `WindowId` do `use`):

```rs
// grid.rs#test
fn transform_has_correct_scale_for_window() {
    // ...
    // Antiga versao
    // let mut descriptor = WindowDescriptor::default();
    // descriptor.height = 200.;
    // descriptor.width = 200.;
    // let window = Window::new(WindowId::new(), &descriptor, 200, 200, 1., None, None);

    let window = Window {
        resolution: WindowResolution::new(200., 200.),
        ..default()
    };    
    // ...
}

fn translate_position_to_window() {
    // ...
    // Antiga versao
    // let mut descriptor = Window::default();
    // descriptor.
    // descriptor.height = 400.;
    // descriptor.width = 400.;
    // let window = Window::new(WindowId::new(), &descriptor, 400, 400, 1., None, None);
    
    let window = Window {
        resolution: WindowResolution::new(400., 400.),
        ..default()
    };
    // ...
}
```

Ja nas funcoes `size_scaling` e `position_translation` a mudanca eh bastante simples, pois `Window` deixou de ser um recurso (`Res`) para ser uma entidade, que devemos manusear com uma query pela window primaria, `primary_window: Query<&Window, With<PrimaryWindow>>`:

```rs
// grid.rs

use crate::components::{Position, Size};
use bevy::{prelude::*, window::PrimaryWindow};

// ...

#[allow(clippy::missing_panics_doc)]
#[allow(clippy::needless_pass_by_value)]
pub fn size_scaling(primary_window: Query<&Window, With<PrimaryWindow>>, mut q: Query<(&Size, &mut Transform)>) {
    let window = primary_window.get_single().unwrap();
    for (sprite_size, mut transform) in q.iter_mut() {
        scale_sprite(transform.as_mut(), sprite_size, window);
    }
}

#[allow(clippy::missing_panics_doc)]
#[allow(clippy::needless_pass_by_value)]
pub fn position_translation(primary_window: Query<&Window, With<PrimaryWindow>>, mut q: Query<(&Position, &mut Transform)>) {
    let window = primary_window.get_single().unwrap();
    for (pos, mut transform) in q.iter_mut() {
        translate_position(transform.as_mut(), pos, window);
    }
}
// ...
``` 

Existe mais uma mudanca relacionada a `Window` no codigo, que eh no modulo `main`, ao adicionarmos o plugin de window, `WindowPlugin`, pois a forma de declarar a window mudou para:

```rs
fn main() {
    App::new()
        .add_plugins(DefaultPlugins.set(WindowPlugin {
            primary_window: Some(Window { 
                resolution: (1000., 1000.).into(), 
                title: "Snake Game".to_string(),  
                ..default()
            }),
            exit_condition: ExitCondition::OnAllClosed,
            close_when_requested: true,
        }))
        // ...
}
```

Agora vamos terminar o upgrade do modulo `snake.rs`.

### System Set

Podemos ver que o teste `snake_grows_when_eating` possui um erro de compilacao em `add_system_set`, pois a forma como lidamos com system sets mudou bastante, ja que o conceito de `SystemSet` passou a ser apenas uma tupla de sistemas:

```rs
// snake.rs
#[test]
fn snake_grows_when_eating() {
    // Setup
    let mut app = App::new();

    // Add systems
    app.
    // ...
    .add_systems((
        movement_system, 
        eating_system.after(movement_system), 
        growth_system.after(eating_system)
    ));
    // ANTIGO
    // .add_system_set(
    //     SystemSet::new()
    //         .with_system(movement_system)
    //         .with_system(eating_system.after(movement_system))
    //         .with_system(growth_system.after(eating_system)),
    // );

    // ...
}
```

No modulo `main`  encontramos o mesmo problema, mas la existem sistemas com `run_criteria`. A mudanca nesse caso eh simples:

```rs
// main.rs
use std::time::Duration;

use bevy::{prelude::*, time::{common_conditions::on_timer}, window::ExitCondition};
use components::GameEndEvent;
use snake::GrowthEvent;
// ...

fn main() {
    // ...

    // ANTIGO
    // .add_system_set(
    //     SystemSet::new()
    //         .with_run_criteria(FixedTimestep::step(1.0))
    //         .with_system(food::spawn_system),
    // )

    // NOVO
    .add_system(
        food::spawn_system
            .run_if(on_timer(Duration::from_secs_f32(1.0)))
    )

    // ANTIGO
    // .add_system_set(
    //     SystemSet::new()
    //         .with_run_criteria(FixedTimestep::step(0.150))
    //         .with_system(snake::movement_system)
    //         .with_system(snake::eating_system.after(snake::movement_system))
    //         .with_system(snake::growth_system.after(snake::eating_system)),
    // )

    // NOVO
    .add_system(
        snake::movement_system.run_if(on_timer(Duration::from_secs_f32(0.15))))
    .add_system(
        snake::eating_system
            .after(snake::movement_system)
            .run_if(on_timer(Duration::from_secs_f32(0.15)))
        )
    .add_system(
        snake::growth_system
            .after(snake::eating_system)
            .run_if(on_timer(Duration::from_secs_f32(0.15))),
    )
    // ...
}
```

Por ultimo, temos a atualizacao do momento de execucao dos sistemas de `grid`. Na versao `0.9` executavamos eles com `add_system_set_to_stage` e definindo o estagio com  `CoreStage::PostUpdate`. Agora basta adicionarmos `.in_base_set(CoreSet::PostUpdate)` a chamado da tupla de sistemas:

```rs
// ANTIGO
// .add_system_set_to_stage(
//     CoreStage::PostUpdate,
//     SystemSet::new()
//         .with_system(grid::position_translation)
//         .with_system(grid::size_scaling),
// )

// NOVO
.add_systems(
    (grid::position_translation, grid::size_scaling).in_base_set(CoreSet::PostUpdate),
)
```

Migrações concluídas com esse código, caso ocorra alguma incompatibilidade com uma versão nova, por favor abra uma [issue](https://github.com/naomijub/Rust-game-dev/issues) ou um PR nos repositórios do github [livro](https://github.com/naomijub/Rust-game-dev) e [codigo](https://github.com/naomijub/bevy-snake). 