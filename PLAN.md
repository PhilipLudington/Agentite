# Agentite Development Plan

> Building a 2D game engine in Klar, one layer at a time.

## Current Status

**Klar Language**: Phase 4 (~95% complete)
- ✅ Structs, enums, generics, traits, modules
- ✅ Ownership-based memory management
- ✅ Native LLVM compilation
- ✅ Result/Option error handling
- ✅ Collections (List, Map, Set)
- ⚠️ No C FFI yet (needed for graphics)
- ⚠️ No math stdlib (sin, cos, sqrt)
- ❌ No Windows support
- ❌ No async/await

**Agentite Engine**: Not started

## Development Philosophy

Given Klar's constraints, we'll build Agentite in **dependency order**:
1. Build what Klar can do natively first (math, ECS, data structures)
2. Define clean interfaces for external dependencies (graphics, audio)
3. Implement platform bindings when Klar's FFI is ready
4. Keep the core engine pure Klar for portability

## Phase 0: Foundation (Current)

**Goal**: Establish project structure and build system

### Tasks
- [ ] Create directory structure
- [ ] Set up build configuration
- [ ] Create test harness
- [ ] Write initial documentation templates

### Directory Structure
```
agentite/
├── src/
│   ├── core/           # ECS, app lifecycle, scheduling
│   ├── math/           # Vectors, matrices, transforms
│   ├── collections/    # Game-specific data structures
│   ├── input/          # Input abstraction layer
│   ├── graphics/       # Rendering abstraction
│   ├── audio/          # Audio abstraction
│   ├── physics/        # Collision and movement
│   ├── assets/         # Resource loading
│   └── ui/             # Immediate-mode UI
├── tests/              # Unit and integration tests
├── examples/           # Example games
├── docs/               # Documentation
└── bindings/           # Platform-specific C FFI (future)
```

### Deliverables
- [ ] `src/lib.kl` - Main library entry point
- [ ] `tests/run_tests.kl` - Test runner
- [ ] `examples/minimal.kl` - Minimal "hello engine" example

---

## Phase 1: Math Library

**Goal**: Build the mathematical foundation for 2D game development

**Why First**: Everything else depends on vectors and transforms. Pure Klar, no external dependencies.

### Types to Implement

```klar
// Core vector types
struct Vec2 { x: f32, y: f32 }
struct Vec3 { x: f32, y: f32, z: f32 }  // For 2.5D effects, colors
struct Vec4 { x: f32, y: f32, z: f32, w: f32 }  // Homogeneous coords

// Matrices (column-major for graphics compatibility)
struct Mat3 { cols: [Vec3; 3] }  // 2D transforms
struct Mat4 { cols: [Vec4; 4] }  // Future 3D support

// Transform components
struct Transform2D {
    position: Vec2
    rotation: f32      // Radians
    scale: Vec2
}

// Geometric primitives
struct Rect { x: f32, y: f32, width: f32, height: f32 }
struct Circle { center: Vec2, radius: f32 }
struct Line { start: Vec2, end: Vec2 }

// Color
struct Color { r: f32, g: f32, b: f32, a: f32 }
```

### Operations to Implement

**Vec2 Operations:**
- Arithmetic: add, sub, mul (scalar), div (scalar), neg
- Products: dot, cross (returns scalar for 2D)
- Geometric: length, length_squared, normalize, distance
- Interpolation: lerp, slerp (for angles)
- Utilities: min, max, clamp, abs, floor, ceil, round

**Mat3 Operations:**
- Construction: identity, translation, rotation, scale
- Multiplication: mat * mat, mat * vec
- Inverse, transpose, determinant

**Transform2D Operations:**
- to_matrix, from_matrix
- apply_to_point, apply_to_vector
- compose, inverse

**Math Functions (implement in Klar):**
- Trigonometry: sin, cos, tan, atan2 (Taylor series or lookup tables)
- Roots: sqrt (Newton-Raphson), rsqrt
- Utilities: abs, min, max, clamp, sign, floor, ceil, round
- Interpolation: lerp, smoothstep, inverse_lerp
- Angles: degrees_to_radians, radians_to_degrees, normalize_angle

### Traits

