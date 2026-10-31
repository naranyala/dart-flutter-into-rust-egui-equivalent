# Dart/Flutter to Rust/egui: Technical Translation Catalog

A technical field guide for mapping Flutter and Dart mental models to Rust and egui.

This repository is intended as a practical translation catalog: a reference for understanding how concepts from Flutter/Dart map onto Rust/egui architecture, rendering, input, state, layout, async work, and application structure.

The aim is not to mimic Flutter exactly. The aim is to help a developer who already understands Flutter think correctly about how to build equivalent applications in Rust using egui.

---

## Executive summary

Flutter and egui share a broad goal: building interactive user interfaces, but they do so with very different execution models.

Flutter is a retained UI framework built around a widget tree, composition, and a render pipeline managed by the framework. The developer describes the UI as a hierarchy of widgets, and the framework decides how and when to rebuild parts of the tree.

egui is an immediate-mode UI library. The developer emits UI commands each frame from the current application state. The UI is not preserved as a widget tree; instead, it is rebuilt from state every frame, and interaction is interpreted from response objects that tell you what happened.

This means the most important translation is not “Widget X becomes egui Widget Y,” but rather:

- Flutter stateful composition becomes app state + update loop
- Flutter build context becomes runtime UI context + explicit state ownership
- Flutter callbacks become `Response` values and command logic
- Flutter widget tree becomes immediate UI generation from current state
- Flutter layouts become explicit layout methods and region allocation
- Flutter async streams become channels, state transitions, and repaint triggers

---

## Core mental model differences

### 1. Flutter: retained widget graph

Flutter works by constructing a widget tree and letting the framework reconcile it to the render tree. UI changes happen through rebuilds and state propagation.

```dart
class CounterPage extends StatefulWidget {
  const CounterPage({super.key});

  @override
  State<CounterPage> createState() => _CounterPageState();
}

class _CounterPageState extends State<CounterPage> {
  int count = 0;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Center(
        child: Column(
          children: [
            Text('$count'),
            ElevatedButton(
              onPressed: () => setState(() => count++),
              child: const Text('Increment'),
            ),
          ],
        ),
      ),
    );
  }
}
```

The conceptual model is: widget tree + rebuild + diff + paint.

### 2. egui: immediate UI from current state

egui emits UI each frame by running your code against the current state.

```rust
struct CounterApp {
    count: i32,
}

impl eframe::App for CounterApp {
    fn update(&mut self, ctx: &egui::Context, _frame: &mut eframe::Frame) {
        egui::CentralPanel::default().show(ctx, |ui| {
            ui.vertical_centered(|ui| {
                ui.label(self.count.to_string());
                if ui.button("Increment").clicked() {
                    self.count += 1;
                }
            });
        });
    }
}
```

The conceptual model is: state + frame + emit UI + react to response.

---

## Translation matrix: Flutter to Rust/egui

