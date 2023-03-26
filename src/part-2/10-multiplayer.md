# Multiplayer Local

**INICIALMENTE ESCRITO NA VERSÃO 0.9 DA BEVY**

Primeiro passo para entendermos um jogo multiplayer é criarmos as regras para o jogo executar corretamente no modo multiplayer, podemos fazer isso adicionando mais um player de forma local. O suporte ao multiplayer exige uma pequena refatoração, que será por onde começaremos.

## Refatorando

Com a atualização do Rust para versão `1.66`, o linter do Rust sugeriu algumas novas refatorações bem simples, mas muito bem observadas no módulo `grid`. A primeira é na função `translate_position` e na função `scale_sprite` que possuíam um casting desnecessário, podendo-se remover o `as f32` das chamadas de funções `window.width()` e `window.height()`:

```rs
// Grid.rs
// Antes
fn scale_sprite(transform: &mut Transform, sprite_size: &Size, window: &Window) {
    transform.scale = Vec3::new(
        sprite_size.width / GRID_WIDTH as f32 * window.width() as f32,
        sprite_size.height / GRID_HEIGHT as f32 * window.height() as f32,
        1.0,
    );
}

fn translate_position(transform: &mut Transform, pos: &Position, window: &Window) {
    transform.translation = Vec3::new(
        convert(pos.x as f32, window.width() as f32, GRID_WIDTH as f32),
        convert(pos.y as f32, window.height() as f32, GRID_HEIGHT as f32),
        0.0,
    );
}

// Depois
fn scale_sprite(transform: &mut Transform, sprite_size: &Size, window: &Window) {
    transform.scale = Vec3::new(
        sprite_size.width / GRID_WIDTH as f32 * window.width(),
        sprite_size.height / GRID_HEIGHT as f32 * window.height(),
        1.0,
    );
}

fn translate_position(transform: &mut Transform, pos: &Position, window: &Window) {
    transform.translation = Vec3::new(
        convert(pos.x as f32, window.width(), GRID_WIDTH as f32),
        convert(pos.y as f32, window.height(), GRID_HEIGHT as f32),
        0.0,
    );
}
```

A segunda refatoração é, no mesmo módulo, a simplificação da função `convert` para utilizar a função `mul_add` em vez de uma multiplicação seguida por uma adição. A vantagem do uso de `mul_add` é que reduz os erros por arredondamento que poderiam ser causados pelo uso de `f32`:

```rs
// grid.rs
// Antes
fn convert(pos: f32, bound_window: f32, grid_side_lenght: f32) -> f32 {
    let tile_size = bound_window / grid_side_lenght;
    pos / grid_side_lenght * bound_window - (bound_window / 2.) + (tile_size / 2.)
}

// Depois
fn convert(pos: f32, bound_window: f32, grid_side_lenght: f32) -> f32 {
    let tile_size = bound_window / grid_side_lenght;
    (pos / grid_side_lenght).mul_add(bound_window, -bound_window / 2.) + (tile_size / 2.)
}
```

> **Trait `MulAdd`** equivale a representar `(self * a) + b` e tem como sintaxe `fn mul_add<A: Self, B: Self>(self, a: A, b: B) -> Self`.

Outra refatoração que fiz foi para melhorar os resultados de testes em modo `release`  e em Windows, utilizando a biblioteca `approx = "0.5.1"`. Esta biblioteca garante uma comparação mais correta de epsilons de f32. Assim, podemos mudar todos os testes que contém comparações de f32 para utilizar o `assert_relative_eq`, que recebe um epsilon de valor de erro na comparação:

```rs
// grid.rs
mod test {
    use super::*;
    use approx::assert_relative_eq;

    fn convert_position_x_for_grid_width() {
        let x = convert(4., 400., GRID_WIDTH as f32);

        assert_relative_eq!(x, -20., epsilon = 0.00001);
    }

    #[test]
    fn convert_position_y_for_grid_height() {
        let x = convert(5., 400., GRID_HEIGHT as f32);

        assert_relative_eq!(x, 20., epsilon = 0.00001);
    }
    // ...
}
```

Por último, vamos atualizar os testes com múltiplas chamadas de `app.update()`. Fazemos isso substituindo todas as linhas por um `for` com range:
```rs
// Antes
app.update(); // 3 + 1
app.update(); // 3 + 2
app.update(); // 3 + 3
app.update(); // 3 + 4
app.update(); // 3 + 5
app.update(); // 3 + 6

// Depois
for _ in 0..6 {
    app.update(); // 3 + _
}
```

## Criando Outro Player

