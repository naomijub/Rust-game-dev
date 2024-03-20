# Colisões

Uma parte muito importante de jogos é a definição dos critérios de perda ou derrota. No caso do snake game, há dois critérios:
1. A cobra "come" um pedaço dela mesma.
2. A cobra sai dos limites, ou paredes, do jogo.

Testar a cobra comendo um pedaço dela mesma é bastante complicado considerando um cenário na qual as comidas surgem de forma aleátoria, pois a cobra precisa possuir pelo menos 5 segmentos para que ocorra uma colisão da cabeça da cobra com um segmento. Neste caso, um teste de gameplay seria mais fácil e possivelmente mais valioso, porém não é algo que planejei dentro do escopo deste livro. Por outro lado, testar que a cobra sai dos limites do jogo é bastante trivial, basta definir uma direção e garantir que após `n` updates, a cobra vai colidir com as paredes. Uma vez que a condição de colisão ocorreu, podemos publicar um evento de _game end_, e pausa o jogo com um status de jogo. Único teste que não vou escrever neste caso é o teste de colisão com a parede de baixo, mas seria igual aos outros, porém com 3 updates extras para fazer retorno e mudar a direção para baixo.

O primeiro teste consite basicamente em fazer com que a cobra se movimente para cima até ultrapassar a parede superior e ai detectamos um componente do tipo `GameEndEvent::GameOver`, derivado do evento `GameEndEvent`. Adicionaremos este teste em um novo módulo chamado `game.rs`:

```rust
#[test]
fn game_end_event_with_game_over() {
    // Setup
    let mut app = App::new();

    // Sistemas
    app.insert_resource(Segments::default())
        .insert_resource(LastTailPosition::default())
        .add_event::<GameEndEvent>() // <--
        .add_startup_system(snake::spawn_system)
        .add_system(snake::movement_system)
        .add_system(snake::movement_input_system.before(snake::movement_system))
        .add_system(game_over_system.after(snake::movement_system)); // <--

    // tecla para cima
    let mut input = Input::<KeyCode>::default();
    input.press(KeyCode::W);
    app.insert_resource(input);

    // executgar sistema algumas vezes
    app.update(); // x: 3, y: 4
    app.update(); // x: 3, y: 5
    app.update(); // x: 3, y: 6
    app.update(); // x: 3, y: 7
    app.update(); // x: 3, y: 8
    app.update(); // x: 3, y: 9

    // Verificar que não há componente de game end
    let mut query = app.world.query::<&GameEndEvent>();
    assert_eq!(query.iter(&app.world).count(), 0);

    app.update(); // x: 3, y: 10

    // Verificar que há componente de game end
    let mut query = app.world.query::<&GameEndEvent>();
    assert_eq!(query.iter(&app.world).count(), 1);
}
```

Com este teste podemos começar a implementar o primeiro critério de falha, que neste caso seria `y` da posição da cabeça menor que zero ou maior ou igual a `GRID_HEIGHT`, ou seja, `head.position.y < 0 || head.position.y >= GRID_HEIGHT`. Na função `snake::movement_system`, temos acesso a `head.position` dentro do block que contém o `match head.direction`, assim podemos adicionar a condicional de posições depois do match e publicar o evento `GameEndEvent::GameOver` pelo `EventWriter` que precisamos adicionar nos argumentos da função:

```rust
pub fn movement_system(
    segments: ResMut<Segments>,
    mut last_tail_position: ResMut<LastTailPosition>,
    mut game_end_writer: EventWriter<GameEndEvent>, // <-- Adicionar EventWriter
    mut heads: Query<(Entity, &Head)>,
    mut positions: Query<(Entity, &Segment, &mut Position)>,
) {
    let positions_clone: HashMap<Entity, Position> = positions
        .iter()
        .map(|(entity, _segment, position)| (entity, position.clone()))
        .collect();
    if let Some((id, head)) = heads.iter_mut().next() {
        (*segments).windows(2).for_each(|entity| {
            if let Ok((_, _segment, mut position)) = positions.get_mut(entity[1]) {
                if let Some(new_position) = positions_clone.get(&entity[0]) {
                    *position = new_position.clone();
                }
            };
        });
        
        let _ = positions.get_mut(id).map(|(_, _segment, mut pos)| {
            match &head.direction {
                Direction::Left => {
                    pos.x -= 1;
                }
                Direction::Right => {
                    pos.x += 1;
                }
                Direction::Up => {
                    pos.y += 1;
                }
                Direction::Down => {
                    pos.y -= 1;
                }
            };
            if pos.y < 0
                || pos.y as u16 >= GRID_HEIGHT // <-- Condicional de limites do grid
            {
                game_end_writer.send(GameEndEvent::GameOver); // <-- publicar evento
            }
        });
        
        *last_tail_position = LastTailPosition(Some(
            positions_clone
                .get(segments.last().unwrap())
                .unwrap()
                .clone(),
        ));
    }
}
```