```klar
trait Numeric {
    fn zero() -> Self
    fn one() -> Self
}

trait Lerp {
    fn lerp(self: ref Self, other: ref Self, t: f32) -> Self
}

trait Approx {
    fn approx_eq(self: ref Self, other: ref Self, epsilon: f32) -> bool
}
```

### Testing Strategy
- Property-based tests (e.g., `(a + b) - b ≈ a`)
- Known value tests (sin(0) = 0, sin(π/2) ≈ 1)
- Edge cases (zero vectors, identity transforms)

### Deliverables
- [ ] `src/math/vec2.kl`
- [ ] `src/math/vec3.kl`
- [ ] `src/math/vec4.kl`
- [ ] `src/math/mat3.kl`
- [ ] `src/math/mat4.kl`
- [ ] `src/math/transform.kl`
- [ ] `src/math/rect.kl`
- [ ] `src/math/color.kl`
- [ ] `src/math/functions.kl` (sin, cos, sqrt, etc.)
- [ ] `src/math/mod.kl` (module re-exports)
- [ ] `tests/math_tests.kl`

---

## Phase 2: ECS Core

**Goal**: Implement the Entity Component System foundation

**Why Second**: The ECS is the architectural backbone. Pure Klar, builds on math types.

### Core Types

```klar
// Entity is just an ID
struct Entity {
    id: u64
    generation: u32  // For detecting stale references
}

// Component storage (sparse set based)
struct ComponentStorage[T] {
    dense: List[T]           // Actual component data
    sparse: List[?u32]       // Entity ID -> dense index
    entities: List[Entity]   // dense index -> Entity
}

// The world holds all state
struct World {
    entities: EntityAllocator
    // Component storages registered dynamically
    storages: Map[TypeId, dyn ComponentStorageTrait]
    resources: Map[TypeId, dyn Any]
}

// Query for iterating entities with specific components
struct Query[T] {
    world: ref World
    // ... iteration state
}
```

### Entity Management

```klar
impl World {
    fn new() -> World
    fn spawn() -> EntityBuilder
    fn despawn(entity: Entity) -> bool
    fn is_alive(entity: Entity) -> bool

    // Component operations
    fn add[T](entity: Entity, component: T)
    fn remove[T](entity: Entity) -> ?T
    fn get[T](entity: Entity) -> ?ref T
    fn get_mut[T](entity: Entity) -> ?inout T
    fn has[T](entity: Entity) -> bool
}
```

### Query System

```klar
// Query components (compile-time would be ideal, runtime for now)
fn query[T](world: ref World) -> Query[T]
fn query2[A, B](world: ref World) -> Query[(A, B)]
fn query3[A, B, C](world: ref World) -> Query[(A, B, C)]

// Query filters
fn with[T](query: Query[...]) -> Query[...]      // Must have T
fn without[T](query: Query[...]) -> Query[...]   // Must not have T
fn optional[T](query: Query[...]) -> Query[...]  // T becomes ?T
```

### Resources (Global Singletons)

```klar
impl World {
    fn insert_resource[T](resource: T)
    fn get_resource[T]() -> ?ref T
    fn get_resource_mut[T]() -> ?inout T
    fn remove_resource[T]() -> ?T
}
```

### Events

```klar
struct Events[T] {
    events: List[T]
    read_index: Map[ReaderId, usize]
}

impl Events[T] {
    fn send(event: T)
    fn read(reader: ReaderId) -> EventReader[T]
    fn clear()  // Called each frame
}
```

### Testing Strategy
- Entity lifecycle (spawn, despawn, generation reuse)
- Component CRUD operations
- Query iteration correctness
- Resource management
- Event delivery

### Deliverables
- [ ] `src/core/entity.kl`
- [ ] `src/core/component.kl`
- [ ] `src/core/storage.kl`
- [ ] `src/core/world.kl`
- [ ] `src/core/query.kl`
- [ ] `src/core/resource.kl`
- [ ] `src/core/events.kl`
- [ ] `src/core/mod.kl`
- [ ] `tests/ecs_tests.kl`

---

## Phase 3: Scheduling & App Lifecycle

**Goal**: Build the game loop and system scheduling

### App Structure

