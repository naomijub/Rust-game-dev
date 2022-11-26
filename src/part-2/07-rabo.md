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

Agora, executando nossos testes percebemos que o novo teste, `entity_snake_has_two_segments`, falha por possuir somente uma entidade. Para isso, precisamos criar um novo sistema que adiciona um novo `Segment` em posição específica e retorna uma `Entity` id para adicionarmos no recurso `Segments`. Este sistema que cria um novo segmento é bastante semelhante ao segmento que instancia a cobra em si, `spawn_system`, sua maior diferença é o fato de que passamos uma posição, `Position`, como argumento para o segmento e que o tamanho do quadrado é menor, com coloração diferente.


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

Com este novo sistema, podemos adicionar um segmento extra no nosso `snake::spawn_system`, que armazenará os segmentos cabeça e o recem criado como entidades em `Segments`:

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

Anteriormente já falamos que os segmentos da cobra estão armazenados em um `Vec` e podemos iterar sobre pares destes elementos utilizando a função `windows`, agora nos falta entender como criar um teste para verificar se um segmento assume a posição de seu antecessor ao andar, mantendo a direção de movimento. Assim, nosso novo teste considera que a cobra é inicializada na posição `x = 3` e `y = 3`, e seu segmento extra em `x = 3` e `y = 2`. A primeira parte do teste define as novas posições da cabeça, `Head`, e do segment, `Segment`, com `let new_position_head_right = Position { x: 4, y: 3 };` e `let new_position_segment_right = Position { x: 3, y: 3 };`, ou seja, `Head` se move para `x = 4` e `y = 3` e segment para `x = 3` e `y = 3` ao apertarmos a tecla `D` e executarmos um novo update:

```rs
#[test]
fn snake_segment_has_followed_head() {
    // Setup
    let mut app = App::new();
    let new_position_head_right = Position { x: 4, y: 3 };
    let new_position_segment_right = Position { x: 3, y: 3 };

    // Adiciona os systemas
    app.insert_resource(Segments::default())
        .add_startup_system(spawn_system)
        .add_system(movement_system)
        .add_system(movement_input_system.before(movement_system));

    // adiciona resource apertando a tecla D, movimento para direita
    let mut input = Input::<KeyCode>::default();
    input.press(KeyCode::D);
    app.insert_resource(input);

    // executa sistemas
    app.update();

    let mut query = app.world.query::<(&Head, &Position)>();
    query.iter(&app.world).for_each(|(head, position)| {
        // garante que nova posição da cabeçá é esperada:
        assert_eq!(&new_position_head_right, position); 
        // garante que nova direção é para direita:
        assert_eq!(head.direction, Direction::Right);
    });

    let mut query = app.world.query::<(&Segment, &Position, Without<Head>)>();
    query.iter(&app.world).for_each(|(_segment, position, _)| {
        // garante que nova posição do segmento é esperada:
        assert_eq!(&new_position_segment_right, position);
    });
    // ...
}
```

Por descargo de consciência podemos adicionar mais uma parte ao teste que muda a direção de movimento da cobra para cima:

```rs
#[test]
fn snake_segment_has_followed_head() {
    // Setup
    let mut app = App::new();
    let new_position_head_right = Position { x: 4, y: 3 };
    let new_position_segment_right = Position { x: 3, y: 3 };

    // Adiciona os systemas
    app.insert_resource(Segments::default())
        .add_startup_system(spawn_system)
        .add_system(movement_system)
        .add_system(movement_input_system.before(movement_system));

    // adiciona resource apertando a tecla D, movimento para direita
    let mut input = Input::<KeyCode>::default();
    input.press(KeyCode::D);
    app.insert_resource(input);

    // executa sistemas
    app.update();

    let mut query = app.world.query::<(&Head, &Position)>();
    query.iter(&app.world).for_each(|(head, position)| {
        // garante que nova posição da cabeça é esperada:
        assert_eq!(&new_position_head_right, position); 
        // garante que nova direção é para direita:
        assert_eq!(head.direction, Direction::Right);
    });

    let mut query = app.world.query::<(&Segment, &Position, Without<Head>)>();
    query.iter(&app.world).for_each(|(_segment, position, _)| {
        // garante que nova posição do segmento é esperada:
        assert_eq!(&new_position_segment_right, position);
    });

    // NOVAS POSIÇÕES ESPERADAS
    let new_position_head_up = Position { x: 4, y: 4 }; // <--
    let new_position_segment_up = Position { x: 4, y: 3 }; // <--

    // adiciona resource apertando a tecla W, movimento para cima
    let mut input = Input::<KeyCode>::default();
    input.press(KeyCode::W); // <--
    app.insert_resource(input);

    // executa sistemas de novo
    app.update();

    let mut query = app.world.query::<(&Head, &Position)>();
    query.iter(&app.world).for_each(|(head, position)| {
        // garante que nova posição da cabeça é esperada:
        assert_eq!(&new_position_head_up, position);
        // garante que nova direção da cabeça é esperada:
        assert_eq!(head.direction, Direction::Up); 
    });

    let mut query = app.world.query::<(&Segment, &Position, Without<Head>)>();
    query.iter(&app.world).for_each(|(_segment, position, _)| {
        // garante que nova posição do segmento é esperada:
        assert_eq!(&new_position_segment_up, position);
    })
}
```

Ao exercutarmos este código veremos que o assert `assert_eq!(&new_position_segment_right, position);` falha indicando que o segmento não se moveu, mas a cabeça se moveu. Como estamos falando de movimento, sabemos que precisamos alterar o sistema `movement_system` para incluir uma iteração sobre os elementos de `Segments`. Assim a primeira alteração é adicionar o recurso `segments` nos argumentos do sistema, `segments: ResMut<Segments>`. Depois disso vamos precisar extrair duas queries diferentes para posição, `Position`, e cabeça, `Head`, já que agora nem todas posições terão cabeças, mas todas posições terão `Segment`, fazemos isso com as queries `mut heads: Query<(Entity, &Head)>` e `mut positions: Query<(Entity, &Segment, &mut Position)>`. Precisamos de `Segment`, pois `Food` também possui `Position` e queremos evitar iterar por elementos desnecessarios em um ECS.

O código que vamos adicionar ao `movement_system` é simples, basicamente iteramos por `segments` dois a dois elementos por vez e adicionamos a posição do elementos anterior, `entity[0]`, na posição do elemento posterior, `entity[1]`. Algo como:

```rs
(*segments).windows(2).for_each(|entity| {
    if let Ok((_, _segment, mut position)) = positions.get_mut(entity[1]) {
        if let Ok((_, _, mut new_position)) = positions.get_mut(entity[0]) {
            *position = new_position.clone();
        }
    };
});
```

Infelizmente, esse código em particular não compila devido ao fato de acessarmos `positions` como referência mutável duas vezes, inverter uma referência imutável e outra mutável também não funcionaria devida a uma regra do borrow checker do Rust que impede que um mesmo valor seja acesso como mutável e imutável no mesmo bloco. O mesmo acontece com duas referências diferentes mutáveis no mesmo bloco, o compilador não teria garantia que ambas não mutaria o mesmo valor ao mesmo tempo. Uma solução para este problema é adicionar um clone de `positions`, mas podemos fazer este clone ser mais inteligente, transformando ele em um `HashMap` na qual a chave é a entidade e a position é o valor, para depois acessarmos este clone de `positions` de forma imutável:

```rs
pub fn movement_system(
    segments: ResMut<Segments>,
    mut heads: Query<(Entity, &Head)>,
    mut positions: Query<(Entity, &Segment, &mut Position)>,
) {
    // Criar um hashmap clonado de positions com `Entity => Position`
    let positions_clone: HashMap<Entity, Position> = positions
        .iter()
        .map(|(entity, _segment, position)| (entity, position.clone()))
        .collect();
    // Acessar a cabeça (única existente por hora)
    if let Some((id, head)) = heads.iter_mut().next() {
        // Iterar sobre segments 2 a 2
        (*segments).windows(2).for_each(|entity| {
            // Acessar a posição da `entity[1]` em positions
            if let Ok((_, _segment, mut position)) = positions.get_mut(entity[1]) {
                // Acessar a posição da `entity[0]` em positions_clone
                if let Some(new_position) = positions_clone.get(&entity[0]) {
                    // Substituir position por new_position
                    *position = new_position.clone();
                }
            };
        });

        // mesmo código de antes para mover a cabeça
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
        });
    }
}
```

Agora sim, nosso teste passa e se executarmos `cargo run` vemos que o segmento rosa sempre segue a cabeça. Próximo passo é entender como criar um sistema para expandir a cobra ao comer.

## Alimentando a cobra

Faz algum tempo que nossa cobra está rodeada de comida, mas não pode comer, por isso chegou a hora de elaborarmos um teste que vai garatir que nossa cobra possa comer e que ela vai crescer ao comer. Nosso teste vai partir de um setup um pouco mais complexo, pois agora precisamos registrar um evento de crescimento, `GrowthEvenet`, precisamos registrar a última posição do vetor de segmentos, `LastTailPosition`, o nosso antigo sistema de spawn de comida, `crate::food::spawn_system`, e um system set organizado o que acontece primeiro. Algo como o seguinte bloco:

```rs
app.insert_resource(Segments::default())
    .insert_resource(LastTailPosition::default())
    .add_event::<GrowthEvent>()
    .add_startup_system(spawn_system)
    .add_system(crate::food::spawn_system)
    .add_system_set(
        SystemSet::new()
            .with_system(movement_system)
            .with_system(eating_system.after(movement_system))
            .with_system(growth_system.after(eating_system))
    );
```

Depois disso teremos dois updates, primeiro update cria o contexto de mundo e nos permite verificar que uma comida foi spawnada e que a cobra possui dois segmentos:

```rs
app.update();

let mut query = app.world.query::<(&Segment, &Position)>();
assert_eq!(query.iter(&app.world).count(), 2);
let mut query = app.world.query::<(&Food, &Position)>();
assert_eq!(query.iter(&app.world).count(), 1);
```

Por último, executamos mais uma vez o update e verificamos se a cobra possui 3 segmentos agora:

```rs
app.update();

let mut query = app.world.query::<(&Segment, &Position)>();
assert_eq!(query.iter(&app.world).count(), 3);
```

Infelizmente, um teste somento com isso não nos garantiria sucesso, já que o `crate::food::spawn_system` gera uma posição aleatória dentro do grid, para isso precisamos modificar a função `crate::food::spawn_system` para termos controle  da posição que a comida vai surgir. Fazemos isso adicionado uma macro de compilação exclusiva de teste, `cfg!`:

```rs
// food.rs
#[allow(clippy::cast_possible_wrap)]
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
            x: if cfg!(test) { 3 } else { (random::<u16>() % GRID_WIDTH) as i16 }, // <--
            y: if cfg!(test) { 5 } else { (random::<u16>() % GRID_HEIGHT) as i16 }, // <--
        })
        .insert(Size::square(0.65));
}
```

Agora sim, nosso teste poderá fazer sentido ao compilar:

```rs
// snake.rs
#[test]
fn snake_grows_when_eating() {
    // Setup
    let mut app = App::new();

    // sistemas
    app.insert_resource(Segments::default())
        .insert_resource(LastTailPosition::default())
        .add_event::<GrowthEvent>()
        .add_startup_system(spawn_system)
        .add_system(crate::food::spawn_system)
        .add_system_set(
            SystemSet::new()
                .with_system(movement_system)
                .with_system(eating_system.after(movement_system))
                .with_system(growth_system.after(eating_system))
        );

    // update de configuração
    app.update();

    let mut query = app.world.query::<(&Segment, &Position)>();
    assert_eq!(query.iter(&app.world).count(), 2);
    let mut query = app.world.query::<(&Food, &Position)>();
    assert_eq!(query.iter(&app.world).count(), 1);

    // update de execução
    app.update();

    let mut query = app.world.query::<(&Segment, &Position)>();
    assert_eq!(query.iter(&app.world).count(), 3);
}
```

