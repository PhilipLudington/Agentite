# Agentite Design Philosophy

> A game engine built for minds that think in code—both artificial and human.

## Vision

Agentite is a 2D game engine written in Klar with a singular focus: **making game development predictable, discoverable, and verifiable**. It serves two audiences:

1. **AI Agents** building games through code generation
2. **The Game Development Benchmark** testing AI capabilities with novel technology

These audiences share a need: they must understand unfamiliar systems quickly and use them correctly without extensive trial-and-error.

## Core Principles

### 1. Predictability Over Flexibility

> "The best API is one where the right answer is obvious."

- **One way to do things.** When there are multiple approaches, choose one and document why.
- **No magic.** Every behavior must be traceable through explicit code paths.
- **Fail fast, fail loud.** Invalid states should be impossible to construct; when errors occur, they should be caught immediately with actionable messages.

### 2. Discoverability Through Structure

> "Code should teach you how to use it."

- **Consistent naming conventions.** If you know one API, you can guess the others.
- **Self-documenting patterns.** The type system and function signatures reveal intent.
- **Hierarchical organization.** Related functionality lives together; unrelated functionality is clearly separated.

### 3. Data-Oriented Design

> "Let the data flow."

Agentite uses an **Entity Component System (ECS)** architecture:

- **Entities** are lightweight identifiers (integers)
- **Components** are plain data structs with no behavior
- **Systems** are functions that transform component data

This separation enables:
- Cache-friendly memory layouts for performance
- Easy serialization and inspection of game state
- Clear boundaries for testing and reasoning

### 4. AI-First Ergonomics

> "Optimize for the reader who has never seen this code before."

**For AI Code Generation:**
- Declarative APIs where games are described as data
- Builder patterns that chain naturally and self-document
- Comprehensive examples for every feature
- Error messages that include the fix, not just the problem

**For AI Benchmarking:**
- Novel patterns that cannot be memorized from training data
- Thorough documentation that rewards reading comprehension
- Consistent conventions that reward pattern recognition
- Clear success criteria for any given task

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                      Game Loop                              │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐  │
│  │  Input  │ -> │  Update │ -> │ Render  │ -> │  Audio  │  │
│  │ Systems │    │ Systems │    │ Systems │    │ Systems │  │
│  └─────────┘    └─────────┘    └─────────┘    └─────────┘  │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                        World                                │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                   Entity Storage                      │  │
│  │  Entity 0: [Position, Sprite, Velocity]              │  │
│  │  Entity 1: [Position, Sprite, Player, Health]        │  │
│  │  Entity 2: [Position, Sprite, Enemy, AI]             │  │
│  └──────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                    Resources                          │  │
│  │  [Time, Input, Assets, Camera, ...]                  │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

## Module Structure

```
agentite/
├── core/           # ECS, scheduling, app lifecycle
├── math/           # Vectors, matrices, transforms
├── input/          # Keyboard, mouse, gamepad
├── graphics/       # Sprites, rendering, camera
├── audio/          # Sound effects, music
├── physics/        # Collision detection, movement
├── ui/             # Immediate-mode UI
└── assets/         # Resource loading and management
```

## API Design Guidelines

### Naming Conventions

| Concept | Convention | Example |
|---------|------------|---------|
| Components | PascalCase noun | `Position`, `Velocity`, `Health` |
| Systems | snake_case verb phrase | `apply_velocity`, `check_collisions` |
| Resources | PascalCase noun | `Time`, `Input`, `Assets` |
| Events | PascalCase past tense | `CollisionOccurred`, `EntitySpawned` |
| Builders | Type + `Builder` | `SpriteBuilder`, `EntityBuilder` |

### Pattern: Query-Based Systems

Systems declare what data they need; the ECS provides it:

```klar
// System declares its data requirements explicitly
system apply_velocity(query: Query<Position, Velocity>, time: Resource<Time>) {
    for (position, velocity) in query {
        position.x += velocity.x * time.delta
        position.y += velocity.y * time.delta
    }
}
```

### Pattern: Builder APIs

Complex objects are constructed through chainable builders:

```klar
let player = world.spawn()
    .with(Position { x: 100.0, y: 100.0 })
    .with(Velocity { x: 0.0, y: 0.0 })
    .with(Sprite::from_asset("player.png"))
    .with(Player { lives: 3 })
    .with(Collider::rectangle(32.0, 32.0))
    .build()
```

### Pattern: Declarative Game Definition

Games can be defined as data structures:

```klar
let game = Game::new()
    .title("My Game")
    .window(800, 600)
    .add_plugin(GraphicsPlugin)
    .add_plugin(InputPlugin)
    .add_plugin(PhysicsPlugin)
    .add_system(Stage::Update, player_movement)
    .add_system(Stage::Update, enemy_ai)
    .add_system(Stage::PostUpdate, check_collisions)
    .run()
```

## Error Philosophy

Every error message follows this template:

```
ERROR: [What went wrong]

  [Where it happened with context]

BECAUSE: [Why this is an error]

FIX: [Exactly what to do]

EXAMPLE:
  [Working code that does it correctly]
```

Example:

```
ERROR: Component `Health` not found on entity 42

  at: check_player_death system
  query: Query<Player, Health>

BECAUSE: Entity 42 has components [Player, Position, Sprite] but
         the system requires both Player AND Health components.

FIX: Either add Health when spawning the player:
     world.spawn().with(Player).with(Health { current: 100, max: 100 })

     Or make Health optional in your query:
     Query<Player, Option<Health>>

EXAMPLE:
  // Spawning with required components
  let player = world.spawn()
      .with(Player { ... })
      .with(Health { current: 100, max: 100 })
      .build()
```

## Documentation Standards

Every public API must include:

1. **Purpose** - One sentence explaining what it does
2. **Parameters** - Type and meaning of each parameter
3. **Returns** - What the function returns and when
4. **Example** - Working code demonstrating typical usage
5. **Errors** - What can go wrong and how to handle it
6. **See Also** - Related APIs for discoverability

## Performance Commitments

- **60 FPS** with 10,000 simple sprites
- **Sub-millisecond** system scheduling overhead
- **Zero allocations** in the hot path after initialization
- **Predictable frame times** - no GC pauses or hidden costs

## Scope Boundaries

### In Scope
- 2D sprite rendering and animation
- Tile-based maps
- 2D physics and collision detection
- Audio playback (effects and music)
- Input handling (keyboard, mouse, gamepad)
- Immediate-mode UI for menus/HUD
- Asset loading (images, sounds, data files)
- Scene management

### Out of Scope (v1.0)
- 3D rendering
- Networking/multiplayer
- Visual scripting
- Level editor
- Platform-specific features

## Success Metrics

For **AI Agent Development:**
- An AI can implement a simple game (Pong, Breakout) with only documentation
- Common errors are caught at compile time, not runtime
- The happy path requires no backtracking or retries

For **Benchmark Testing:**
- Tasks have clear, verifiable success criteria
- Documentation provides all information needed to complete tasks
- Novel patterns prevent benchmark gaming through memorization

---

*Agentite: Where artificial minds build playable worlds.*