```klar
struct App {
    world: World
    schedule: Schedule
    plugins: List[dyn Plugin]
    running: bool
}

impl App {
    fn new() -> AppBuilder
    fn run(self: inout Self)  // Starts game loop
}

struct AppBuilder {
    fn title(name: string) -> Self
    fn window(width: u32, height: u32) -> Self
    fn add_plugin[P: Plugin](plugin: P) -> Self
    fn add_system(stage: Stage, system: fn(inout World)) -> Self
    fn add_startup_system(system: fn(inout World)) -> Self
    fn build() -> App
}
```

### Stages

```klar
enum Stage {
    First          // Before everything
    PreUpdate      // Input processing
    Update         // Main game logic
    PostUpdate     // Physics, collision response
    PreRender      // Prepare render data
    Render         // Actual rendering
    Last           // Cleanup, diagnostics
}
```

### System Scheduling

```klar
struct Schedule {
    stages: Map[Stage, List[System]]
}

struct System {
    name: string
    run: fn(inout World)
    // Future: dependencies for parallel execution
}

impl Schedule {
    fn run_stage(stage: Stage, world: inout World)
    fn run_all(world: inout World)
}
```

### Plugin System

```klar
trait Plugin {
    fn build(app: inout AppBuilder)
    fn name() -> string
}

// Example built-in plugins
struct TimePlugin {}
impl TimePlugin: Plugin {
    fn build(app: inout AppBuilder) {
        app.add_resource(Time::new())
        app.add_system(Stage::First, update_time)
    }
    fn name() -> string { return "TimePlugin" }
}
```

### Time Resource

```klar
struct Time {
    delta: f32           // Seconds since last frame
    elapsed: f32         // Total seconds since start
    frame_count: u64     // Number of frames
    fixed_delta: f32     // For fixed timestep (default 1/60)
}
```

### Deliverables
- [ ] `src/core/app.kl`
- [ ] `src/core/schedule.kl`
- [ ] `src/core/plugin.kl`
- [ ] `src/core/time.kl`
- [ ] `tests/app_tests.kl`

---

## Phase 4: Input System

**Goal**: Abstract input handling (keyboard, mouse, gamepad)

### Input Types

```klar
// Keyboard
enum KeyCode {
    A, B, C, /* ... */ Z,
    Num0, Num1, /* ... */ Num9,
    Space, Enter, Escape, Tab,
    Left, Right, Up, Down,
    LShift, RShift, LCtrl, RCtrl, LAlt, RAlt,
    F1, F2, /* ... */ F12,
    // ... more keys
}

enum KeyState {
    Released      // Not pressed
    JustPressed   // Pressed this frame
    Pressed       // Held down
    JustReleased  // Released this frame
}

// Mouse
enum MouseButton { Left, Right, Middle, X1, X2 }

struct MouseState {
    position: Vec2
    delta: Vec2
    scroll: Vec2
    buttons: Map[MouseButton, KeyState]
}

// Gamepad (future)
struct GamepadState {
    connected: bool
    buttons: Map[GamepadButton, KeyState]
    left_stick: Vec2
    right_stick: Vec2
    left_trigger: f32
    right_trigger: f32
}
```

### Input Resource

```klar
struct Input {
    keyboard: Map[KeyCode, KeyState]
    mouse: MouseState
    gamepads: List[GamepadState]
}

impl Input {
    // Keyboard queries
    fn pressed(key: KeyCode) -> bool
    fn just_pressed(key: KeyCode) -> bool
    fn just_released(key: KeyCode) -> bool

    // Mouse queries
    fn mouse_position() -> Vec2
    fn mouse_delta() -> Vec2
    fn mouse_pressed(button: MouseButton) -> bool
    fn mouse_just_pressed(button: MouseButton) -> bool

    // Action mapping (future)
    fn action_pressed(action: string) -> bool
}
```

### Input Events

```klar
enum InputEvent {
    KeyPressed(KeyCode)
    KeyReleased(KeyCode)
    MouseMoved(Vec2)
    MousePressed(MouseButton)
    MouseReleased(MouseButton)
    MouseScrolled(Vec2)
    GamepadConnected(u32)
    GamepadDisconnected(u32)
}
```

### Note on Platform Binding

The input system defines the **abstraction**. Actual event polling requires platform bindings (SDL3, etc.) which will be implemented when Klar FFI is ready. For now, we can:
1. Implement the data structures and logic
2. Create a mock input system for testing
3. Document the platform integration points