A primeira coisa que precisamos para começar a dar suporte para multiplayer local é adicionarmos o conceito de player, ou seja, um componente chamado `Player` que possui um `id` com o id do player, podemos fazer isso no módulo `components.rs`. Além disso, vale a pena criar uma função constante para retornar o valor de `id`:

```rs
#[derive(Component, Clone, Debug, PartialEq, Eq, PartialOrd, Ord)]
pub struct Player {
    id: u8,
}

impl Player {
    pub const fn id(&self) -> usize {
        self.id as usize
    }
}
```

Depois disso, podemos aumentar o tamanho do mapa para que duas cobras possam andar juntas sem se colidir constantemente, fazemos isso mudando o valor da janela gerada no `WindowPlugin` para `1000f32`:

```rs
.add_plugins(DefaultPlugins.set(WindowPlugin {
    window: WindowDescriptor {
        title: "Snake Game".to_string(),
        width: 1000.0,
        height: 1000.0,
        ..default()
    },
    // ...
```

Somente esta mudança não garante mais espaço para duas cobras, assim, em `grid.rs` aumentamos a quantidade de tiles de `10` para `20`, mas apenas em modo release. Definimos compilação em modo `debug` com a flag de compilação `#[cfg(debug_assertions)]` e modo `release` com a flag de compilação `#[cfg(not(debug_assertions))]`, um `not` a mais:

```rs
#[cfg(debug_assertions)]
pub(crate) const GRID_WIDTH: u16 = 10;
#[cfg(not(debug_assertions))]
pub(crate) const GRID_WIDTH: u16 = 20;
#[cfg(debug_assertions)]
pub(crate) const GRID_HEIGHT: u16 = 10;
#[cfg(not(debug_assertions))]
pub(crate) const GRID_HEIGHT: u16 = 20;
```

### Testando com uma janela maior
Essa mudança fará com que uma série de testes falhem em modo `release`, necessário para windows, assim, as correções passam a ser utilizar asserts ou declarações diferentes em modo `debug` e modo `release`. Por exemplo:

```rust
// Grid.rs
fn convert_position_x_for_grid_width() {
    let x = convert(4., 400., GRID_WIDTH as f32);

    #[cfg(debug_assertions)] // <-- DEBUG
    assert_relative_eq!(x, -20., epsilon = 0.00001); // <-- DEBUG
    #[cfg(not(debug_assertions))] // <-- RELEASE
    assert_relative_eq!(x, -110., epsilon = 0.00001); // <-- RELEASE
}

#[test]
fn translate_position_to_window() {
    let position = Position { x: 2, y: 8 };
    let mut default_transform = Transform::default();
    let expected = Transform {
        #[cfg(debug_assertions)] // <-- DEBUG
        translation: Vec3::new(-100., 140., 0.), // <-- DEBUG
        #[cfg(not(debug_assertions))] // <-- RELEASE
        translation: Vec3::new(-150., -29.999996, 0.), // <-- RELEASE
        ..default()
    };

    let mut descriptor = WindowDescriptor::default();
    descriptor.height = 400.;
    descriptor.width = 400.;
    let window = Window::new(WindowId::new(), &descriptor, 400, 400, 1., None, None);
    translate_position(&mut default_transform, &position, &window);
    
    assert_eq!(default_transform, expected);
}
```

