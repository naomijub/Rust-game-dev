# Melhorando a Cadência do Movimento

O atual movimento da cobra está ligado aos comandos do teclado diferentemente do snake game que a cobra se movimenta independente dos comandos do teclado e a cada x segundos, em vez de a cada frame. Para podermos fazer com que a cobra se movimente independente dos comandos do teclado, precisamos de uma forma de armazenar a direção que ela está se movimentando, além de evitar que a cobra vá para a direção oposta. Assim, precisamos criar o enum que armazena a direção e criar uma função que indica a direção oposta.

```rs
// components.rs
#[test]
fn opposite_direction() {
    assert_eq!(Direction::Up.opposite(), Direction::Down);
    assert_eq!(Direction::Down.opposite(), Direction::Up);
    assert_eq!(Direction::Right.opposite(), Direction::Left);
    assert_eq!(Direction::Left.opposite(), Direction::Right);
}

// ...
#[derive(PartialEq, Debug, Copy, Clone)]
pub enum Direction {
    Left,
    Up,
    Right,
    Down,
}

impl Direction {
    pub fn opposite(self) -> Self {
        match self {
            Self::Left => Self::Right,
            Self::Right => Self::Left,
            Self::Up => Self::Down,
            Self::Down => Self::Up,
        }
    }
}
```

Em uma etapa inicial do desenvolvimento, eu adicionaria Direction como um componente na entidade cobra, porém, a medida que a cobra ficar maior, vai ser difícil sincronizar a direção de cada elemento. Sabendo disso, vamos fazer um pouco de overengineering, e colocar `Direction` como elemento do componente `snake::Head`. Além disso, definimos a direção padrão como `Direction::Up` utilizando a trait `Default`:

```rs
// snake.rs

#[derive(Component)]
pub struct Head {
    direction: Direction
}

impl Default for Head {
    fn default() -> Self {
        Self { direction: Direction::Up }
    }
}

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
        .insert(Head::default() ) // <--
        .insert(Position { x: 3, y: 3 })
        .insert(Size::square(0.8));
}
```

## Separando o movimento em duas etapas

Agora que nossa cobra possui uma direção armazenada na cabeça podemos mudar seu sistema de movimento para que seja executado a cada 0.15 segundos, fazemos isso da mesma forma que fizemos com o sistema de geração de comidas:

```rs
fn main() {
    App::new()
        .add_systems(Startup, setup_camera)
        .add_systems(Startup, snake::spawn_system)
        .insert_resource(ClearColor(Color::rgb(0.04, 0.04, 0.04)))
        .add_plugins(
            DefaultPlugins
                .set(WindowPlugin {
                    primary_window: Some(Window {
                        resolution: (500.0, 500.0).into(),
                        title: "Snake".into(),
                        resizable: false,
                        ..default()
                    }),
                    ..default()
                })
                .build(),
        )
        .add_systems(PostUpdate, (grid::position_translation, grid::size_scaling))
        .add_systems(Update, snake::movement_system)
        .add_systems(
            Update,
            food::spawn_system.run_if(on_timer(Duration::from_secs_f32(1.0))),
        )
        .add_systems(
            Update,
            snake::movement_system.run_if(on_timer(Duration::from_secs_f32(0.150))),
        )
        .add_systems(PostUpdate, (grid::position_translation, grid::size_scaling))
        .run();
}
```

Nosso sistema de movimento está atrelado à translação da cobra em uma posição para cada tecla que apertarmos e agora sabemos que o sistema de movimento deve ser independente do sistema de direção, que vamos chamar de `snake::movement_input_system`. Então precisamos que o sistema de input/direção aconteça antes do sistema de movimento, e, ainda, precisamos garantir que o sistema de movimento aconteça a cada `0.15` segundos. Para solucionar este problema, vamos adicionar uma função especial para sistemas `before`. Para utilizar está função, precisamos adicionar o sistema de input que vamos criar e indicar que ele deve ocorrer antes do sistema de movimento com `.add_systems(Update, snake::movement_input_system.before(snake::movement_system)`, adicione ao `App::new()` na função main. Agora vamos para o sistema `movement_input_system`.

Primeira coisa que devemos fazer é alterar os testes para considerar que o `snake::movement_system` recebe uma query com `Position` e `snake::Head`, além de considerar que o movimento é feito com base no enum `Direction` contido dentro de `snake::Head`:


```rs
// snake.rs
    #[test]
    fn snake_starts_moviment_up() { // <-- novo teste
        // Setup app
        let mut app = App::new();

        // Add startup system
        app.add_systems(Startup, spawn_system);


        // Run systems
        app.update();

        let mut query = app.world.query::<&Head>();
        let head = query.iter(&app.world).next().unwrap();
        assert_eq!(head.direction, Direction::Up);
    }

    #[test]
        fn snake_head_has_moved_up() {
        // Setup
        let mut app = App::new();
        let default_position = Position { x: 5, y: 6 };

        // Adicionando sistemas
        app.add_systems(Startup, spawn_system)
            .add_systems(Update, movement_system)
            .add_systems(Update, movement_input_system.before(movement_system));


        // Adicionando inputs de `KeyCode`s
        let mut input = Input::<KeyCode>::default();
        input.press(KeyCode::W);
        app.insert_resource(input);

        // Executando sistemas pelo menos uma vez
        app.update();

        //Assert
        let mut query = app.world.query::<(&Head, &Position)>();
        query.iter(&app.world).for_each(|(head, position)| {
            assert_eq!(&default_position, position);
            assert_eq!(head.direction, Direction::Up); // <-- novo assert

        })
    }

    #[test]
    fn snake_head_moves_up_and_right() {
        // Setup
        let mut app = App::new();
        let up_position = Position { x: 5, y: 6 };

        // Adiciona systemas
        app.add_systems(Startup, spawn_system)
            .add_systems(Update, movement_system)
            .add_systems(Update, movement_input_system.before(movement_system));

        // Testa movimento para cima
        let mut input = Input::<KeyCode>::default();
        input.press(KeyCode::W);
        app.insert_resource(input);
        app.update();

        let mut query = app.world.query::<(&Head, &Position)>();
        query.iter(&app.world).for_each(|(head, position)| {
            assert_eq!(position, &up_position);
            assert_eq!(head.direction, Direction::Up); // <- Novo assert
        });

        let up_right_position = Position { x: 6, y: 6 };

        // Testa movimento para direita
        let mut input = Input::<KeyCode>::default();
        input.press(KeyCode::D);
        app.insert_resource(input);
        app.update();

        let mut query = app.world.query::<(&Head, &Position)>();
        query.iter(&app.world).for_each(|(head, position)| {
            assert_eq!(&up_right_position, position);
            assert_eq!(head.direction, Direction::Right); // <- Novo assert

        })
    }


    #[test]
    fn snake_head_moves_down_and_left() {
        // Setup
        let mut app = App::new();
        let down_left_position = Position { x: 4, y: 6 };

        // Add systems
        app.add_startup_system(spawn_system)
            .add_system(movement_system)
            .add_system(movement_input_system.before(movement_system)); // <--

        // Move Left
        let mut input = Input::<KeyCode>::default();
        input.press(KeyCode::A);
        app.insert_resource(input);
        app.update();

         // Move down
         let mut input = Input::<KeyCode>::default();
         input.press(KeyCode::S);
         app.insert_resource(input);
         app.update();

        // Assert
        let mut query = app.world.query::<(&Head, &Position)>();
        query.iter(&app.world).for_each(|(head, position)| {
            assert_eq!(&down_left_position, position);
            assert_eq!(head.direction, Direction::Left);
        })
    }
    #[test]
    fn snake_cannot_start_moving_down() { // <-- novo teste
         // Setup
         let mut app = App::new();
         let down_left_position = Position { x: 5, y: 6 };
 
         // Add systems
         app.add_systems(Startup, spawn_system)
            .add_systems(Update, movement_system)
 r          .add_systems(Update, movement_input_system.before(movement_system));
 
          // Move down
          let mut input = Input::<KeyCode>::default();
          input.press(KeyCode::S);
          app.insert_resource(input);
          app.update();
 
         // Assert
         let mut query = app.world.query::<(&Head, &Position)>();
         query.iter(&app.world).for_each(|(_head, position)| {
             assert_eq!(&down_left_position, position);
         })
    }

```

> Novos testes: `snake_starts_moviment_up` que checa se a cobra inicia seu movimento para cima. `snake_cannot_start_moving_down` checa que não é possível se movimentar na direção oposta.

Com estes testes sabemos que o movement system agora deve receber uma query com a posição mutável e com a `Head` para obtermos a direção. Com base na direção, movemos a posição da cabeça:

```rs
pub fn movement_system(mut heads: Query<(&mut Position, &Head)>) {
    if let Some((mut pos, head)) = heads.iter_mut().next() {
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
    }
}
```

Já o `movement_input_system` recebe uma leitura de teclado (`KeyCode`) e muda a direção com base nesta leitura do teclado e evita mudanças na direção oposta:

```rs
pub fn movement_input_system(
    keyboard_input: Res<Input<KeyCode>>, 
    mut heads: Query<&mut Head>) {
    if let Some(mut head) = heads.iter_mut().next() {
        let dir: Direction = if keyboard_input.pressed(KeyCode::A) {
            Direction::Left
        } else if keyboard_input.pressed(KeyCode::S) {
            Direction::Down
        } else if keyboard_input.pressed(KeyCode::W) {
            Direction::Up
        } else if keyboard_input.pressed(KeyCode::D) {
            Direction::Right
        } else {
            head.direction
        };
        if dir != head.direction.opposite() {
            head.direction = dir;
        }
    }
}
```

Ao executarmos `cargo run` podemos ver a cobra se movimentando sozinha e obedecendo o sistema de direção. Próximo passo é adicionarmos o rabo.