### Deliverables
- [ ] `src/input/keys.kl`
- [ ] `src/input/mouse.kl`
- [ ] `src/input/gamepad.kl`
- [ ] `src/input/input.kl`
- [ ] `src/input/events.kl`
- [ ] `src/input/mod.kl`
- [ ] `tests/input_tests.kl`

---

## Phase 5: Graphics Abstraction

**Goal**: Define rendering interfaces (implementation requires FFI)

### Sprite System

```klar
// What we render
struct Sprite {
    texture: Handle[Texture]
    region: ?Rect           // For sprite sheets, None = whole texture
    color: Color            // Tint/modulation
    flip_x: bool
    flip_y: bool
    anchor: Vec2            // 0,0 = top-left, 0.5,0.5 = center
}

// Texture handle (actual data managed by renderer)
struct Handle[T] {
    id: u64
    // Marker for type safety
}

struct Texture {
    width: u32
    height: u32
    // Actual GPU data is opaque
}
```

### Rendering Components

```klar
// Basic 2D rendering
struct SpriteRenderer {
    sprite: Sprite
    layer: i32              // Z-ordering
    visible: bool
}

// Animation
struct SpriteAnimation {
    frames: List[Rect]      // Regions in sprite sheet
    frame_duration: f32     // Seconds per frame
    current_frame: u32
    elapsed: f32
    looping: bool
}

// Text rendering
struct Text {
    content: String
    font: Handle[Font]
    size: f32
    color: Color
    alignment: TextAlign
}

enum TextAlign { Left, Center, Right }
```

### Camera

```klar
struct Camera2D {
    position: Vec2
    zoom: f32
    rotation: f32
    viewport: Rect          // Screen region to render to
}

impl Camera2D {
    fn world_to_screen(point: Vec2) -> Vec2
    fn screen_to_world(point: Vec2) -> Vec2
    fn view_matrix() -> Mat3
}
```

### Renderer Trait

```klar
// Platform-specific renderers implement this
trait Renderer {
    fn begin_frame()
    fn end_frame()

    fn clear(color: Color)
    fn set_camera(camera: ref Camera2D)

    fn draw_sprite(sprite: ref Sprite, transform: ref Transform2D)
    fn draw_rect(rect: Rect, color: Color)
    fn draw_rect_outline(rect: Rect, color: Color, thickness: f32)
    fn draw_circle(center: Vec2, radius: f32, color: Color)
    fn draw_line(start: Vec2, end: Vec2, color: Color, thickness: f32)
    fn draw_text(text: ref Text, position: Vec2)

    // Texture management
    fn load_texture(path: string) -> Result[Handle[Texture], RenderError]
    fn unload_texture(handle: Handle[Texture])
}
```

### Render System

```klar
// System that collects and sorts renderables
fn collect_sprites(
    query: Query[(Entity, SpriteRenderer, Transform2D)],
    camera: Resource[Camera2D]
) -> List[RenderCommand]

// Sorted draw commands
struct RenderCommand {
    sprite: ref Sprite
    transform: Transform2D
    layer: i32
}
```

### Deliverables
- [ ] `src/graphics/sprite.kl`
- [ ] `src/graphics/texture.kl`
- [ ] `src/graphics/camera.kl`
- [ ] `src/graphics/text.kl`
- [ ] `src/graphics/renderer.kl`
- [ ] `src/graphics/mod.kl`
- [ ] `tests/graphics_tests.kl` (data structure tests)

---

## Phase 6: Physics & Collision

**Goal**: 2D physics simulation and collision detection

### Collision Shapes

```klar
enum Collider {
    Circle { radius: f32 }
    Rectangle { half_extents: Vec2 }
    Capsule { radius: f32, height: f32 }
    // Polygon { vertices: List[Vec2] }  // Future
}

struct ColliderComponent {
    shape: Collider
    offset: Vec2            // Relative to entity position
    is_trigger: bool        // Triggers don't block movement
    layer: u32              // Collision layer bitmask
    mask: u32               // What layers to collide with
}
```

### Physics Components

```klar
struct RigidBody {
    body_type: BodyType
    velocity: Vec2
    angular_velocity: f32
    mass: f32
    friction: f32
    restitution: f32        // Bounciness
    gravity_scale: f32
}

enum BodyType {
    Static      // Doesn't move (walls, platforms)
    Dynamic     // Full physics simulation
    Kinematic   // Moves but not affected by forces
}
```

