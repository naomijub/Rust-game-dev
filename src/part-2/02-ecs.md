# Entity Component System (ECS)

O sistema de gerenciamento de dados da Bevy é chamado de *Entity Component System*, ou **ECS**, e sua principal característica é a simplicidade do gerenciamento de dados. Uma boa analogia ao seu funcionamento é com bancos de dados tabulares, na qual os componentes, *components*, são os tipos de dados, ou colunas, e as entidades, *entities*, são as linhas, mais especificamente o ID das linhas. Por exemplo, você poderia ter diversas entidades com o componente `Health` e cada entidade possui um component `Health` diferente, já que NPCs, players e objetos do mundo podem ter `Health`s diferentes (*health* significa vida em inglês). Assim, o conjunto de componentes que uma entidade possui é chamado de arquétipo, *Archetype*.

Considerando a entidade player possuindo componentes como vida, força, ataque, defesa, inventario, as entidades inimigos com vida, força, ataque, defesa, inteligência, e a entidade planta com apenas vida, fica muito fácil escrever uma lógica de jogo que gerencia esses tipos de entidades, como verificar se uma entidade com vida encontrou outra entidade com vida, simplificando muito a criação de lógicas de jogo. Essa lógica de gerenciamento é chamado sistema, *system*. Estes sistemas são executados em paralelo pelo *smart scheduling algorithm* da Bevy e com isso devemos manter nossas entidades o mais horizontal possível, evitando grandes componentes com muitos campos. Isso influência muito a performance do sistema, pois quando mais vertical a entidade mais problemas de acesso aos dados em paralelo teremos.

> Para você que vem da orientação a objectos, no paradigma de ECS é mais comum possuir uma entidade com diversos componentes, como a entidade Player que possui os componentes Vida(u32), Posição(x, y, z), Direção(x, y, z), Escala(x, y, z), Rotação(x, y, z), Defesa(u16), Ataque(u16), Força(u16) em vez de uma classe `Player` com os campos vida: u32, posição: [x, y, z], direção: [x, y, z], escala: [x, y, z], rotação: [x, y, z], defesa: u16, ataque: u16, força: u16:

```rust
// Prefira isso:
// Entidade Player;

#[derive(Component)]
pub struct Vida(u32)

#[derive(Component)]
pub struct Posição(x, y, z)

#[derive(Component)]
pub struct Direção(x, y, z)

#[derive(Component)]
pub struct Escala(x, y, z)

#[derive(Component)]
pub struct Rotação(x, y, z)

#[derive(Component)]
pub struct Defesa(u16)

#[derive(Component)]
pub struct Ataque(u16)

#[derive(Component)]
pub struct Força(u16)

// Em vez disso:
pub struct Player {
    pub vida: u32, 
    pub posição: [x, y, z], 
    pub direção: [x, y, z], 
    pub escala: [x, y, z], 
    pub rotação: [x, y, z], 
    pub defesa: u16, 
    pub ataque: u16, 
    pub força: u16,
}

```

## Criando entidades

Entidades são simplesmente IDs inteiros associados a um comando `spawn` de `commands`, `commands.spawn(...)` e para adicionar componentes basta utilizarmos a diretica `insert` em um `spawn`:

```rust
fn spawn_entity(mut commands: Commands) {
    commands
        .spawn()
        .insert(Label("Player"))
        .insert(Vida(10))
        .insert(Posição(0, 2, 0))
        .insert(Direção(0, 2, 0))
        .insert(Escala(0, 2, 0))
        .insert(Rotação(0, 2, 0))
        .insert(Defesa(10))
        .insert(Ataque(10))
        .insert(Força(10));
}
```

Além disso, existe o conceito de *bundles*. *Bundles* são como *templates* que tornam a criação de entidades com diversos componentes mais simples:

```rust
#[derive(Bundle)]
struct Transform {
    posição: Posição(x, y, z),
    direção: Direção(x, y, z),
    escala: Escala(x, y, z),
    rotação: Rotação(x, y, z),
}

#[derive(Bundle)]
struct Player {
    vida: u32, 
    defesa: u16, 
    ataque: u16, 
    força: u16,

    #[bundle] // Nested bundles
    transform: Transform
}
```

Como podemos ver em `transform: Transform`, bundles também podem ser encadeados. Tuplas arbitrárias também são consideradas bundles. Note, que bundles não podem ser consultados com uma *query*.

## Recursos (*Resources*)

Recursos são um tipo de instância que permite armazenar um tipo de dado de forma global, independente de entidades, e qualquer tipo Rust pode ser usado como um recurso independente de implementação de traits. 
Para criar um novo recurso você pode simplesmente usar uma `Struct` ou um `enum` e derivar a trait `Resource`.
```rust
#[derive(Resource)]
struct GoalsReached {
    main_goal: bool,
    bonus: u32,
}
```
Bevy usa resources para muitas coisas. Você pode usa-la para acessar várias features da engine.

### Acessando

Para acessar o valor de um recurso de um systema use `Res/ResMut`:
```rust
fn my_system(
    // irá lançar um panic se o resources não existir
    mut goals: ResMut<GoalsReached>,
    other: Res<MyOtherResource>,
    // use Option se o resource caso ele possa não existir
    mut fancy: Option<ResMut<MyFancyResource>>,
) {
    if let Some(fancy) = &mut fancy {
        // código
    }
}
```

