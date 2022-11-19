# Adicionando um Rabo a Cobra

O rabo da cobra é uma parte um pouco mais complexa, pois para cada segmento é preciso saber o próximo segmento e para onde cada segmento esta se movendo. Assim, a forma mais simples de resolver este problema é adicionando todos os segmentos do rabo da cobra em um `Vec` e manter eles em um recurso ordenado em vez de entidades e componentes simples. Essa mudança nos garante que quando atualizarmos a posição de um segmento, atualizamos seu valor com relação ao segmento anterior. Isso pode ser feito iterando sobre todos os segmentos em pares com a função [`windows`](https://doc.rust-lang.org/std/slice/struct.Windows.html) que nos permite acessar o elemento anterior e o atual da lista. Outro elemento importante é que precisamos definir uma cor para o rabo da cobra, define que seria um tom de rosa com:

```rs
// snake.rs
const SNAKE_SEGMENT_COLOR: Color = Color::rgb(0.8, 0.0, 0.8);
```

Agora que definimos uma cor, podemos começar a pensar no primeiro teste. O teste mais simples para este caso parece ser verificar se a entidade relacionada aos segmentos da cobra possui 2 segmento inicialmente:

```rs
// snake.rs
#[test]
fn entity_snake_has_two_segments() {
    // Setup app
    let mut app = App::new();

    // Adicionar sistema de spawn e recurso com segmentos
    app
        .insert_resource(Segments::default())
        .add_startup_system(spawn_system);

    // Executar sistema
    app.update();

    // Buscar todas entidades com componente `Segment`
    let mut query = app.world.query_filtered::<Entity, With<Segment>>();
    assert_eq!(query.iter(&app.world).count(), 2);
}
```

Executando esse teste vemos que precisamos adicionar o recurso `Segments` ao nosso código, um vetor de entidades como falamos anteriormente:

```rs
// snake.rs
#[derive(Default, Deref, DerefMut)]
pub struct Segments(Vec<Entity>);
```

Com isso, precisamos reescrever nosso `snake::spawn_system` para utilizar `Segments`. Como é um startup system, será executado para iniciar entidades do jogo:

```rs
pub fn spawn_system(mut commands: Commands, mut segments: ResMut<Segments>) {
    *segments = Segments(vec![
        commands
            .spawn_bundle(SpriteBundle {
                sprite: Sprite {
                    color: SNAKE_HEAD_COLOR,
                    ..default()
                },
                transform: Transform {
                    scale: Vec3::new(10.0, 10.0, 10.0),
                    ..default()
                },
                ..default()
            })
            .insert(Head::default())
            .insert(Segment)
            .insert(Position { x: 3, y: 3 })
            .insert(Size::square(0.8))
            .id(),
    ]);
}
```

Neste novo `spawn_system` recebemos o recurso `Segments` como um recurso mutável `mut segments: ResMut<Segments>` e definimos `segments` através de uma dereferência mutável, `#[derive(.., DerefMut)]`, reasignando o `Segment::default()`, que equivale a um vetor vazio, ao nosso recem adicionado componente. Outra mudança que precisamos fazer é adicionar o recurso `Segments` ao nosso startup do app na `main.rs` e em todos testes de `snake.rs`:

```rs
// main.rs

fn main() {
    App::new()
        .insert_resource(WindowDescriptor {
            title: "Snake Game".to_string(),
            width: 500.0,
            height: 500.0,
            ..default()
        })
        .insert_resource(snake::Segments::default()) // <-- adicionar
        .add_startup_system(snake::spawn_system)
        // ...
        .run();
}

// snake.rs
#[test]
fn entity_has_snake_head() {
    let mut app = App::new();

    app
        .insert_resource(Segments::default()) // <-- adicionar em todos testes
        .add_startup_system(spawn_system);

    app.update();

    let mut query = app.world.query_filtered::<Entity, With<Head>>();
    assert_eq!(query.iter(&app.world).count(), 1);
}
```

Agora, executando nossos testes percebemos que o novo teste, `entity_snake_has_two_segments`, falha por possuir somente uma entidade. Para isso, precisamos criar um novo sistema que adiciona um novo `Segment` em posição específica e retorna uma `Entity` id para adicionarmos no recurso `Segments`

```rs
pub fn spawn_segment_system(mut commands: Commands, position: Position) -> Entity {
    commands
        .spawn_bundle(SpriteBundle {
            sprite: Sprite {
                color: SNAKE_SEGMENT_COLOR,
                ..default()
            },
            transform: Transform {
                    scale: Vec3::new(10.0, 10.0, 10.0),
                    ..default()
                },
            ..default()
        })
        .insert(Segment)
        .insert(position)
        .insert(Size::square(0.65))
        .id()
}
```

Com este novo sistema, podemos adicionar um segmento extra no nosso `snake::spawn_system`:

```rs
pub fn spawn_system(mut commands: Commands, mut segments: ResMut<Segments>) {
    *segments = Segments(vec![
        commands
            .spawn_bundle(SpriteBundle {
                sprite: Sprite {
                    color: SNAKE_HEAD_COLOR,
                    ..default()
                },
                transform: Transform {
                    scale: Vec3::new(10.0, 10.0, 10.0),
                    ..default()
                },
                ..default()
            })
            .insert(Head::default())
            .insert(Segment)
            .insert(Position { x: 3, y: 3 })
            .insert(Size::square(0.8))
            .id(),
            spawn_segment_system(commands, Position { x: 3, y: 2 }), // <-- novo segmento
    ]);
}
```

Ao executarmos os teste agora, vemos que todos passam e podemos focar no próximo passo, fazer os segmentos se moverem seguindo a cabeça.

## Fazendo o rabo seguir a cabeça