Mais exemplos podem ser encontrados no [PR#14](https://github.com/naomijub/bevy-snake/pull/14/files), inclusive as mudanças para os testes das próximas partes.

## Utilizando `Player`

Primeira mudança que fiz foi algo bem simples, mas muito representativo, adicionar uma outra cor de segmentos de cobra, `SNAKE_SEGMENT_COLOR` virou:

```rs
const SNAKE_HEAD_COLOR: Color = Color::rgb(0.7, 0.7, 0.7);
const SNAKE1_SEGMENT_COLOR: Color = Color::rgb(0.8, 0.0, 0.8); // <--
const SNAKE2_SEGMENT_COLOR: Color = Color::rgb(0., 0.8, 0.8); // <--
```

Próximo passo é adicionarmos uma referência ao vetor de segmentos da segunda cobra, podemos fazer isso modificando a struct `Segments` para `pub struct Segments([Vec<Entity>; 2]);`, que significa que nossa struct `Segments` possui um array de tamanho 2 do tipo Vetor de `Entity`, assim o id do player 1 será `0` e o id do player 2 será `1`, conforme o sistema de indexação de arrays. Essa pequena mudança quebra todo nosso código para `snake.rs`, mas vou tentar explicar como ajustar as funções do modo mais lógico possível.

### Sistema de Geração de Cobras (Spawn)

Agora nosso sistema de spawn exige que segments seja um Array com dois vetores de entidades, para isso podemos refatorar o código dentro de `spawn_system` para ser reutilizável. Fazemos isso criando a função privada `spawn_entity_with_segment`, que recebe uma referência mutável de `Commands` e um `u8` que corresponderá ao id do Player. É importante que o segmento da cabeça da cobra seja ciente de qual seu `player_id`, para isso adicionamos essa informação nos componentes da primeira entidade, `.insert(components::Player { id: player_id })`, e que player 1 e player 2 não iniciem na mesma posição, para isso utilizamos um `if/else` na posição X do componente `Position`, definindo a posição do player 1 em `x = 3`, e do player 2 em `x = GRID_WIDTH - 3`, `if player_id == 0 { 3 } else { (GRID_WIDTH - 3) as i16 }`:

```rs
fn spawn_entity_with_segment(commands: &mut Commands, player_id: u8) -> Vec<Entity> {
    vec![
        commands
            .spawn(SpriteBundle {
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
            .insert(components::Player { id: player_id })
            .insert(Head::default())
            .insert(Segment)
            .insert(Position {
                x: if player_id == 0 {
                    3
                } else {
                    (GRID_WIDTH - 3) as i16
                },
                y: 3,
            })
            .insert(Size::square(0.8))
            .id(),
        spawn_segment_system(
            commands,
            Position {
                x: if player_id == 0 {
                    3
                } else {
                    (GRID_WIDTH - 3) as i16
                },
                y: 2,
            },
            player_id,
        ),
    ]
}
```

Com essa mudança podemos atualizar a função `spawn_system` para chamar a função `spawn_entity_with_segment` para ambos os players:

```rs
pub fn spawn_system(mut commands: Commands, mut segments: ResMut<Segments>) {
    *segments = Segments([
        spawn_entity_with_segment(&mut commands, 0),
        spawn_entity_with_segment(&mut commands, 1),
    ]);
}
```

Note que `spawn_entity_with_segment` recebe como argumento `commands: &mut Commands` e por isso, devemos modificar o tipo de `commands` em `spawn_segment_system` para corresponder a mesma referência mutável. É nesta função, `spawn_segment_system` que vamos definir a cor dos segmentos de player que criamos anteriormente, com `if player_id == 0 { SNAKE1_SEGMENT_COLOR } else { SNAKE2_SEGMENT_COLOR }`:

```rs
pub fn spawn_segment_system(commands: &mut Commands, position: Position, player_id: u8) -> Entity {
    commands
        .spawn(SpriteBundle {
            sprite: Sprite {
                color: if player_id == 0 {
                    SNAKE1_SEGMENT_COLOR
                } else {
                    SNAKE2_SEGMENT_COLOR
                },
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

### Sistema de Movimento

A base para o sistema de movimento é fazer com que o `wasd` mova o player 1 e as setas (`arrows`) movam o player 2. Agora, nosso `movement_input_system` não pode se restringir a iterar sobre `heads` apenas com `.next()`, já que sabemos que há pelo menos dois elementos em `heads`. Além disso, precisamos de uma forma de determinar de qual player estamos falando, por isso adicionamos o componente `Player` na `Query` de `heads`, `mut heads: Query<(&mut Head, &Player)>`, e iteramos sobre todos os elementos com um `iter_mut().for_each((mut head, player)| { ... })`. O resto do código segue a mesma lógica de antes:

```rs
pub fn movement_input_system(
    keyboard_input: Res<Input<KeyCode>>,
    mut heads: Query<(&mut Head, &Player)>,
) {
    heads.iter_mut().for_each(|(mut head, player)| {
        let dir: Direction = if player.id() == 0 {
            if keyboard_input.pressed(KeyCode::A) {
                Direction::Left
            } else if keyboard_input.pressed(KeyCode::S) {
                Direction::Down
            } else if keyboard_input.pressed(KeyCode::W) {
                Direction::Up
            } else if keyboard_input.pressed(KeyCode::D) {
                Direction::Right
            } else {
                head.direction
            }
        } else if player.id() == 1 {
            if keyboard_input.pressed(KeyCode::Left) {
                Direction::Left
            } else if keyboard_input.pressed(KeyCode::Down) {
                Direction::Down
            } else if keyboard_input.pressed(KeyCode::Up) {
                Direction::Up
            } else if keyboard_input.pressed(KeyCode::Right) {
                Direction::Right
            } else {
                head.direction
            }
        } else {
            head.direction
        };
        if dir != head.direction.opposite() {
            head.direction = dir;
        }
    });
}
```

A mudança no `movement_system` é essencialmente a mesma, agora precisamos adicionar a informação de `Player` na `Query` de `heads`, `heads: Query<(Entity, &Head, &Player)>,`, e refatorar a iteração sobre `heads` para ser um `for` em vez de um `.next()`, considerando que agora possuímos o `id` de `Player`, que causa colisão com o antigo `id`, que mudei para `entity_id`. Note que estamos destruturando o valor de `Player` em `id` ao utilizarmos `Player { id }` no `for` loop. A única outra grande mudança é que agora para acessarmos `segments`, precisamos indicar qual o `player_id` do segment, fazemos isso substituindo as referências a `segments` por `segments[player_id]`:

```rs
pub fn movement_system(
    segments: ResMut<Segments>,
    mut last_tail_position: ResMut<LastTailPosition>,
    mut game_end_writer: EventWriter<GameEndEvent>,
    heads: Query<(Entity, &Head, &Player)>,
    mut positions: Query<(Entity, &Segment, &mut Position)>,
    game_end: Query<&GameEndEvent>,
) {
    let positions_clone: HashMap<Entity, Position> = positions
        .iter()
        .map(|(entity, _segment, position)| (entity, position.clone()))
        .collect();
    for (entity_id, head, Player { id }) in heads.iter() {
        let player_id = (*id) as usize;
        (*segments[player_id]).windows(2).for_each(|entity| {
            if let Ok((_, _segment, mut position)) = positions.get_mut(entity[1]) {
                if let Some(new_position) = positions_clone.get(&entity[0]) {
                    *position = new_position.clone();
                }
            };
        });
        if game_end.is_empty() {
            let _ = positions.get_mut(entity_id).map(|(_, _segment, mut pos)| {
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
                if pos.x < 0
                    || pos.y < 0
                    || pos.x as u16 >= GRID_WIDTH
                    || pos.y as u16 >= GRID_HEIGHT
                {
                    game_end_writer.send(GameEndEvent::GameOver);
                }

                if positions_clone
                    .iter()
                    .filter(|(k, _)| k != &&entity_id)
                    .map(|(_, v)| v)
                    .any(|segment_position| &*pos == segment_position)
                {
                    game_end_writer.send(GameEndEvent::GameOver);
                }
            });
        }
        *last_tail_position = LastTailPosition(Some(
            positions_clone
                .get(segments[player_id].last().unwrap())
                .unwrap()
                .clone(),
        ));
    }
}
```

### Sistema de crescimento

A única mudança realmente significativa no sistema de crescimento é que o evento `GrowthEvent` precisa ter informação de qual player deve crescer, fazemos isso adicionando a informação de player ao `GrowthEvent`:

```rs
pub struct GrowthEvent {
    pub player_id: u8,
}
```

Agora, nosso `eating_system`, precisa transmitir a informação do `id` do `Player` para o `GrowthEvent`, fazemos isso adicionando `Player` a query de `head_positions`, `head_positions: Query<(&Position, &Player), With<Head>>,` e definindo o `id` destruturado como `player_id`  em `GrowthEvent`:

```rs
pub fn eating_system(
    mut commands: Commands,
    mut growth_writer: EventWriter<GrowthEvent>,
    food_positions: Query<(Entity, &Position), With<Food>>,
    head_positions: Query<(&Position, &Player), With<Head>>,
) {
    for (head_pos, Player { id }) in head_positions.iter() {
        for (ent, food_pos) in food_positions.iter() {
            if food_pos == head_pos {
                commands.entity(ent).despawn();
                growth_writer.send(GrowthEvent { player_id: *id });
            }
        }
    }
}
```

A próxima mudança é fazer com que o sistema de crescimento reconheça as informações relativas a `player_id`, contidas em `GrowthEvent`. Como pode haver mais de um evento no `EventBuffer`, precisamos iterar sobre o `growth_reader` com um `for_each`, em vez de um `.next()`, além disso, para garantir um sistema robusto, fazemos uma checagem se o `player_id` é menor que o tamanho do array `segments`, já que vamos acessar o array utilizando indexação sem checagem, `segments.[player_id]`. As outras mudanças simplesmente refletem as mudanças esperadas em `spawn_segment_system`:

```rs
pub fn growth_system(
    mut commands: Commands,
    last_tail_position: Res<LastTailPosition>,
    mut segments: ResMut<Segments>,
    mut growth_reader: EventReader<GrowthEvent>,
) {
    growth_reader.iter().for_each(|event| {
        let player_id = event.player_id as usize;
        if player_id < segments.len() {
            segments[player_id].push(spawn_segment_system(
                &mut commands,
                last_tail_position.0.clone().unwrap(),
                event.player_id,
            ));
        }
    });
}
```