### Collision Detection

```klar
struct Collision {
    entity_a: Entity
    entity_b: Entity
    normal: Vec2            // From A to B
    penetration: f32
    contact_point: Vec2
}

// Collision queries
fn check_collision(a: ref Collider, a_pos: Vec2, b: ref Collider, b_pos: Vec2) -> ?Collision

// Specific shape pairs
fn circle_circle(a: Circle, b: Circle) -> ?Collision
fn circle_rect(circle: Circle, rect: Rect) -> ?Collision
fn rect_rect(a: Rect, b: Rect) -> ?Collision

// Broad phase (spatial partitioning)
struct SpatialHash {
    cell_size: f32
    cells: Map[(i32, i32), List[Entity]]
}
```

### Collision Events

```klar
enum CollisionEvent {
    Started(Collision)      // Just started touching
    Ongoing(Collision)      // Still touching
    Ended(Entity, Entity)   // No longer touching
}
```

### Physics System

```klar
fn physics_step(
    bodies: Query[(Entity, RigidBody, Transform2D)],
    time: Resource[Time>
) {
    // 1. Apply gravity
    // 2. Integrate velocities
    // 3. Broad phase collision detection
    // 4. Narrow phase collision detection
    // 5. Resolve collisions
    // 6. Update positions
}
```

### Deliverables
- [ ] `src/physics/collider.kl`
- [ ] `src/physics/rigidbody.kl`
- [ ] `src/physics/collision.kl`
- [ ] `src/physics/spatial_hash.kl`
- [ ] `src/physics/systems.kl`
- [ ] `src/physics/mod.kl`
- [ ] `tests/physics_tests.kl`

---

## Phase 7: Audio System

**Goal**: Define audio playback interfaces

### Audio Types

```klar
struct AudioSource {
    clip: Handle[AudioClip]
    volume: f32             // 0.0 to 1.0
    pitch: f32              // 1.0 = normal
    looping: bool
    spatial: bool           // 2D positional audio
    state: PlaybackState
}

enum PlaybackState {
    Stopped
    Playing
    Paused
}

struct AudioClip {
    duration: f32           // Seconds
    // Actual audio data is opaque
}
```

### Audio Commands

```klar
// Fire-and-forget sound effects
fn play_sound(clip: Handle[AudioClip], volume: f32)
fn play_sound_at(clip: Handle[AudioClip], position: Vec2, volume: f32)

// Music (single track with crossfade)
fn play_music(clip: Handle[AudioClip], fade_in: f32)
fn stop_music(fade_out: f32)
fn set_music_volume(volume: f32)
```

### Audio Trait

```klar
trait AudioBackend {
    fn load_clip(path: string) -> Result[Handle[AudioClip], AudioError]
    fn unload_clip(handle: Handle[AudioClip])
    fn play(source: ref AudioSource)
    fn pause(source: ref AudioSource)
    fn stop(source: ref AudioSource)
    fn set_listener_position(position: Vec2)
}
```

### Deliverables
- [ ] `src/audio/clip.kl`
- [ ] `src/audio/source.kl`
- [ ] `src/audio/backend.kl`
- [ ] `src/audio/mod.kl`
- [ ] `tests/audio_tests.kl`

---

## Phase 8: Asset Management

**Goal**: Resource loading and lifecycle management

### Asset Handle System

```klar
struct AssetServer {
    // Loaded assets by path
    loaded: Map[string, dyn Asset]
    // Reference counts for unloading
    ref_counts: Map[string, u32]
}

trait Asset {
    fn load(path: string) -> Result[Self, AssetError]
    fn unload(self: inout Self)
}

impl AssetServer {
    fn load[T: Asset](path: string) -> Handle[T]
    fn get[T: Asset](handle: Handle[T]) -> ?ref T
    fn unload[T: Asset](handle: Handle[T])
    fn is_loaded[T: Asset](handle: Handle[T]) -> bool
}
```

### Asset Types

```klar
// Images
impl Texture: Asset { ... }

// Audio
impl AudioClip: Asset { ... }

// Fonts
impl Font: Asset { ... }

// Tilemaps (Tiled format)
struct Tilemap {
    width: u32
    height: u32
    tile_width: u32
    tile_height: u32
    layers: List[TilemapLayer]
    tilesets: List[Tileset]
}

// Sprite sheets with metadata
struct SpriteSheet {
    texture: Handle[Texture]
    frames: Map[string, Rect]  // Named regions
    animations: Map[string, AnimationData]
}
```

