# Agentite

A 2D game engine written in [Klar](https://github.com/mrphil/Klar), designed for AI agents and the Game Development Benchmark.

## Vision

Agentite is built for minds that think in code—both artificial and human. It prioritizes:

- **Predictability** — One obvious way to do things, no magic
- **Discoverability** — Consistent patterns, self-documenting APIs
- **Performance** — Data-oriented ECS architecture, zero allocations in hot paths
- **AI Ergonomics** — Clear error messages, comprehensive documentation, novel but learnable patterns

## Status

🚧 **Early Development** — Building core systems

| Component | Status |
|-----------|--------|
| Math Library | Not started |
| ECS Core | Not started |
| Input System | Not started |
| Graphics | Not started |
| Physics | Not started |
| Audio | Not started |
| SDL3 Bindings | Blocked (waiting on Klar FFI) |

## Example (Target API)

```klar
import agentite.prelude.*

fn main() -> i32 {
    let game: App = App.new()
        .title("My Game")
        .window(800, 600)
        .add_plugin(DefaultPlugins)
        .add_startup_system(setup)
        .add_system(Stage.Update, player_movement)
        .build()

    game.run()
    return 0
}

fn setup(world: inout World) {
    // Spawn player
    world.spawn()
        .with(Position { x: 400.0, y: 300.0 })
        .with(Sprite.from_asset("player.png"))
        .with(Player { speed: 200.0 })
        .build()
}

fn player_movement(
    input: Resource[Input],
    time: Resource[Time],
    query: Query[(Player, inout Position)]
) {
    for (player, position) in query {
        if input.pressed(KeyCode.Right) {
            position.x += player.speed * time.delta
        }
        if input.pressed(KeyCode.Left) {
            position.x -= player.speed * time.delta
        }
    }
}
```

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Game Loop                              │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐  │
│  │  Input  │ -> │  Update │ -> │ Render  │ -> │  Audio  │  │
│  └─────────┘    └─────────┘    └─────────┘    └─────────┘  │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                     ECS World                               │
│  Entities: lightweight IDs                                  │
│  Components: plain data structs                             │
│  Systems: functions that transform data                     │
│  Resources: global singletons (Time, Input, Assets)         │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                   SDL3 Platform Layer                       │
│  Window • GPU Rendering • Audio • Input Events              │
└─────────────────────────────────────────────────────────────┘
```

## Documentation

- [Design Philosophy](DESIGN.md) — Core principles and API guidelines
- [Development Plan](PLAN.md) — Phased implementation roadmap

## Requirements

- [Klar](https://github.com/mrphil/Klar) compiler (Phase 4+)
- SDL3 (for platform bindings, when FFI is available)
- macOS or Linux (Windows support pending Klar)

## License

MIT License — see [LICENSE](LICENSE)

## Contributing

Agentite is developed alongside the Klar language. Contributions welcome once core APIs stabilize.

---

*Built for the [Game Development Benchmark](https://github.com/mrphil/GameDevBenchmark) — testing AI's ability to learn novel technologies.*