Pronto, agora podemos ir jantar, mas, infelizmente, nossa  cobra ainda não. Para isso, começamos com o `snake::eating_system`. O objetivo de `snake::eating_system` é simples, iterar sobre todas as entidades com componente comida, `Food`, e ver se a posição delas, `Position`, é igual a `Position` das entidades com `Head`. Caso, a firmação anterior seja verdade, removemos, `despawn`, a entidade `Food` e lançamos no sistema um evento para crescer a cobra, `GrowthEvent`, na sua última posição, `pub struct LastTailPosition(Option<Position>)`. Um detalhe importante, é que para publicar um evento, `EventWriter`, precisamos registrar esse evento no `App`, com `add_event::<GrowthEvent>()`, o mesmo vale para lermos os eventos, `EventReader`.

```rs
pub struct GrowthEvent;

#[derive(Default)]
pub struct LastTailPosition(Option<Position>);

pub fn eating_system(
    mut commands: Commands,
    mut growth_writer: EventWriter<GrowthEvent>,
    food_positions: Query<(Entity, &Position), With<Food>>,
    head_positions: Query<&Position, With<Head>>,
) {
    for head_pos in head_positions.iter() {
        for (ent, food_pos) in food_positions.iter() {
            if food_pos == head_pos {
                commands.entity(ent).despawn();
                growth_writer.send(GrowthEvent);
            }
        }
    }
}
```

Na função anterior, recebemos um `EventWriter` do tipo `GrowthEvent`, que vai publicar quaisquer eventos necessários, uma query com todas as entidades e posições de comidas, `Query<(Entity, &Position), With<Food>>` e uma query com a posição das cabeças, no caso, apenas uma, `Query<&Position, With<Head>>`. Iteramos por tudo e checamos se as posições são iguais para então removermos a entidade associada a comida e publicarmos um evento de crescimento. Próximo passo é lermos o evento com `EventReader<GrowthEvent>` e adicionarmos um clone da última posição do rabo, como um novo segmento, aos segmentos, `Segments`:

```rs
pub fn growth_system(
    commands: Commands,
    last_tail_position: Res<LastTailPosition>,
    mut segments: ResMut<Segments>,
    mut growth_reader: EventReader<GrowthEvent>,
) {
    if growth_reader.iter().next().is_some() {
        segments.push(spawn_segment_system(commands, last_tail_position.0.clone().unwrap()));
    }
}
```

Nosso teste está quase passando, precisamos adicionar a informação da última posição de segmentos quando nos movimentamos, senão `LastTailPosition`será sempre `None` e nossa cobra não crescerá. Para isso, adicionamos o recuso mutável, `ResMut`, ao sistema `movement_system` e no final dele, buscamos a última posição dos segmentos:

```rs
pub fn movement_system(
    segments: ResMut<Segments>,
    mut last_tail_position: ResMut<LastTailPosition>,
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
        // ...
        *last_tail_position = LastTailPosition(Some(positions_clone.get(segments.last().unwrap()).unwrap().clone())); // <--
    }
}
```

Aghora sim, nosso teste passa! Sabrmos que podemos utilizar `unwrap`, pois a cobra inicia seu movimento com 2 segmentos e sabemos que ambos os segmentos possuem `Position`, assim, `positions_clone.get` , também, nunca será `None`. Possivelmente precisaremos adicionar alguns `.insert_resource(LastTailPosition::default())`no setup dos testes.

No próximo capítulo vamos aprender um pouco mais sobre colisões.