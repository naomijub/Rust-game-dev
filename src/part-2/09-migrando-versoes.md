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

Migrações concluídas com esse código, caso ocorra alguma incompatibilidade com uma versão nova, por favor abra uma [issue](https://github.com/naomijub/Rust-game-dev/issues) ou um PR nos repositórios do github [livro](https://github.com/naomijub/Rust-game-dev) e [codigo](https://github.com/naomijub/bevy-snake). 