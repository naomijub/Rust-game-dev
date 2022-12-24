# Sobre a Bevy

Bevy engine é uma das game engines mais promissoras do mercado e um grande esforço coletivo para a comunidade rust_gamedev. Se trata de uma engine orientada a dados, gratuíta e open source, sob as licenças Apache e MIT, ou seja, perfeita para qualquer projeto. Ela possui como objetivos de design:

* Um conjunto completo de features para jogos 2D e 3D, podendo inclusive ser aplicada para outros objetivos.
* Simples e poderosa, mas mantendo o fácil aprendizado.
* Orientada a dados utilizando o paradigma ECS (Entity component system, no próximo capítulo).
* Modular, use o que quiser, adicione o que quiser, e substitua o que quiser.
* Rápida, paralela e em Rust <3.
* Compilação rápida

A atual versão da Bevy é [![Crates.io](https://img.shields.io/crates/v/bevy.svg)](https://crates.io/crates/bevy) e este livro foi desenvolvido com a versão `0.7`, mas contém guias de migração para as versões `0.8` e `0.9`. A compatibilidade com Rust esta garantida para a versão `1.66`.

## Iniciando o projeto

> Para iniciar um projeto com a Bevy é necessário possuir Rust e Cargo, caso você não possua basta fazer download em https://rustup.rs/.

Vamos iniciar nosso projeto com um simples `cargo new bevy-snake --bin`, que gera um projeto executável em Rust chamado `bevy-snake`. Este projeto vai possuir um `Cargo.toml` (onde os metadados do projeto estão localizados), um `src/main.rs` e um `.gitignore`:

```rust
// src/main.rs
fn main() {
    println!("Hello, world!");
}
```

```sh
# .gitignore 
/target
```

Agora adicionamos versão atual da bevy (`bevy = "0.7"`) a seção `[dependencies]` do Cargo.toml. Adicionamos também a crate de aleatoriedade `rand`:

```toml
[dependencies]
bevy = "0.7"
rand = "0.7"
```

Com essas mudanças no `Cargo.toml` podemos começar a usar o `prelude` da bevy e criar nosso primeiro app com:

```rust
use bevy::prelude::*;

fn main() {
    App::new().run();
}
```

### Instanciando uma Janela

Instanciar uma janela com a Bevy é bastante trivial e pode ser feito através do uso de plugins, neste caso o `DefaultPlugins` contém um conjunto básico de plugins que tornam a bevy operacional:

```rust
fn main() {
    App::new().add_plugins(DefaultPlugins).run();
}
```

Agora se executarmos `cargo run` veremos uma janela com fundo cinza. Por padrão, os plugins da Bevy não incluem camera, pois o uso de camera é muito variado em jogos, assim, precisamos criar nosso próprio sistema de cameras. Usaremos uma camera ortográfica 2D com o commando `OrthographicCameraBundle::new_2d()` em uma função que fará a configuração do sistema de cameras inicial alterando a variável do tipo `mut Commands`. `Commands` é um tipo muito comum ao escrever sistemas com a Bevy e é usado para enfileirar comandos com o objetivo de modificar o mundo (que chamaremos de `world`) e os recursos (que chamaremos de `resources`). Assim, na função a seguir, `setup_camera`, receberemos como argumento `mut commands: Commands` e utilizaremos ele para instanciar (chamado de `spawn`) uma nova entidade bundle com os componentes de uma câmera 2D ortográfica:

```rust
fn setup_camera(mut commands: Commands) {
    commands.spawn_bundle(OrthographicCameraBundle::new_2d());
}
```

E agora basta adicionar esse função ao nosso `App` através de um `add_startup_system`:

```rust
fn main() {
    App::new()
        .add_startup_system(setup_camera)
        .add_plugins(DefaultPlugins)
        .run();
}

fn setup_camera(mut commands: Commands) {
    commands.spawn_bundle(OrthographicCameraBundle::new_2d());
}
```

## Plugins

A Bevy é pensada de forma que todas suas partes sejam modularizáveis, assim, todas as core features da engine são implementadas como plugins que podem ser substituídos, evoluídos e customizados, além disso, os próprios jogos são encarados como plugins. Assim, se você não precisar de uma UI, basta não registrar o sistema de UI, quer um sistema de UI diferente, registre o seu próprio. Para o caso de servidores, basta não registrar o plugin `RenderPlugin`.
 
Caso você não precise de uma experiência tão avançada com a Bevy, é possível utilizar o `DefaultPlugins` que utilizamos anteriormente, que possui sistemas como Rendering, gerenciamento de assets, sistema de UI, janelas e gerenciamento de entrada de dados.

### Criando um Plugin

Para criar um plugin simplesmente precisamos implementar a trait `Plugin` em um tipo que comporte as informações necessárias. No caso do plugin que vamos implementar é apenas um `hello world` para plugins, então não precisamos de dados, criando apenas um 

```rust
pub struct HelloPlugin;

impl Plugin for HelloPlugin {
    fn build(&self, app: &mut App) {
        // lógica do plugin
    }
}
```

Agora precisamos de uma função que nosso sistema vai executar, neste caso um simples `println`:

```rust
fn hello_plugin() {
    println!("hello plugin!");
}

```

E adicionamos essa função como um `startup_system` no nosso plugin:

```rust
impl Plugin for HelloPlugin {
    fn build(&self, app: &mut App) {
        app.add_startup_system(hello_plugin);
    }
}
```

Por último, basta adicionarmos nosso plugin ao `App` principal e executar `cargo run`:

```rust
fn main() {
    App::new()
        .add_startup_system(setup_camera)
        .add_plugin(HelloPlugin)
        .add_plugins(DefaultPlugins)
        .run();
}
```

Veremos algo no terminal como:

```
2022-06-20T05:28:52.725036Z  INFO bevy_render::renderer: AdapterInfo { name: "AMD Radeon Pro 5500M", vendor: 0, device: 0, device_type: DiscreteGpu, backend: Metal }
hello plugin!
```