### Deliverables
- [ ] `src/assets/handle.kl`
- [ ] `src/assets/server.kl`
- [ ] `src/assets/texture.kl`
- [ ] `src/assets/audio.kl`
- [ ] `src/assets/tilemap.kl`
- [ ] `src/assets/spritesheet.kl`
- [ ] `src/assets/mod.kl`
- [ ] `tests/assets_tests.kl`

---

## Phase 9: UI System

**Goal**: Immediate-mode UI for menus and HUD

### UI Context

```klar
struct UiContext {
    // Layout state
    cursor: Vec2
    current_layer: i32

    // Interaction state
    hot: ?WidgetId          // Mouse is over
    active: ?WidgetId       // Being interacted with

    // Input
    mouse_pos: Vec2
    mouse_down: bool
    mouse_clicked: bool
}
```

### Basic Widgets

```klar
impl UiContext {
    // Layout
    fn begin_horizontal()
    fn end_horizontal()
    fn begin_vertical()
    fn end_vertical()
    fn spacing(pixels: f32)

    // Widgets (return true if interacted)
    fn button(label: string) -> bool
    fn label(text: string)
    fn image(texture: Handle[Texture], size: Vec2)
    fn slider(value: inout f32, min: f32, max: f32) -> bool
    fn checkbox(label: string, checked: inout bool) -> bool
    fn text_input(buffer: inout String, max_len: u32) -> bool

    // Containers
    fn begin_window(title: string, rect: inout Rect) -> bool
    fn end_window()
    fn begin_panel(rect: Rect)
    fn end_panel()
}
```

### UI Styling

```klar
struct UiStyle {
    font: Handle[Font]
    font_size: f32

    text_color: Color
    button_color: Color
    button_hover_color: Color
    button_active_color: Color

    padding: f32
    spacing: f32
    border_radius: f32
}
```

### Deliverables
- [ ] `src/ui/context.kl`
- [ ] `src/ui/layout.kl`
- [ ] `src/ui/widgets.kl`
- [ ] `src/ui/style.kl`
- [ ] `src/ui/mod.kl`
- [ ] `tests/ui_tests.kl`

---

## Phase 10: Platform Bindings (Requires Klar FFI)

**Goal**: Connect to actual graphics/audio/input systems

**Blocked on**: Klar C FFI implementation

### Planned Bindings

1. **SDL3** - Window, input, audio, GPU rendering
   - `bindings/sdl3/`
   - Cross-platform (macOS, Linux, Windows when supported)
   - **Why SDL3 over SDL2:**
     - New GPU API (SDL_GPU) with built-in shader cross-compilation
     - Cleaner, more consistent API surface
     - Better defaults (e.g., high-DPI by default)
     - Modern event handling with properties system
     - First-class HDR and colorspace support

2. **SDL3 GPU API** - Hardware-accelerated rendering
   - Uses SDL_GPU instead of raw OpenGL/Vulkan/Metal
   - Automatic backend selection (Vulkan, D3D12, Metal)
   - Built-in shader compilation from SDL_shadercross
   - Sprite batching via custom pipelines

3. **Alternative: Software Renderer**
   - Pure Klar CPU renderer for testing
   - Write pixels to buffer, SDL3 presents via SDL_Surface

### Integration Strategy

```klar
// Platform trait implementations
struct SDL3Platform {
    window: SDL_Window
    gpu_device: SDL_GPUDevice
    audio_device: SDL_AudioDeviceID
}

impl SDL3Platform: Platform {
    fn create_window(title: string, width: u32, height: u32) -> Result[Self, PlatformError]
    fn poll_events() -> List[InputEvent]
    fn begin_gpu_frame() -> SDL_GPUCommandBuffer
    fn submit_gpu_frame(cmd: SDL_GPUCommandBuffer)
    fn quit()
}

// SDL3 GPU renderer implementation
struct SDL3Renderer {
    device: SDL_GPUDevice
    sprite_pipeline: SDL_GPUGraphicsPipeline
    // Batched sprite data
    vertex_buffer: SDL_GPUBuffer
    texture_sampler: SDL_GPUSampler
}

impl SDL3Renderer: Renderer {
    fn begin_frame() { ... }
    fn end_frame() { ... }
    fn draw_sprite(sprite: ref Sprite, transform: ref Transform2D) { ... }
    // ... etc
}
```

