# A Cabeça da Cobra
 
Para começar o jogo precisamos do primeiro componente, neste caso a cabeça da cobra, que definirá os próximos possíveis passos, assim como para onde os blocos seguintes se moverão. Este primeiro componente se chamará `SnakeHead` e será uma struct vazia com a trait `Component` associada a ela:
 
```rust
#[derive(Component)]
pub struct SnakeHead;
```
 
A função de `SnakeHead` é basicamente ser um marcador para as entidades do tipo snake, que nos permitirá filtrar as estas entidades quando formos fazer queries com os players. Muitos componentes não precisam de estados e podem funcionar apenas como marcadores, um padrão bastante comum no mundo ECS, já que optamos por uma estratégia de *has a* (possui um) em vez de *is a* (é um, da orientação a objetos). Outro detalhe importante é a adição de uma cor específica para a cabeça da cobra `const SNAKE_HEAD_COLOR: Color = Color::rgb(0.7, 0.7, 0.7);`.
 
Nosso próximo passo é gerar uma entidade snake, que possui um componente do tipo `SnakeHead`, e essa entidade pode ser gerada adicionando um sistema inicial com `add_startup_system(spawn_snake)`, dada a função `spawn_snake`:
 
```rust
use bevy::prelude::*;
 
const SNAKE_HEAD_COLOR: Color = Color::rgb(0.7, 0.7, 0.7);
 
fn main() {
   App::new()
       .add_startup_system(setup_camera)
       .add_startup_system(spawn_snake)
       .add_plugins(DefaultPlugins)
       .run();
}
 
fn setup_camera(mut commands: Commands) {
   commands.spawn_bundle(OrthographicCameraBundle::new_2d());
}
 
#[derive(Component)]
pub struct SnakeHead;
 
fn spawn_snake(mut commands: Commands) {
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
       .insert(SnakeHead);
}
```
 
> **SpriteBundle**
>
> `SpriteBundle` é um tipo de componente que agrega características comuns a uma entidade que utiliza sprites como o próprio sprite (especificidades da imagem), transform (relação de posição, escala e rotação), visibilidade, transform global e o manuseio de imagens.
 
Neste caso, não temos nenhuma imagem específica como sprite, mas definimos um transform com uma escala de `10 x 10 x 10` pixels e uma cor de filtro acinzentada para a região definida pelo transform, as outras propriedades foram definidas como `..default()`. Ao executarmos `cargo run` o resultado é algo como:
 
![Entidade snake com `SpriteBundle`](../imagens/snake_pixel.png)
 
## Nosso primeiro teste
 
No mundo moderno, jogos sem testes estão fadados ao fracasso. Não estou dizendo que todos os jogos possuem uma bateria maravilhosa de testes automatizados, mas desde que escrevi o livro **Lean Game Development** até hoje, o mercado de games AAA mudou muito. Hoje em dia vejo jogos sendo desenvolvidos com TDD e com QA advogando por testes automatizados de gameplay emt todos os sistemas, garantindo uma jogabilidade equilibrada/desejada em qualquer plataforma. Hoje em dia um jogo, middleware, game server ou ferramenta sem nenhum teste esta fadado ao fracasso por conta do número excessivo de bugs e clientes infelizes. Sendo assim, é importante ter uma noção de como testar minimamente seus sistemas com a Bevy. Sendo assim, vamos aprender a escrever o teste mais simples possível, verificar se nosso sistema `spawn_snake` de fato adiciona um componente `SnakeHead` à entidade desejada.
 
Primeiro passo do teste será mover tudo que é relacionado a `snake` para um módulo chamado `snake.rs`:
 
**`main.rs`**:
```rust
use bevy::prelude::*;
 
mod snake;
 
use snake::spawn_snake;
 
fn main() {
   App::new()
       .add_startup_system(setup_camera)
       .add_startup_system(spawn_snake)
       .add_plugins(DefaultPlugins)
       .run();
}
 
fn setup_camera(mut commands: Commands) {
   commands.spawn_bundle(OrthographicCameraBundle::new_2d());
}
```
 
