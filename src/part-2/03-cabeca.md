# A Cabaça da Cobra

Para começar o jogo precisamos do primeiro componente, neste caso a cabeça da cobra, que definirá os próximos possíveis passos, assim como para onde os blocos seguintes se moverão. Este primeiro componente se chamará `SnakeHead` e será uma struct vazia com a trait `Compoent` associada a ela:

```rust
#[derive(Component)]
pub struct SnakeHead;
```

A função de `SnakeHead` é basicamente ser um marcador para as entidades do tipo snake, que nos permitirá filtrar as estas entidades quando formos fazer queries com os players. Muitos componentes não precisam de estados e podem funcionar apenas como marcadores, um padrão bastante comum no mundo ECS, já que optamos por uma estratégia de *has a* (possui um) em vez de *is a* (é um, da orientação a objetos). Outro detalhe importante é a adição de uma cor ao específica para a cabeça da cobra `const SNAKE_HEAD_COLOR: Color = Color::rgb(0.7, 0.7, 0.7);`.

Nosso próximo passo é gerar uma entidade snake, que possui um componente do tipo `SnakeHead`, e essa entidade pode ser gerada adicionando um sistema incial com `add_startup_system(spawn_snake)`, dada a função `spawn_snake`:

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
> `SpriteBundle` é um tipo de componente que agrega características comuns a uma entidade que utiliza sprites como o proprio sprite (especificidades da imagem), transform (relação de posição, escala e rotação), visibilidade, transform global e o manuseio de imagens. 

Neste caso, não temos nenhuma imagem específica como sprite, mas definimos um transform com uma escala de `10 x 10 x 10` pixels e uma cor de filtro acinzentada para a região definida pelo transform, as outras propriedades foram definidas como `..default()`. Ao executarmos `cargo run` o resultado é algo como:

![Entidade snake com `SpriteBundle`](../imagens/snake_pixel.png)

## Nosso primeiro teste

No mundo moderno, jogos sem testes estão fadados ao fracasso. Não estou dizendo que todos os jogos possuem uma bateria maravilhosa de testes automatizados, mas desde que escrevi o livro **Lean Game Development** até hoje, o mercado de games AAA mudou muito. Hoje em dia vejo jogos sendo desenvolvidos com TDD e com QA advogando por testes automatizados de gameplay emt todos os sistemas, garantindo uma jogabilidade equilibrada/desejada em qualquer plataforma. Hoje em dia um jogo, middleware, game server ou ferramenta sem nenhum teste esta fadado ao fracasso por conta do número excessivo de bugs e clientes infelizes. Sendo assim, é importante ter uma noção de como testar minimamente seus sistemas com a Bevy. Sendo assim, vamos aprender a escrever o teste mais simples possível, verificar se nosso sistema `spawn_snake` de fato adiciona um compoente `SnakeHead` a entidade desejada.

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

Descrevendo o teste `entity_has_snake_head` (verifica se entidade possui componente snake head) temos como primeiro passo (`1`) criar um `App` mutável para podermos adicionar sistemas como o `spawn_snake` (`2`) e executarmos todos os sistemas pelo menos uma vez com `app.update()` (`3`). Próximo passo é realizarmos uma `query` (`4`) no sistema de ECS para procurarmos por uma entidade que possua o componente `SnakeHead` (`With<SnakeHead>`). Com o resultado desta `query` verificamos se a quantidade de entidades que possuam o componente `SnakeHead` é igual a `1` (`5`).

## Queries

O principal objetivo de queries é nos permitir acessar componentes de entidades. No código a seguir, temos uma query do tipo `Query<(&Health, &mut Transform, Option<&Player>)>` que representa todas as entidades que possuam `Health` e `Transform`, com a propriedade `Health` sendo apenas leitura e a propriedade `Transform` sendo mutável. Além disso, caso o componente `Player` esteja presente, permite leitura a ele. Depois disso iteramos sobre todos os ítens dessa query, de forma mutável, para podermos alterar a propriedade transform, `(health, mut transform, player) in query.iter_mut()`. Por último, caso o componente `Player` esteja presente, sabemos que está entidade é do tipo player e aplicamos uma lógica extra.

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

para obter o ID de uma entidade com queries basta adicionar `Entity` a query e a variável `entity_id` correspodnerá ao id:

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