Agora todos os testes que lidam com `movement_system` falham e é preciso adicionar `.add_event::<GameEndEvent>()` ao setup de sistemas, além disso, adicione a função `main`. Outro elemento importante que é adicionar o `GameEndEvent`, que vamos adicionar no módulo `components.rs`:


```rust
// main.rs

...
fn main() {
    App::new()
        .add_systems(Startup, setup_camera)
        .insert_resource(snake::Segments::default())
        .insert_resource(snake::LastTailPosition::default())
        .add_event::<GameEndEvent>() // <-- Adicionar
        .add_event::<snake::GrowthEvent>()
...

```

```rust
// components.rs

#[derive(Component, Clone, Debug, Event, PartialEq, Eq)]
pub enum GameEndEvent {
    GameOver,
}

impl Default for GameEndEvent {
    fn default() -> Self {
        Self::GameOver
    }
}
```

Perfeito, mas o teste ainda não passa, pois não temos nenhum sistema escutando pelo evento `GameEndEvent`, podemos adicionar um sistema `game_over_system` que adicionará o componente `GameEndEvent::GameOver` que buscamos no teste. Este sistema verificará se existe algum evento do tipo `GameEndEvent`, se houver cria uma entidade com `GameEndEvent::GameOver` como componente e da um print no console com `"Game Over!"`;

```rust
// game.rs

pub fn game_over_system(mut commands: Commands, mut reader: EventReader<GameEndEvent>) {
    if reader.iter().next().is_some() {
        commands.spawn().insert(GameEndEvent::GameOver);
        println!("{}", GameEndEvent::GameOver);
    }
}
...
```

```rust
// main.rs

...

```

Para printar no console um enum podemos implementar a trait Display:

```rust
// components.rs
...
use std::fmt::{self, Display};
...

impl Display for GameEndEvent {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            GameEndEvent::GameOver => write!(f, "Game Over!"),
        }
    }
}
...
```

Os testes abaixo vão falhar.
```bash
    snake::test::snake_cannot_start_moving_down
    snake::test::snake_grows_when_eating
    snake::test::snake_head_has_moved_up
    snake::test::snake_head_moves_down_and_left
    snake::test::snake_head_moves_up_and_right
    snake::test::snake_segment_has_followed_head
```
Para resolver isso precisamos adicionar o `.add_event::<GameEndEvent>()` no setup dos apps.

Agora nosso teste passa, mas quero adicionar um assert extra no nosso teste, que a posição da cobra não mudará após um game over. Fazemos isso adicionando uma verificação que a posição após o `GameEndEvent::GameOver` não mudará mesmo após updates.

```rust
#[test]
fn game_end_event_with_game_over() {
    // ...

    let mut query = app.world.query::<&GameEndEvent>();
    assert_eq!(query.iter(&app.world).count(), 1);

    let mut query = app.world.query_filtered::<&Position, With<Head>>();
    let position_at_gameover = query.iter(&app.world).next().unwrap();
    let snake_position_after_game_over = position_at_gameover.clone();

    app.update();

    let mut query = app.world.query_filtered::<&Position, With<Head>>();
    let position_after_gameover = query.iter(&app.world).next().unwrap();

    assert_eq!(snake_position_after_game_over, position_after_gameover.clone());
}
```

Após adicionar essas linhas o teste quebrará.