| Flutter / Dart concept | Rust / egui equivalent | Technical translation |
| --- | --- | --- |
| `runApp` | `eframe::run_native` / `eframe::App` | Bootstraps the UI loop and application context |
| `MaterialApp` | `eframe::App` with custom state | Root application shell |
| `StatefulWidget` | app struct + mutable state | State lives in app instance |
| `build(BuildContext ctx)` | `App::update(&mut self, ctx, frame)` | UI emitted on each frame |
| `setState(() => ...)` | mutate fields, then repaint | State mutation triggers redraw |
| `Widget` | `egui::Ui` closures and widgets | UI is produced by immediate commands |
| `BuildContext` | `&egui::Context`, `&mut Ui`, `Rect`, `Response` | Context is explicit and local |
| `Column` / `Row` | `ui.vertical`, `ui.horizontal` | Layout is composed in closures |
| `Expanded` / `SizedBox` | `ui.allocate_ui`, `ui.add_sized` | Space is allocated explicitly |
| `Text` | `ui.label`, `ui.heading`, etc. | Immediate text rendering |
| `TextField` | `ui.text_edit_singleline`, `TextEdit::multiline` | String state bound to input |
| `Button` | `ui.button`, `Button::new` | Clicked state is returned via `Response` |
| `Container` | `ui.scope` / layout builder | Regions and spacing are explicit |
| `Padding` | `ui.add_space`, `style.spacing`, `ui.painter` | Visual spacing is controlled with style/layout |
| `ListView` | `ScrollArea::vertical` + loop | Scroll container and looped render |
| `GridView` | `ui.columns`, loops, `ui.allocate_ui` | Column-based layout grid |
| `Navigator` | `enum Screen` / state machine | Navigation modeled as state transitions |
| `MaterialPageRoute` | custom screen switch | Screen transitions are app-defined |
| `ThemeData` | `egui::Visuals` + `Context::set_visuals` | Theme is runtime style configuration |
| `FutureBuilder` | async task + channel or polling + repaint | UI updates when state changes |
| `StreamBuilder` | `mpsc`/`watch` + repaint | Values move through channels |
| `ChangeNotifier` | `Arc<Mutex<T>>`, `watch`, `mpsc` | Shared state propagation |
| `Provider` / `InheritedWidget` | app state, context-local state struct | Data flow is explicit |
| `AnimationController` | timer + interpolation + request_repaint | Repaint loop with time-based interpolation |
| `GestureDetector` | `Response` hover, drag, click, pointer data | Input is directly observable |
| `Material` components | `egui` widgets + custom styling | Custom components often require explicit wrappers |

---

## App lifecycle and root architecture

### Flutter app lifecycle

Flutter applications typically have:

- application boot
- widget tree construction
- stateful rebuilds
- lifecycle hooks like `initState`, `didChangeDependencies`, `dispose`

### egui app lifecycle

egui applications are more straightforward:

- construct the app state
- create a type implementing `eframe::App`
- drive the UI in `update()` every frame
- use `ctx.request_repaint()` to force redraws when external events happen

```rust
struct App {
    counter: i32,
    message: String,
}

impl eframe::App for App {
    fn update(&mut self, ctx: &egui::Context, frame: &mut eframe::Frame) {
        // Update loop
        egui::CentralPanel::default().show(ctx, |ui| {
            ui.heading("Example App");
            ui.label("Current state is live on every frame.");
            ui.separator();

            if ui.button("Increment").clicked() {
                self.counter += 1;
                self.message = format!("Counter is now {}", self.counter);
            }

            ui.text_edit_singleline(&mut self.message);
            ui.label(format!("Counter: {}", self.counter));
        });

        // Optional: trigger repaint from external async work
        // ctx.request_repaint();
    }
}
```

### Translation

In Flutter, widgets are often able to own local state and are recreated as part of the framework lifecycle. In egui, your app owns the state; you mutate it and decide when to repaint. The state is not hidden inside widget lifecycles.

---

## Rendering model and UI generation

### Flutter rendering model

Flutter renders through a tree reconciliation process:

- widgets describe what should be displayed
- the element tree tracks identity
- render objects compute layout and painting
- build tree changes are reconciled to the render object tree

### egui rendering model

egui uses immediate mode:

- each frame calls your UI code
- widgets are built directly from the current state
- interaction results (`Response`) are returned immediately
- the backend produces paint commands from those UI operations

Practically, this means:

```rust
let response = ui.button("Open settings");
if response.clicked() {
    self.show_settings = true;
}
```

A Flutter developer usually thinks of a widget as a object with identity. In egui, the important object is the `Response` generated by the operation that just happened.

---

## Layout systems: Flutter vs egui

### Flutter layout primitives

Flutter layout is a rich flexible system based on render box layout, constraints, and composition. Typical constructs:

- `Row`, `Column`
- `Stack`, `Positioned`
- `Expanded`, `Flexible`
- `Align`, `Center`, `SizedBox`
- `IntrinsicHeight`, `AspectRatio`
- `Wrap`