**`snake.rs`**:
```rust
use bevy::prelude::*;
 
const SNAKE_HEAD_COLOR: Color = Color::rgb(0.7, 0.7, 0.7);
 
#[derive(Component)]
pub struct SnakeHead;
 
pub fn spawn_snake(mut commands: Commands) {
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
       .insert(SnakeHead);
}
```
 
Agora em Snake vamos criar um teste dentro de um módulo de testes (`#[cfg(test)] mod test {...}`) que verifique se um componente `SnakeHead` está presente:
 
```rust
#[cfg(test)]
mod test {
   use super::*;
 
   #[test]
   fn entity_has_snake_head() {
       // 1 Inicialização do App
       let mut app = App::new();
 
       // 2 Adicionar o `spawn_snake` startup system
       app.add_startup_system(spawn_snake);
 
       // 3 Executar todos os sistemas pelo menos uma vez
       app.update();
 
       // 4 Fazer uma query por entidades que contenham o componente `SnakeHead`
       let mut query = app.world.query_filtered::<Entity, With<SnakeHead>>();
 
       // 5 Verificar se a contagem de componentes da query foi igual a 1
       assert_eq!(query.iter(&app.world).count(), 1);
   }
}
```
 
Descrevendo o teste `entity_has_snake_head` (verifica se entidade possui componente snake head) temos como primeiro passo (`1`) criar um `App` mutável para podermos adicionar sistemas como o `spawn_snake` (`2`) e executarmos todos os sistemas pelo menos uma vez com `app.update()` (`3`). O próximo passo é realizarmos uma `query` (`4`) no sistema de ECS para procurarmos por uma entidade que possua o componente `SnakeHead` (`With<SnakeHead>`). Com o resultado desta `query` verificamos se a quantidade de entidades que possuem o componente `SnakeHead` é igual a `1` (`5`).
 
## Queries
 
O principal objetivo de queries é nos permitir acessar componentes de entidades. No código a seguir, temos uma query do tipo `Query<(&Health, &mut Transform, Option<&Player>)>` que representa todas as entidades que possuam `Health` e `Transform`, com a propriedade `Health` sendo apenas leitura e a propriedade `Transform` sendo mutável. Além disso, caso o componente `Player` esteja presente, permite a leitura dele. Depois disso iteramos sobre todos os ítens dessa query, de forma mutável, para podermos alterar a propriedade transform, `(health, mut transform, player) in query.iter_mut()`. Por último, caso o componente `Player` esteja presente, sabemos que esta entidade é do tipo player e aplicamos uma lógica extra.
 
```rust
fn check_zero_health(
   mut query: Query<(&Health, &mut Transform, Option<&Player>)>,
) {
   // Obtem todas as entidades do tipo
   for (health, mut transform, player) in query.iter_mut() {
       eprintln!("Entity at {} has {} HP.", transform.translation, health.hp);
 
       // centraliza se `hp` é menor ou igual a `0.0`
       if health.hp <= 0.0 {
           transform.translation = Vec3::ZERO;
       }
 
       if let Some(player) = player {
           // entidade é do tipo `Player`
           // lógica extra
       }
   }
}
```
 
para obter o ID de uma entidade com queries basta adicionar `Entity` a query e a variável `entity_id` corresponderá ao id:
 
```rust
// adicione `Entity` a `Query` para obter os IDs
fn query_entities(q: Query<(Entity, /* ... */)>) {
   for (entity_id, /* ... */) in q.iter() {
       // `entity_id` é o ID da entidade que estamos acessando.
   }
}
```
 
Caso exista certeza que uma query vai identificar apenas uma entidade, é possível utilizar `single` e `single_mut` para acessar seus componentes:
 
```rust
fn query_player(mut q: Query<(&Player, &mut Transform)>) {
   let (player, mut transform) = q.single_mut();
   // lógica
}
```
 
Outro recurso interessante de queries são os *Query Filters*, um tipo especial de queries que permite reduzir a quantidade de entidade que uma query retorna. *Query filters* se utilizam dos filtros `With` e `Without` para garantir que a entidade tenha (`With`) ou não tenha (`Without`) certos componentes. No exemplo a seguir, a query acessa todas as entidades com o componente `Health` que sejam  `Players` amigáveis e que opcionalmente possuam `PlayerName`
 