Essa mudança é facilmente resolvida adicionando uma query que busca por um `GameEndEvent`, `Query<&GameEndEvent>`, e verificando se ela não está vazia em um `if`:

```rs
pub fn movement_system(
    segments: ResMut<Segments>,
    mut last_tail_position: ResMut<LastTailPosition>,
    mut game_end_writer: EventWriter<GameEndEvent>,
    heads: Query<(Entity, &Head)>,
    mut positions: Query<(Entity, &Segment, &mut Position)>,
    game_end: Query<&GameEndEvent>, // <-- GameEndEvent Query
) {
    // ...
    if let Some((id, head)) = heads.iter().next() {
        (*segments).windows(2).for_each(|entity| {
            if let Ok((_, _segment, mut position)) = positions.get_mut(entity[1]) {
                if let Some(new_position) = positions_clone.get(&entity[0]) {
                    *position = new_position.clone();
                }
            };
        });
        if game_end.is_empty() {  // <-- if verificando se houve um evento de fimd e jogo
            let _ = positions.get_mut(id).map(|(_, _segment, mut pos)| {
                match &head.direction {
                    // ...
                };
                if pos.y < 0
                    || pos.y as u16 >= GRID_HEIGHT
                {
                    game_end_writer.send(GameEndEvent::GameOver);
                }
            });
        }
        // ...
    }
}
```

Próximo teste é verificar se o mesmo acontece se movendo para esquerda e para a direita. Começamos pela esquerda:

```rust
    #[test]
    fn game_end_event_with_game_over_when_moving_left() {
        // Setup
        let mut app = App::new();

        // Add systems
        app.insert_resource(Segments::default())
            .insert_resource(LastTailPosition::default())
            .add_event::<GameEndEvent>() // <--
            .add_systems(Startup, snake::spawn_system)
            .add_systems(Update, snake::movement_system)
            .add_systems(
                Update,
                snake::movement_input_system.before(snake::movement_system),
            )
            .add_systems(Update, game_over_system.after(snake::movement_system)); // <--

        // Add new input resource
        let mut input = Input::<KeyCode>::default();
        input.press(KeyCode::A);
        app.insert_resource(input);

        // Run systems again
        app.update(); // x: 4, y: 5
        app.update(); // x: 3, y: 5
        app.update(); // x: 2, y: 5
        app.update(); // x: 1, y: 5
        app.update(); // x: 0, y: 5

        let mut query = app.world.query::<&GameEndEvent>();
        assert_eq!(query.iter(&app.world).count(), 0);

        app.update(); // x: -1, y: 5

        let mut query = app.world.query::<&GameEndEvent>();
        assert_eq!(query.iter(&app.world).count(), 1);
    }

```

Com isso adicionamos a verificação se `head.position` não é menor que zero:

```rs
pub fn movement_system(
    // ...
) {
    // ...
    if let Some((id, head)) = heads.iter().next() {
        // ...
        if game_end.is_empty() {  // <-- if verificando se houve um evento de fimd e jogo
            let _ = positions.get_mut(id).map(|(_, _segment, mut pos)| {
                match &head.direction {
                    // ...
                };
                if pos.x < 0 // <-- Nova Verificação
                    || pos.y < 0
                    || pos.y as u16 >= GRID_HEIGHT
                {
                    game_end_writer.send(GameEndEvent::GameOver);
                }
            });
        }
        // ...
    }
}
```

Depois, repetimos o teste se movendo para direita e com uma verificação se `head.position.x` maior ou igual a `GRID_WIDTH`:

```rust
#[test]
fn game_end_event_with_game_over_when_moving_right() {
    // Setup
    let mut app = App::new();

    // Add systems
    app.insert_resource(Segments::default())
        .insert_resource(LastTailPosition::default())
        .add_event::<GameEndEvent>() // <--
        .add_systems(Startup, snake::spawn_system)
        .add_systems(Update, snake::movement_system)
        .add_systems(
            Update,
            snake::movement_input_system.before(snake::movement_system),
        )
        .add_systems(Update, game_over_system.after(snake::movement_system)); // <--

    // Add new input resource
    let mut input = Input::<KeyCode>::default();
    input.press(KeyCode::D);
    app.insert_resource(input);

    // Run systems again
    app.update(); // x: 5, y: 5
    app.update(); // x: 6, y: 5
    app.update(); // x: 7, y: 5
    app.update(); // x: 8, y: 5
    app.update(); // x: 9, y: 5
    app.update(); // x: 10, y: 5



    let mut query = app.world.query::<&GameEndEvent>();
    assert_eq!(query.iter(&app.world).count(), 1);
}
```
Agora adicionamos a condição para o teste passar.

