# Multiplayer Local

**CORRIGIR ACENTOS EM TECLADO COMPATIVEL**
Primeiro passo para entendermos um jogo multiplayer é criarmos as regras para o jogo executar corretamente no modo multiplayer, podemos fazer isso adicionando mais um player de forma local. O suporte ao multiplayer exige uma pequena refatoração, que sera por onde comecaremos.

## Refatorando

Com a atualização do Rust para versão `1.66`, o linter do Rust sugeriu algumas novas refatorações bem simples, mas muito bem observadas no modulo `grid`. A primeira é na função `translate_position` e na função `scale_sprite` que possuiam um casting desnecessário, podendo-se remover o `as f32` das chamadas de funções `window.width()` e `window.height()`:

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

A segunda refatoração é, no mesmo módulo, a simplificação da função `convert` para utilizar a função `mul_add` em vez de uma multiplicação seguida por uma adição. A vantagem do uso de `mul_add` é que reduz os erros por arrendondamento que poderiam ser causados pelo uso de `f32`:

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

Outra refatoração que fiz foi para melhorar os resultados de testes em modo `release`  e em Windows, utilizando a biblioteca `approx = "0.5.1"`. Esta biblioteca garante uma comparação mais correta de epsilons de f32. Assim, podemos mudar todos os testes que contém comaprações de f32 para utilizar o `assert_relative_eq`, que recebe um epsilon de valor de erro na comparação:

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

Por último, vamos atualizar os testes com múltiplas chamadas de `app.update()`. Fazemos isso substituíndo todas as linhas por um `for` com range:
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