```rust
fn debug_player_hp(
   query: Query<(&Health, Option<&PlayerName>), (With<Player>, Without<Enemy>)>,
) {
   for (health, name) in query.iter() {
       // ...
   }
}
```
 
> **Utilizando filtros**
>
> * Elementos adicionados em uma Tupla, como `(With<Player>, Without<Enemy>)`, são considerados `AND`/`E` lógicos.
> * Para utilizar `OR`/`OU` lógicos é preciso envolver as tuplas em um filtro do tipo `Or<(…)>`.
 
## Movendo a cabeça da cobra
 
Não existe o Snake game sem movimento, então o próximo passo é controlarmos os movimentos da cabeça da cobra com as teclas `WASD` ou direcionais. Para isso, podemos começar com a movimentação para cima utilizando o teste:
 
```rust
#[test]
fn snake_head_has_moved_up() {
   // Setup
   let mut app = App::new();
   let default_transform = Transform {..default()};
 
   // Adicionando sistemas
   app.add_startup_system(spawn_snake)
   .add_system(snake_movement);
 
   // Adicionando inputs de `KeyCode`s
   let mut input = Input::<KeyCode>::default();
   input.press(KeyCode::W);
   app.insert_resource(input);
 
   // Executando sistemas pelo menos uma vez
   app.update();
 
   // Query para obter entidades com `SnakeHead` e `Transform`
   let mut query = app.world.query::<(&SnakeHead, &Transform)>();
 
   // Verificando se o valor de Y no `Transform` mudou
   query.iter(&app.world).for_each(|(_head, transform)| {
       assert!(default_transform.translation.y < transform.translation.y);
       assert_eq!(default_transform.translation.x, transform.translation.x);
   })
}
```
 
Neste teste adicionamos um `Transform` com valores padrão de `translation` para comparar quando o transform da query mudar, adicionamos um novo sistema de movimento `add_system(snake_movement)` e criamos um recurso que gerencia inputs de teclado `Input::<KeyCode>::default()`, na qual setamos seu evento `press` como `KeyCode::W`. Para resolver este teste precisamos criar o sistema `snake_movement`, que é bastante trivial neste caso, apenas um sistema que busca por um query contendo `&SnakeHead` e `&Transform`, depois modifica o valor de Y de forma que sempre aumente:
 
```rust
// snake.rs
pub fn snake_movement(mut head_positions: Query<(&SnakeHead, &mut Transform)>) {
   for (_head, mut transform) in head_positions.iter_mut() {
       transform.translation.y += 1.;
   }
}
 
// main.rs
// ...
mod snake;
use snake::{spawn_snake, snake_movement};
 
fn main() {
   App::new()
       .add_startup_system(setup_camera)
       .add_startup_system(spawn_snake)
       .add_plugins(DefaultPlugins)
       .add_system(snake_movement)
       .run();
}
// ...
```
 
### Controlando a direção de movimento
 
Nosso movimento atual está longe de ser realista ou funcional, para isso precisamos que a cobra se movimente com base nas teclas `wasd` e podemos começar com um teste que move a cobra 1 unidade para cima, verificando que apenas o `y` mudou em relacao ao original, depois uma unidade para direita, verificando que apenas o `x` mudou em relação ao anterior. Por último, um novo teste movendo para baixo e para esquerda, verificando se as posições são inferiores às originais em `x` e `y`. Assim, o primeiro teste fica:
 