### egui layout primitives

egui is simpler but still powerful. It emphasizes region allocation and layout closures.

- `ui.horizontal(|ui| { ... })`
- `ui.vertical(|ui| { ... })`
- `ui.columns(count, |uis| { ... })`
- `ui.allocate_ui(size, |ui| { ... })`
- `ui.scope(|ui| { ... })`
- `ui.with_layout(Layout::top_down(Align::LEFT), |ui| { ... })`

Example:

```rust
ui.horizontal(|ui| {
    ui.label("Name:");
    ui.add_sized([180.0, 22.0], egui::TextEdit::singleline(&mut self.name));
    if ui.button("Save").clicked() {
        self.save_name();
    }
});
```

### Important translation

Flutter layout is constraint-based and inherited. egui layout is immediate and region-based. In Flutter, a child receives constraints and decides what size to take. In egui, the parent allocates a region, and the child uses what it needs within that area.

---

## State management translation

### Flutter state patterns

Flutter has a wide range of patterns:

- `setState`
- `InheritedWidget`
- `Provider`
- `Riverpod`
- `Bloc`
- `Redux`
- `ChangeNotifier`

### egui state patterns

Rust/egui usually relies on:

- app struct fields
- enums for screens, modes, and substates
- shared state via `Arc<Mutex<T>>` or `Arc<RwLock<T>>`
- `tokio::sync::watch` or `mpsc` for async updates
- event-driven commands and actions

Example with central app state:

```rust
struct AppState {
    input: String,
    selected_tab: Tab,
    dirty: bool,
}

enum Tab {
    Overview,
    Details,
    Settings,
}
```

Then use it inside `update()`.

```rust
impl eframe::App for AppState {
    fn update(&mut self, ctx: &egui::Context, _frame: &mut eframe::Frame) {
        egui::TopBottomPanel::top("topbar").show(ctx, |ui| {
            ui.horizontal(|ui| {
                if ui.selectable_label(self.selected_tab == Tab::Overview, "Overview").clicked() {
                    self.selected_tab = Tab::Overview;
                }
                if ui.selectable_label(self.selected_tab == Tab::Details, "Details").clicked() {
                    self.selected_tab = Tab::Details;
                }
            });
        });

        egui::CentralPanel::default().show(ctx, |ui| {
            match self.selected_tab {
                Tab::Overview => {
                    ui.label("Overview panel");
                }
                Tab::Details => {
                    ui.label("Details panel");
                }
                Tab::Settings => {
                    ui.label("Settings panel");
                }
            }
        });
    }
}
```

### Translation rules

- `StatefulWidget` -> app state struct
- `setState` -> mutate state in `update()`
- `Provider` -> shared state or app-scoped state container
- `InheritedWidget` -> explicit state passed around or stored in app root
- `StreamBuilder` -> channels or polling loop + repaint

---

## Interaction model: callbacks and responses

### Flutter

Flutter interaction is usually callback-based.

```dart
TextButton(
  onPressed: () {
    doThing();
  },
  child: const Text('Do thing'),
)
```

### egui

egui returns a `Response` object for each widget call.

```rust
let response = ui.button("Do thing");
if response.clicked() {
    do_thing();
}
```

This is important because the event model is not built around an implicit callback system. Instead, the UI interaction is encoded in the returned object.

You can inspect many properties:

- `clicked()`
- `double_clicked()`
- `hovered()`
- `dragged()`
- `context_menu()`
- `contains_pointer()`
- `is_pointer_button_down_on()`

This enables precise, imperative logic:

```rust
let response = ui.add(egui::Button::new("Launch"));
if response.clicked() {
    self.launch_action();
}

if response.hovered() {
    ui.ctx().output(|o| o.cursor_icon = egui::CursorIcon::PointingHand);
}
```

---

## Input handling and focus

### Flutter input