```rust
pub fn movement_system(
    // ...
) {
    // ...
    if let Some((id, head)) = heads.iter().next() {
        // ...
        if game_end.is_empty() {  
            let _ = positions.get_mut(id).map(|(_, _segment, mut pos)| {
                match &head.direction {
                    // ...
                };
                if pos.x < 0
                    || pos.y < 0
                    || pos.x as u16 >= GRID_WIDTH // <-- Nova verificação
                    || pos.y as u16 >= GRID_HEIGHT
                {
                    game_end_writer.send(GameEndEvent::GameOver);
                }
            });
        }
        // ...
    }
}
```

## Colidindo com o rabo

Como mencionei antes, escrever um teste para este cenário é um pouco mais trabalhoso que eu gostaria e acaba sendo mais fácil fazer com alguma ferramenta de testes automatizados, mas caso você queira um desafio, para escrever este teste você pode executar o sistema de spawn de segmentos (`spawn_segment_system`) com posições, `Position`, especificas e ao realizar um update a posição de `Head` vai ser igual a posição de um elemento do rabo. Agora vamos ao código, é uma mudança muito simples em movement system, basta adicionarmos mais uma cláusula `if` que checa se a posição de `Head` é a mesma que qualquer posição de `Segment`, infelizmente não temos uma estrutura de dados que possui todas a `Positions` com `Segments` identificadas, mas possuimos `positions_clone` que é um `HashMap<Entity, Position>`. 

Para descobrirmos o valor de position que não contém `Head` precisamos filtrar por todas `Positions`, cuja `Entity` correspondente não é igual ao `id` de `Head`, algo como `positions_clone.iter().filter(|(k, _)| k != &&id)`. Com isso, teremos um iterável que possui todos os pares `Entity, Position` que não correspondem ao conjunto `Entity, Position, Head` e podemos continuar iterando somente com `Positions` adicionando `.map(|(_, v)| v)`, para depois verificamos se existe qualquer `Position` que equivale ao par `Head, Position`, utilizando o valor da variável `pos`, `.any(|segment_position| &*pos == segment_position)`.  Adicionamos esta lógica logo após o outro if de game over e publicamos outro `GameEndEvent::GameOver`:

```rs
pub fn movement_system(
    segments: ResMut<Segments>,
    mut last_tail_position: ResMut<LastTailPosition>,
    mut game_end_writer: EventWriter<GameEndEvent>,
    heads: Query<(Entity, &Head)>,
    mut positions: Query<(Entity, &Segment, &mut Position)>,
    game_end: Query<&GameEndEvent>,
) {
    let positions_clone: HashMap<Entity, Position> = positions
        .iter()
        .map(|(entity, _segment, position)| (entity, position.clone()))
        .collect();
    if let Some((id, head)) = heads.iter().next() {
        // ...
        if game_end.is_empty() {
            let _ = positions.get_mut(id).map(|(_, _segment, mut pos)| {
                match &head.direction {
                    // ...
                };
                if pos.x < 0
                    || pos.y < 0
                    || pos.x as u16 >= GRID_WIDTH
                    || pos.y as u16 >= GRID_HEIGHT
                {
                    game_end_writer.send(GameEndEvent::GameOver);
                }

                if positions_clone.iter()
                    .filter(|(k, _)| k != &&id)
                    .map(|(_, v)| v)
                    .any(|segment_position| &*pos == segment_position)
                {
                    game_end_writer.send(GameEndEvent::GameOver);
                }
            });
        }
        // ...
    }
}
```

Agora é hora de um teste manual e voilá, "a cobra morde o rabo!". Proxima colisão que devemos impedir é a de comidas surgindo em posições já ocupadas. 

