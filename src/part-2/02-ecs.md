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