Flutter input is strongly integrated with widget tree focus management, keyboard handling, text editing, and gesture arenas.

### egui input

egui gets input from the OS and exposes it through `egui::Context` and `ui` calls. The library handles input events and gives you responses.

Examples:

```rust
ui.text_edit_singleline(&mut self.search_text);

if ui.input_mut(|i| i.consume_key(egui::Modifiers::NONE, egui::Key::Enter)) {
    self.submit_search();
}
```

You can also inspect input state:

```rust
if ui.input(|i| i.modifiers.shift) {
    // shift key active
}
```

### Translation

Flutter developers often expect a focus tree and widget-level focus management. In egui, focus and input are evaluated using the current input state and response objects rather than a focus node hierarchy.

---

## Data and model architecture

### Flutter architecture patterns

Common patterns include:

- widget-local state
- repository/service layer
- bloc / cubit
- riverpod providers
- domain-driven model separation

### Rust/egui architecture patterns

Typical Rust patterns are closer to:

- `AppState` for runtime state
- domain structs and models
- services or commands with separation of responsibilities
- `Arc<Mutex<_>>` for shared state
- `enum` for app workflow and screens

Example structure:

```text
src/
  main.rs
  app.rs
  state.rs
  models.rs
  services/
    data_service.rs
    history_service.rs
  screens/
    dashboard.rs
    editor.rs
    settings.rs
```

### Translation

Flutter architecture often emphasizes widget composition and dependency injection. Rust architecture is often more explicit about ownership and message flow. That means your app state should be designed around clear ownership boundaries rather than simply attaching everything to widgets.

---

## Async and concurrency model

### Flutter async

Flutter uses async/await and streams extensively. Common patterns:

- `FutureBuilder`
- `StreamBuilder`
- async REST calls
- scheduled tasks
- timers

### egui async

Rust has a more explicit concurrency model. You often use:

- `std::thread`
- `tokio` tasks
- `async` functions
- `mpsc` / `watch` channels
- `Arc<Mutex<T>>`

```rust
use std::sync::mpsc;

let (tx, rx) = mpsc::channel();

std::thread::spawn(move || {
    let result = fetch_data();
    tx.send(result).unwrap();
});

// In app update loop
if let Ok(data) = rx.try_recv() {
    self.data = data;
    ctx.request_repaint();
}
```

### Important note

A Flutter app often has a relatively friendly UI main thread model. In Rust, UI work is still on the UI thread, while long-running work often occurs in background threads or async tasks. State updates need to be synchronized and then paired with `ctx.request_repaint()` when data is ready.

---

## The event loop and redraw strategy

### Flutter rebuild model

Flutter decides when to rebuild based on state changes and widget identity.

### egui redraw model

egui is simpler: the frame function runs repeatedly, and your app decides when to repaint.

```rust
if self.needs_refresh {
    ctx.request_repaint();
}
```

This should be seen as a direct translation of app-level update signals. If a background task completes, that task should notify the app and then request a repaint.

Very often the pattern is:

1. async task completes
2. channel receives message
3. app mutates state
4. `ctx.request_repaint()` is called
5. next frame draws updated UI

---

## Widgets and custom component patterns

### Flutter custom widgets

Flutter developers often encapsulate reusable UI in custom widget classes:

```dart
class ProfileCard extends StatelessWidget {
  final String name;

  const ProfileCard({super.key, required this.name});

  @override
  Widget build(BuildContext context) {
    return Card(
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Text(name),
      ),
    );
  }
}
```

### egui custom widgets

In egui, reuse usually happens through helper functions or component-like structs.

```rust
fn profile_card(ui: &mut egui::Ui, name: &str) {
    ui.group(|ui| {
        ui.horizontal(|ui| {
            ui.label("User:");
            ui.strong(name);
        });
    });
}
```

Or as a method on an app state:

```rust
impl App {
    fn profile_card(&mut self, ui: &mut egui::Ui) {
        ui.group(|ui| {
            ui.label("Profile");
            ui.horizontal(|ui| {
                ui.label("Name:");
                ui.text_edit_singleline(&mut self.name);
            });
        });
    }
}
```

### Translation

Flutter custom widget classes are often replaced by Rust functions, closure-based UI helpers, or small UI component structs.

---

## Navigation and screen flow

### Flutter navigation model

Flutter navigation is built around routes and the navigator stack.

```dart
Navigator.push(
  context,
  MaterialPageRoute(builder: (_) => const SettingsPage()),
);
```

### egui navigation model

In egui, navigation is usually represented as app state or an enum-driven view stack.

```rust
enum Screen {
    Home,
    Settings,
    Editor,
}

struct App {
    current_screen: Screen,
}

impl eframe::App for App {
    fn update(&mut self, ctx: &egui::Context, _frame: &mut eframe::Frame) {
        match self.current_screen {
            Screen::Home => {
                // draw home
            }
            Screen::Settings => {
                // draw settings
            }
            Screen::Editor => {
                // draw editor
            }
        }
    }
}
```

If you need a stack, you can model it explicitly:

```rust
struct App {
    screens: Vec<Screen>,
}
```

### Translation

In Flutter, route transitions are framework-supported and stack-based. In egui, navigation is usually encoded directly as application state and view selection, which is often more explicit and less magic.

---

## Design system and visual styling

### Flutter styling model

Flutter styling is largely theme and widget-based:

- `ThemeData`
- `TextTheme`
- `ColorScheme`
- `Card`, `AppBar`, `ElevatedButton`
- inherited theme data

### egui styling model

egui uses a more direct visual configuration model:

- `egui::Visuals`
- `ui.style_mut()`
- `egui::FontId`
- `egui::Color32`
- custom widget wrappers and style overrides

```rust
let mut style = (*ctx.style()).clone();
style.spacing.item_spacing = egui::vec2(8.0, 8.0);
style.visuals.extreme_bg_color = egui::Color32::from_rgb(18, 18, 18);
ctx.set_style(style);
```

### Translation

Flutter developers often rely on a comprehensive design system. egui gives you a smaller built-in styling surface but exposes enough configuration to build your own visual language.

---

## Testing mental model

### Flutter testing

Flutter encourages widget testing and golden tests: rendering and interaction are tested at the hierarchy level.

### egui testing

A Rust/egui app is more naturally tested with:

- unit tests for business logic and state transitions
- app logic tests for data transformations and screen transitions
- integration tests around state update logic
- UI tests when needed, but usually less framework-driven

Example:

```rust
#[test]
fn increment_updates_count() {
    let mut app = App { count: 0 };
    app.increment();
    assert_eq!(app.count, 1);
}
```

### Translation

Flutter testing is often centered around widget behavior. Rust/egui testing is often centered on data and app logic because the UI is generated immedately from state.

---

## Common Flutter patterns and their egui equivalents

### 1. `setState` -> mutate state and repaint

Flutter:

```dart
setState(() => counter++);
```

egui:

```rust
self.counter += 1;
ctx.request_repaint();
```

---

### 2. `StatefulWidget` -> app root or screen state

Flutter:

```dart
class SearchPage extends StatefulWidget {
  // stateful UI
}
```

egui:

```rust
struct SearchPageState {
    query: String,
    result: Vec<Item>,
}
```

---

### 3. `FutureBuilder` -> async task + channel notifications

Flutter:

```dart
FutureBuilder(
  future: loadData(),
  builder: (context, snapshot) => Text(snapshot.data ?? 'Loading'),
)
```

egui:

```rust
// spawn task
// send result via channel
// in update(): if message received, update state and repaint
```

---

### 4. `ListView` -> `ScrollArea` + loop

Flutter:

```dart
ListView.builder(
  itemCount: items.length,
  itemBuilder: (context, index) => Text(items[index]),
)
```

egui:

```rust
egui::ScrollArea::vertical().show(ui, |ui| {
    for item in &self.items {
        ui.label(item);
    }
});
```

---

### 5. `TextField` -> `TextEdit`

Flutter:

```dart
TextField(controller: controller)
```

egui:

```rust
ui.text_edit_singleline(&mut self.controller_value);
```

---

### 6. `ThemeData` -> `Visuals` + style mutations

Flutter:

```dart
ThemeData.dark();
```

egui:

```rust
let mut visuals = egui::Visuals::dark();
ctx.set_visuals(visuals);
```

---

### 7. `Navigator` -> enum screen state

Flutter:

```dart
Navigator.push(context, route);
```

egui:

```rust
self.screen = Screen::Settings;
```

---

## High-level design guidance

### When a Flutter developer should think in egui terms:

- state ownership is explicit and central
- rendering is generated each frame from state
- event handling is response-based
- UI can be kept lightweight and compositional
- background work is isolated from UI logic

### When a Rust/egui app should mirror Flutter patterns:

- you want a maintainable screen structure
- you want to keep a clear separation between UI state and domain state
- you want to reuse components and keep a modular app shell
- you want a strong conceptual bridge from Flutter to Rust

---

## Best-practice translation checklist

If you're translating a Flutter app into egui, ask these questions:

- Where is the state actually owned?
- What is the root app state?
- Which code is UI generation and which is domain logic?
- What should trigger repaint?
- Are async updates flowing through channels or shared state?
- Is this a screen state, a modal state, or a shared app state?
- Which values are local to a single panel vs global to the app?
- Are we thinking in retained widget structure or state-driven UI?

---

## Trade-offs and practical realities

### Flutter strengths

- mature widget ecosystem
- strong material design integration
- dependable build and hot reload workflow
- excellent for mobile-first app UIs and rich UI hierarchies

### egui strengths

- lean and immediate rendering pipeline
- Rust-native safety and performance
- very strong for desktop, tools, editors, diagnostics, control surfaces
- simpler mental model for stateful UI once you learn it

### When not to expect direct equivalence

Some Flutter concepts are deeply tied to the framework runtime and do not map 1:1 to egui:

- inherited widget trees
- framework-level route builders
- rich material semantics and widget-specific behaviors
- automatic reconciliation over large widget trees

Instead, the correct translation is to model the same behavior with explicit application state, view selection, and imperative UI generation.

---

## Recommended developer mindset

The most useful way to translate Flutter into egui is:

- Stop thinking: “What widget tree should I build?”
- Start thinking: “What state does my app need, and how should the UI be emitted from it each frame?”

This shift is the heart of the translation.

---

## One-page cheat sheet

```text
Flutter concept                | Rust/egui equivalent
-----------------------------|-------------------------------
StatefulWidget                | App struct + state fields
BuildContext                  | Ui context / explicit state access
setState                     | mutate state + repaint
Widget tree                  | immediate UI build each frame
Layout constraints           | allocate_ui and layout closures
Row / Column                 | ui.horizontal / ui.vertical
ListView                     | ScrollArea + loop
TextField                    | TextEdit + String state
Button                       | ui.button() + Response
Navigator                    | enum Screen + app state
ThemeData                    | Visuals + style mutations
FutureBuilder/StreamBuilder  | async task + channel + repaint
Provider/InheritedWidget     | app-level state container
GestureDetector             | Response hover/click/drag checks
```

---

## Final takeaway

Flutter and egui are not equivalent in implementation model, but they are equivalent in intent: both provide a mechanism for expressing interactive UI, responding to user input, and mapping state to visible output.

Flutter developers already understand the essential problems: composition, layout, state, interaction, data flow, and reactivity. Those concepts transfer to egui through a different model:

- app state is explicit
- the UI is emitted from that state
- input is captured via responses
- redraws are scheduled intentionally
- asynchronous work is integrated through channels and repaint signals

That is the real technical translation from Dart/Flutter expertise to Rust/egui.