## Colisões de surgimento de comdias

Particularmente não sou fã dessa, pois na minha concepção uma comida deveria poder surgir embaixo da cobra, desde que não seja na cabeça, mas vale a explicação pelo exemplo. Assim, o teste que vamos escrever é bastante simples, pois vamos apenas checar se a quantidade de entidades com os componentes `Food` e `Position` é `1`, apesar de termos dois updates. Podemos fazer isso por conta  da condição de `spawn` associada a testes em `food::spawn_system`, quando utilizados `if cfg!(test)` com valores pré-fixados.

```rust
#[test]
fn food_only_spawns_once() {
    // Setup
    let mut app = App::new();

    // Add systems
    app.add_system(spawn_system);


    // Run systems
    app.update();

    let mut query = app.world.query::<(&Food, &Position)>();
    assert_eq!(query.iter(&app.world).count(), 1);

    // Run systems
    app.update();

    let mut query = app.world.query::<(&Food, &Position)>();
    assert_eq!(query.iter(&app.world).count(), 1)
}
```

A solução para este teste é bastante simples, precisamos obter uma posição que não coincide com outra posição, fazemos isso com um iterador infinito, que procura pela primeira posição que não coincide com outra. Esse iterator pode ser feita com um `Range` do tipo `(0..)` (de `0` a infinito), depois criamos instâncias aleatórias de `Position` e procuramos por uma `Position` que não está contida em um `HashSet` de `Position`.

```rust
(0..)
    .map(|_| Position {
        x: if cfg!(test) {
            5
        } else {
            (random::<u16>() % GRID_WIDTH) as i16
        },
        y: if cfg!(test) {
            7
        } else {
            (random::<u16>() % GRID_HEIGHT) as i16
        },
    })
    .find(|position| !positions_set.contains(position))
```

`positions_set` é o `HashSet<Position>` que falamos antes, podemos criar ele através de `let positions_set: HashSet<&Position> = positions.iter().collect();`, porém `Position` não implementa a trait `Hash`, que é facilmente resolvível adicionando a macro `Hash` ao `derive` de `Position`:

```rust
#[derive(Component, Clone, Debug, PartialEq, Eq, Hash)]
pub struct Position {
    pub x: i16,
    pub y: i16,
}

```

Agora, precisamos adicionar uma comida ao jogo apenas se o retorno de find é existente, `Option::Some`:

```rust
pub fn spawn_system(mut commands: Commands, positions: Query<&Position>) {
    let positions_set: HashSet<&Position> = positions.iter().collect();

    if let Some(position) = (0..)
        .map(|_| Position {
            x: if cfg!(test) {
                5
            } else {
                (random::<u16>() % GRID_WIDTH) as i16
            },
            y: if cfg!(test) {
                7
            } else {
                (random::<u16>() % GRID_HEIGHT) as i16
            },
        })
        .find(|position| !positions_set.contains(position))
    {
        commands
            .spawn_bundle(SpriteBundle {
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

Para resolvermos esse problema, encapsulamos nosso iterador infinito em um `if let` e em caso de `Option::Some`, adicionamos uma nova comida. Porém, do jeito que escrevemos o iterador infinito vai quebrar os testes já que nunca vai encontrar uma `Position` válida em testes. Assim, podemos fazer uma aproximação para o tamanho do grid, `(0..(GRID_WIDTH * GRID_HEIGHT))`:

```rust
pub fn spawn_system(mut commands: Commands, positions: Query<&Position>) {
    let positions_set: HashSet<&Position> = positions.iter().collect();

    if let Some(position) = (0..(GRID_WIDTH * GRID_HEIGHT))
        .map(|_| Position {
            x: if cfg!(test) {
                5
            } else {
                (random::<u16>() % GRID_WIDTH) as i16
            },
            y: if cfg!(test) {
                7
            } else {
                (random::<u16>() % GRID_HEIGHT) as i16
            },
        })
        .find(|position| !positions_set.contains(position))
    {
        commands
            .spawn_bundle(SpriteBundle {
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

Agora sim, testes passando e comidas surgem de forma eficiente. 