### SDL3 Key Types (for FFI reference)

```
SDL_Window           - Window handle
SDL_GPUDevice        - GPU context
SDL_GPUCommandBuffer - Frame command recording
SDL_GPUTexture       - GPU texture
SDL_GPUBuffer        - Vertex/index/uniform buffers
SDL_GPUGraphicsPipeline - Shader + render state
SDL_AudioDeviceID    - Audio output
SDL_Event            - Input events (unified struct)
```

---

## Example Game: Pong

**Goal**: Validate the engine by building a complete game

### Components Needed
```klar
struct Paddle { speed: f32 }
struct Ball { velocity: Vec2 }
struct Score { left: u32, right: u32 }
struct Wall {}
```

### Systems
```klar
fn player_input(input: Resource[Input], query: Query[(Paddle, Velocity)>)
fn ai_paddle(query: Query[(Paddle, Transform2D)>, ball: Query[(Ball, Transform2D)>)
fn ball_movement(time: Resource[Time], query: Query[(Ball, Transform2D)>)
fn collision_response(events: EventReader[CollisionEvent], ...)
fn score_display(score: Resource[Score>, ui: Resource[UiContext>)
```

### Validation Criteria
- [ ] Smooth paddle movement
- [ ] Ball bounces correctly
- [ ] Score updates on goals
- [ ] Game resets after point
- [ ] AI opponent works
- [ ] No memory leaks

---

## Timeline & Dependencies

```
Phase 0: Foundation          [No dependencies]
    ↓
Phase 1: Math Library        [No dependencies]
    ↓
Phase 2: ECS Core            [Depends on: Math]
    ↓
Phase 3: Scheduling          [Depends on: ECS]
    ↓
Phase 4: Input               [Depends on: Math, ECS]
    ↓
Phase 5: Graphics            [Depends on: Math, ECS]
    ↓
Phase 6: Physics             [Depends on: Math, ECS]
    ↓
Phase 7: Audio               [Depends on: ECS]
    ↓
Phase 8: Assets              [Depends on: Graphics, Audio]
    ↓
Phase 9: UI                  [Depends on: Graphics, Input]
    ↓
Phase 10: Platform Bindings  [BLOCKED: Klar FFI]
    ↓
Example: Pong                [Depends on: All above]
```

## Testing Strategy

### Unit Tests
- Every module has `tests/<module>_tests.kl`
- Property-based tests for math operations
- Edge case coverage

### Integration Tests
- System interaction tests
- Full frame simulation tests
- Memory leak detection

### Example Games as Tests
- Pong: Basic game loop validation
- Breakout: Collision and physics validation
- Flappy Bird: Input and animation validation

---

## Documentation Plan

### For AI Agents
- Every public function has doc comments
- Examples in documentation are runnable
- Error messages include fix suggestions
- Consistent naming patterns throughout

### For Benchmarking
- Clear task specifications
- Success criteria for each module
- Novel patterns documented (not from common engines)

### Documentation Files
- `docs/getting-started.md`
- `docs/architecture.md`
- `docs/ecs-guide.md`
- `docs/api/` (generated from doc comments)
- `docs/examples/` (annotated example code)

---

## Risk Mitigation

| Risk | Mitigation |
|------|------------|
| Klar FFI not ready | Build pure-Klar software renderer as fallback, SDL3 presents |
| Performance issues | Profile early, optimize math library first |
| API changes in Klar | Pin to specific Klar version, adapt incrementally |
| Scope creep | Strict phase boundaries, Pong before anything else |

---

## Success Criteria

### Minimum Viable Engine
- [ ] Can create a window
- [ ] Can render sprites
- [ ] Can handle input
- [ ] Can play sounds
- [ ] Can detect collisions
- [ ] Pong is playable

### AI Benchmark Ready
- [ ] Documentation covers all APIs
- [ ] Error messages are actionable
- [ ] Examples compile and run
- [ ] Novel patterns are documented
- [ ] Tasks have clear success criteria

---

*This plan will evolve as Klar develops and we learn from implementation.*