---

## Future direction: Flutter-like abstraction over egui

A compelling next step is not to replace egui, but to create a Flutter-shaped abstraction layer that preserves familiar developer ergonomics while remaining grounded in Rust's ownership model and egui's immediate-mode runtime.

The central design idea is simple:

- keep raw egui for rendering, input, and painting
- build a Flutter-like surface on top of it for ergonomics and API familiarity
- treat the “widget tree” as a high-level DSL or builder API, not as a literal retained runtime tree

This would result in a library that feels like a Flutter-compatible ecosystem, but internally executes through immediate-mode UI generation.

### Why this is a strong direction

This abstraction aims to satisfy several goals at once:

- mimic a Flutter-like developer experience for app composition
- preserve Rust's explicit state ownership and thread-safety model
- keep immediate-mode efficiency and simplicity from egui
- reduce the conceptual leap for Flutter developers entering Rust UI development

### Core abstraction model

The library would likely be structured in layers:

1. Raw egui runtime
   - `Context`, `Ui`, `Response`, `Rect`, painting, input state, style system, timers, repaint loop

2. Flutter-like framework layer
   - `App`, `MaterialApp`, `Scaffold`, `Column`, `Row`, `Text`, `Button`, `StatefulWidget`, `State<T>`, `BuildContext`

3. Runtime/state engine
   - widget registry, state lifecycle, navigation stack, async updates, repaint scheduling

4. Ecosystem helpers
   - theming, navigation, dialogs, forms, data tables, custom components, async builders, gestures

### The key architectural principle

The abstracted framework should mimic the API shape and semantics of Flutter, but not its internal retained widget architecture.

In other words:

- Flutter interface: familiar and declarative
- egui backend: immediate-mode, fast, efficient, explicit
- Rust runtime: ownership-safe, stateful, compositional

This yields a hybrid architecture that feels familiar without sacrificing the actual strengths of Rust and egui.

### A representative conceptual API

```rust
struct MyApp {
    counter: i32,
}

impl App for MyApp {
    fn build(&mut self, ctx: &mut BuildContext) {
        MaterialApp::new()
            .theme(Theme::dark())
            .home(
                Scaffold::new()
                    .body(
                        Column::new()
                            .padding(16.0)
                            .children([
                                Text::new("Hello from Rust/egui"),
                                SizedBox::height(12.0),
                                Button::new("Increment")
                                    .on_click(|app| app.counter += 1),
                                Text::new(format!("Count: {}", app.counter)),
                            ]),
                    ),
            )
            .build(ctx);
    }
}
```

This is the intended user-facing ergonomics: a Flutter-like UI DSL, but still real-time, immediate-mode, and Rust-native under the hood.

### Core building blocks for the abstraction

- `App` and application lifecycle
- `BuildContext` as a lightweight runtime context object
- `Widget` trait for UI elements
- `State<T>` for stateful widgets and mutable data
- `Navigator` / route stack for screen management
- `Theme` system for colors, typography, spacing, and visual style
- `FutureBuilder` / `StreamBuilder` analogs for async UI updates
- `Response`-based event handling mapped from egui input

### Why this matters for developers

For a Dart/Flutter developer, this abstraction provides a bridge rather than a complete rewrite. The mental model remains familiar:

- app lifecycle and composition
- stateful widgets and local state
- layouts and containers
- navigation and routes
- callbacks and user interactions
- theme and styling

At the same time, the implementation stays true to Rust safety and immediate-mode UI rendering.

---

## Suggested next steps

Use this catalog as a bridge while building a small project in Rust/egui, such as:

- a counter app
- a settings panel
- a list/search dashboard
- a small editor or data viewer
- a form-heavy tool with validation and async load

The fastest way to become fluent in egui is to implement a few Flutter-like patterns in Rust and compare the behavior directly.

---

## License

This repository is intended as a learning and translation reference for developers moving from Dart/Flutter to Rust/egui.