### Gerenciando

Se você precisa criar/remover `resources` em tempo de execução, você pode usar um `commands` (Commands):

```rust
fn my_setup(mut commands: Commands, /* ... */) {
    // adicionar (ou sobreescrever) resource, usando os parametros enviados
    commands.insert_resource(GoalsReached { main_goal: false, bonus: 100 });
    // garente que o resource exista (criando se necessário)
    commands.init_resource::<MyFancyResource>();
    // remove um resource (se ele existir)
    commands.remove_resource::<MyOtherResource>();
}
```

Alternatively, using direct World access from an exclusive system:
Alternativamente, usando diretamente `World` de um `system` exclusivo

```rust
fn my_setup2(world: &mut World) {
    // Os mesmos metodos e comandos estão disponíveis aqui,
    // mas nós podemos também fazer coisas mais sofisticadas

    // verifica se o recurso existe
    if !world.contains_resource::<MyFancyResource>() {
        // Recebe acesso para o recuros, inserindo um valor customizado caso não tenha passado nenhum
        let _bonus = world.get_resource_or_insert_with(
            || GoalsReached { main_goal: false, bonus: 100 }
        ).bonus;
    }
}
```

`Resources` pode também ser configurado no `app buildeer`. Faça isso para recursos que precisam sempre existam quando inicia o app.

```rust
App::new()
    .add_plugins(DefaultPlugins)
    .insert_resource(StartingLevel(3))
    .init_resource::<MyFancyResource>()
    // código
```

### Inicialização

Existem duas formas de inicializar recursos, a primeira é definindo a trait `Default` para eles, quando eles possuem um tipo de dado simples, já a segunda é implementando a trait `FromWorld` que permite atuar sobre o recurso utilizando valores de `World`:

Uma inicialização usando `Default`
```rust
// configurando todos os campos com seu valor default
#[derive(Resource, Default)]
struct GameProgress {
    game_completed: bool,
    secrets_unlocked: u32,
}

#[derive(Resource)]
struct StartingLevel(usize);

//  implementação para valores customizados
impl Default for StartingLevel {
    fn default() -> Self {
        StartingLevel(1)
    }
}

// para enums, você pode espeficar o valor default dele
#[derive(Resource, Default)]
enum GameMode {
    Tutorial,
    #[default]
    Singleplayer,
    Multiplayer,
}
```

Para `resources` que precisam de uma inicialização um pouco mais complexa, podemos implementar o `FromWorld`
```rust
#[derive(Resource)]
struct MyFancyResource { /* stuff */ }

impl FromWorld for MyFancyResource {
    fn from_world(world: &mut World) -> Self {
        // Você tem controle total de tudo do seu ECS World aqui
        // Por exemplo, você pode aceesar (e modificar!) outros resources
        let mut x = world.resource_mut::<MyOtherResource>();
        x.do_mut_stuff();

        MyFancyResource { /* stuff */ }
    }
}
```

A decisão de quando usar recursos ou entity/component é baseada na forma e no momento em que este dado vai ser acessado, mas considerando algo como um jogo com uma unica entidade, pode ainda ser útil utilizar o padrão ECS, pois ele permite maior flexibildiade e compartilhamento de dados, que podem ser muito úteis para a evolução do jogo.

## Sistemas (*Systems*)

Sistemas são funções que a desenvolvedora escreve com o objetivo de ser uma unidade de lógica do jogo atuando sobre as entidades e os componentes. Os sistemas são executados e gerenciados pelas Bevy, mas somente podem ser usados com parâmetros especiais. Os parâmetros especiais são:
* `Res/ResMut` para acessar recursos.
* `Query` para acessar componentes de uma entidade.
* `Commands` para criar e destruir entidades, componentes e recursos.
* `EventWriter/EventReader` para enviar e receber eventos.

Um sistema pode conter no máximo 16 parâmetros, caso seja preciso mais parâmetros pode se agrega-los em tuplas de no máximo 16 parâmetros. Caso estes limites não sejam suficiente, é possível fazer tuplas de tuplas.

```rust
fn complex_system(
    (a, mut b): (Res<ResourceA>, ResMut<ResourceB>),
    (q0, q1, q2): (Query<(/* … */)>, Query<(/* … */)>, Query<(/* … */)>),
) {
    // lógica
}
```

No sistema a cima `ResourceA` é um recurso imutável e esta compartilhando uma tupla com `ResourceB`que é um recurso mutável. Já `q0, q1 e q2` são componentes de uma entidade.

Existem dois tipos de funções para executar sistemas na Bevy

```rust
fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        // sistemas executados apenas quando o App é lançado
        .add_systems(Startup, (setup_camera, debug_start))
        // sistemas executados todos os frames
        .add_systems(Update, (move_player, enemies_ai))
        // ...
        .run();
}
```

Agora vamos começar a implementar nosso snake game e aprofundar nossos conhecimentos em bevy.

**Referência: [unofficial bevy guide](https://bevy-cheatbook.github.io/programming.html)**