```rust
#[test]
fn snake_head_moves_up_and_right() {
   // Setup
   let mut app = App::new();
   let default_transform = Transform {..default()};
 
   // Adiciona systemas
   app.add_startup_system(spawn_snake)
   .add_system(snake_movement);
 
   // Testa movimento para cima
   let mut up_transform = Transform {..default()};
   let mut input = Input::<KeyCode>::default();
   input.press(KeyCode::W);
   app.insert_resource(input);
   app.update();
   let mut query = app.world.query::<(&SnakeHead, &Transform)>();
   query.iter(&app.world).for_each(|(_head, transform)| {
       assert!(default_transform.translation.y < transform.translation.y);
       assert_eq!(default_transform.translation.x, transform.translation.x);
       up_transform = transform.to_owned();
   });
 
   // Testa movimento para direita
   let mut input = Input::<KeyCode>::default();
   input.press(KeyCode::D);
   app.insert_resource(input);
   app.update();
   let mut query = app.world.query::<(&SnakeHead, &Transform)>();
   query.iter(&app.world).for_each(|(_head, transform)| {
       assert_eq!(up_transform.translation.y , transform.translation.y);
       assert!(up_transform.translation.x < transform.translation.x);
   })
}
```
 
Ao executarmos este teste percebemos que a linha `assert_eq!(up_transform.translation.y , transform.translation.y);` falha pois nosso `transform.translation.y` está maior que o anterior, que faz sentido, já que nosso sistema de movimento está apenas aumentando o `y` a cada update. Para resolvermos isso, podemos adicionar os comandos para se mover com `w` e com `d`:
 
```rust
// snake.rs
pub fn snake_movement(
   keyboard_input: Res<Input<KeyCode>>,
   mut head_positions: Query<(&SnakeHead, &mut Transform)>
) {
   for (_head, mut transform) in head_positions.iter_mut() {
       if keyboard_input.pressed(KeyCode::D) {
           transform.translation.x += 1.;
       }
       if keyboard_input.pressed(KeyCode::W) {
           transform.translation.y += 1.;
       }
   }
}
```
 
Teste passando, então podemos fazer o segundo teste, movimento para baixo e para esquerda. O teste é basicamente igual ao anterior, mas reduzimos algumas linhas:
 
```rust
#[test]
fn snake_head_moves_down_and_left() {
   // Setup
   let mut app = App::new();
   let default_transform = Transform {..default()};
 
   app.add_startup_system(spawn_snake)
   .add_system(snake_movement);
 
   // Movimenta para baixo
   let mut input = Input::<KeyCode>::default();
   input.press(KeyCode::S);
   app.insert_resource(input);
   app.update();
 
 
   // Movimenta para esquerda
   let mut input = Input::<KeyCode>::default();
   input.press(KeyCode::A);
   app.insert_resource(input);
   app.update();
 
   // Assert
   let mut query = app.world.query::<(&SnakeHead, &Transform)>();
   query.iter(&app.world).for_each(|(_head, transform)| {
       assert!(default_transform.translation.y > transform.translation.y);
       assert!(default_transform.translation.x > transform.translation.x);
   })
}
```
 
Como esperado, o teste falha e podemos implementar as condições que faltam de pressionar o teclado, `s` e `a`:
 
```rust
pub fn snake_movement(
   keyboard_input: Res<Input<KeyCode>>,
   mut head_positions: Query<(&SnakeHead, &mut Transform)>
) {
   for (_head, mut transform) in head_positions.iter_mut() {
       if keyboard_input.pressed(KeyCode::D) {
           transform.translation.x += 1.;
       }
       if keyboard_input.pressed(KeyCode::W) {
           transform.translation.y += 1.;
       }
       if keyboard_input.pressed(KeyCode::A) {
           transform.translation.x -= 1.;
       }
       if keyboard_input.pressed(KeyCode::S) {
           transform.translation.y -= 1.;
       }
   }
}
```
 
Tudo passa e podemos ir para o próximo passo, explicar e melhorar este código. O argumento `keyboard_input` é um recurso que contém os eventos relacionados a tecla que foi pressionada no `input`, ou seja, `Res<Input<KeyCode>>,`. Nossa query faz sentido e está funcional, porém, como não estamos utilizando o componente `SnakeHead`, representado por `_head`, podemos mudar nossa query para `Query<&mut Transform, With<SnakeHead>>`, que altera nosso código para utilizar apenas o transform como variável:
 
```rust
pub fn snake_movement(
   keyboard_input: Res<Input<KeyCode>>,
   mut head_positions: Query<&mut Transform, With<SnakeHead>>
) {
   for mut transform in head_positions.iter_mut() {
       if keyboard_input.pressed(KeyCode::D) {
           transform.translation.x += 1.;
       }
       if keyboard_input.pressed(KeyCode::W) {
           transform.translation.y += 1.;
       }
       if keyboard_input.pressed(KeyCode::A) {
           transform.translation.x -= 1.;
       }
       if keyboard_input.pressed(KeyCode::S) {
           transform.translation.y -= 1.;
       }
   }
}
```
 
Como mencionamos antes sobre o `With`, ele nos permite buscar todas as entidades que possuam o componente `SnakeHead`, mas explícita para a Bevy que não nos importamos com o conteúdo de `SnakeHead`, apenas com o `Transform`. Isso é importante pois quanto menos componentes o sistema precisar acessar, mais a bevy conseguirá paralelizar as coisas.
 
## CI
 
Uma coisa bastante importante enquanto desenvolvemos é termos um sistema de integração contínua executando. No caso do Rust no Github eu recomendo utilizar o *Github Actions* e minha configuração base para projetos Rust é:
 
```yaml
name: Rust
 
on:
 push:
   branches: [ "main" ]
 pull_request:
   branches: [ "*" ]
 
env:
 CARGO_TERM_COLOR: always
 
jobs:
 build:
   runs-on: ubuntu-latest
 
   steps:
   - uses: actions/checkout@v3
   - name: Install alsa and udev
     run: sudo apt-get update; sudo apt-get install --no-install-recommends libasound2-dev libudev-dev libwayland-dev libxkbcommon-dev
   - name: Build
     run: cargo build --release --verbose
    
 test:
   runs-on: ubuntu-latest
 
   steps:
   - uses: actions/checkout@v2
   - name: Install alsa and udev
     run: sudo apt-get update; sudo apt-get install --no-install-recommends libasound2-dev libudev-dev libwayland-dev libxkbcommon-dev
   - name: tests
     run: cargo test -- --nocapture
  fmt:
   runs-on: ubuntu-latest
 
   steps:
   - uses: actions/checkout@v2
   - name: FMT
     run: cargo fmt -- --check
 
 clippy:
   runs-on: ubuntu-latest
 
   steps:
   - uses: actions/checkout@v2
   - name: Install alsa and udev
     run: sudo apt-get update; sudo apt-get install --no-install-recommends libasound2-dev libudev-dev libwayland-dev libxkbcommon-dev
   - name: install-clippy
     run: rustup component add clippy
   - name: clippy
     run: cargo clippy -- -W clippy::pedantic --deny "warnings"
 
```
 
Ao executarmos o CI, percebemos que a formatação não estava correta, que pode ser corrigida com `cargo fmt`, e há algumas sugestões de linting em relação a nomenclatura das funções e structs no módulo e declaração de argumentos. A questão de nomenclatura solicita que funções e structs não comecem com o nome do módulo. A declaração de argumentos solicita que o tipo de `keyboard_input` seja passado como referência `keyboard_input: &Res<Input<KeyCode>>`, porém isso quebra a injeção de recursos da bevy, necessitando assim que o lint seja descartado com `#[allow(clippy::needless_pass_by_value)]`. Meu único problema com a questão de nomenclatura é perder o contexto de que os sistemas e as structs quando utilizamos importações absolutas em vez de qualificadas. A solução é utilizar importações qualificadas. O código ficou assim:
 
```rust
// Snake.rs
use bevy::prelude::*;
 
const SNAKE_HEAD_COLOR: Color = Color::rgb(0.7, 0.7, 0.7);
 
#[derive(Component)]
pub struct Head;
 
pub fn spawn_system(mut commands: Commands) {
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
       .insert(Head);
}
 
#[allow(clippy::needless_pass_by_value)]
pub fn movement_system(
   keyboard_input: Res<Input<KeyCode>>,
   mut head_positions: Query<&mut Transform, With<Head>>,
) {
   for mut transform in head_positions.iter_mut() {
       if keyboard_input.pressed(KeyCode::D) {
           transform.translation.x += 1.;
       }
       if keyboard_input.pressed(KeyCode::W) {
           transform.translation.y += 1.;
       }
       if keyboard_input.pressed(KeyCode::A) {
           transform.translation.x -= 1.;
       }
       if keyboard_input.pressed(KeyCode::S) {
           transform.translation.y -= 1.;
       }
   }
}
 
#[cfg(test)]
mod test {
   use super::*;
 
   #[test]
   fn entity_has_snake_head() {
       // Setup app
       let mut app = App::new();
 
       // Add startup system
       app.add_startup_system(spawn_system);
 
       // Run systems
       app.update();
 
       let mut query = app.world.query_filtered::<Entity, With<Head>>();
       assert_eq!(query.iter(&app.world).count(), 1);
   }
 
   #[test]
   fn snake_head_has_moved_up() {
       // Setup
       let mut app = App::new();
       let default_transform = Transform { ..default() };
 
       // Add systems
       app.add_startup_system(spawn_system)
           .add_system(movement_system);
 
       // Add input resource
       let mut input = Input::<KeyCode>::default();
       input.press(KeyCode::W);
       app.insert_resource(input);
 
       // Run systems
       app.update();
 
       let mut query = app.world.query::<(&Head, &Transform)>();
       query.iter(&app.world).for_each(|(_head, transform)| {
           assert!(default_transform.translation.y < transform.translation.y);
           assert_eq!(default_transform.translation.x, transform.translation.x);
       })
   }
 
   #[test]
   fn snake_head_moves_up_and_right() {
       // Setup
       let mut app = App::new();
       let default_transform = Transform { ..default() };
 
       // Add systems
       app.add_startup_system(spawn_system)
           .add_system(movement_system);
 
       // Move Up
       let mut up_transform = Transform { ..default() };
       let mut input = Input::<KeyCode>::default();
       input.press(KeyCode::W);
       app.insert_resource(input);
       app.update();
       let mut query = app.world.query::<(&Head, &Transform)>();
       query.iter(&app.world).for_each(|(_head, transform)| {
           assert!(default_transform.translation.y < transform.translation.y);
           assert_eq!(default_transform.translation.x, transform.translation.x);
           up_transform = transform.to_owned();
       });
 
       // Move Right
       let mut input = Input::<KeyCode>::default();
       input.press(KeyCode::D);
       app.insert_resource(input);
       app.update();
       let mut query = app.world.query::<(&Head, &Transform)>();
       query.iter(&app.world).for_each(|(_head, transform)| {
           assert_eq!(up_transform.translation.y, transform.translation.y);
           assert!(up_transform.translation.x < transform.translation.x);
       })
   }
 
   #[test]
   fn snake_head_moves_down_and_left() {
       // Setup
       let mut app = App::new();
       let default_transform = Transform { ..default() };
 
       // Add systems
       app.add_startup_system(spawn_system)
           .add_system(movement_system);
 
       // Move down
       let mut input = Input::<KeyCode>::default();
       input.press(KeyCode::S);
       app.insert_resource(input);
       app.update();
 
       // Move Left
       let mut input = Input::<KeyCode>::default();
       input.press(KeyCode::A);
       app.insert_resource(input);
       app.update();
 
       // Assert
       let mut query = app.world.query::<(&Head, &Transform)>();
       query.iter(&app.world).for_each(|(_head, transform)| {
           assert!(default_transform.translation.y > transform.translation.y);
           assert!(default_transform.translation.x > transform.translation.x);
       })
   }
}
 
// Main.rs
use bevy::prelude::*;
 
mod snake;
 
fn main() {
   App::new()
       .add_startup_system(setup_camera)
       .add_startup_system(snake::spawn_system)
       .add_plugins(DefaultPlugins)
       .add_system(snake::movement_system)
       .run();
}
 
fn setup_camera(mut commands: Commands) {
   commands.spawn_bundle(OrthographicCameraBundle::new_2d());
}
 
```
 
A seguir vamos criar o conceito de Grid.
 