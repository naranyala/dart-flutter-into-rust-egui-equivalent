# Dart/Flutter to Rust/egui: Translation Catalog

A practical guide for translating Dart/Flutter expertise into Rust + egui patterns and mental models.

This repository is a compact catalog of equivalent concepts, idioms, and implementation strategies so developers familiar with Flutter can quickly adapt to Rust GUI development with egui.

## Why this matters

Flutter and egui are built on very different philosophies:

- Flutter is a retained, widget-tree UI framework with a declarative runtime.
- egui is an immediate-mode GUI library in Rust that emphasizes simplicity, low ceremony, and direct UI commands.

The goal is not to mimic Flutter exactly. The goal is to map your understanding of app structure, rendering, state, layout, and interaction into the Rust/egui world.

---

## At a glance

| Flutter/Dart concept | Rust/egui equivalent | Mental model |
| --- | --- | --- |
| `MaterialApp` / `CupertinoApp` | `eframe::App` / custom app state | App shell lifecycle and root state |
| `Widget` | `egui::Ui` + immediate draw commands | UI is emitted as commands, not retained tree |
| `StatefulWidget` | `App::update` + mutable app state | State lives in the app struct |
| `BuildContext` | `&mut Context` / `&mut egui::Ui` | Context is passed explicitly |
| `Column` / `Row` | `ui.horizontal()`, `ui.vertical()` | Layout is composed with helper methods |
| `Container` | `ui.allocate_ui()` / `ui.horizontal_wrapped()` | Spaces are allocated and shaped dynamically |
| `Padding` / `Margin` | `ui.add_space()`, `egui::Spacing`, style | Layout decisions are explicit |
| `Text` | `ui.label()` | Immediate text rendering |
| `TextField` | `ui.text_edit_singleline()` / `ui.text_edit_multiline()` | Input state owned by app |
| `Button` | `ui.button()` | Push button returns bool when clicked |
| `ListView` | `ui.vertical_scroll_auto()` / custom scroll | Lists are rendered as a loop |
| `FutureBuilder` | `tokio` / `async` + UI polling | Background futures update state |
| `StreamBuilder` | `egui::Context::request_repaint()` + channel updates | Repaint when external state changes |
| `Theme` | `egui::Visuals`, `Context::set_visuals()` | Visuals are runtime-configurable |
| `InheritedWidget` | global app state / `Rc<RefCell<_>>` / `Arc<Mutex<_>>` | Data flows through app state |
| `Navigator` | custom state machine / screen enum | State-driven navigation |
| `AnimatedBuilder` / `AnimationController` | `egui::Context::animate()` / `Instant` + custom timers | Animation is explicit and custom |
| `Provider` / `Bloc` | `AppState`, channels, commands, enums | React to state transitions directly |
| `LayoutBuilder` | `ui.available_size()` / `ui.max_rect()` | Measure current available area |
| `GestureDetector` | custom pointer handling in `egui::Response` | Event handling via response objects |
| `Material` / `Cupertino` design system | `egui` built-in widgets and custom styling | Components are simpler and more raw |

---

## Core translation model

### 1. Flutter mindset: widgets are declarative

In Flutter, you often write:

```dart
Column(
  children: [
    Text('Hello'),
    ElevatedButton(
      onPressed: () {},
      child: Text('Click me'),
    ),
  ],
)
```

This is a tree of widget descriptions. The framework re-renders as needed.

### 2. egui mindset: emit UI immediately

In egui, you usually do:

```rust
impl eframe::App for MyApp {
    fn update(&mut self, ctx: &egui::Context, _frame: &mut eframe::Frame) {
        egui::CentralPanel::default().show(ctx, |ui| {
            ui.vertical(|ui| {
                ui.label("Hello");
                if ui.button("Click me").clicked() {
                    // handle click
                }
            });
        });
    }
}
```

The mental shift is:

- Flutter: describe the UI as a structure.
- egui: execute UI code imperatively each frame and react to the returned `Response` objects.

---

## Catalog of common translations

### App shell and lifecycle

#### Flutter
- `MaterialApp`
- `WidgetsApp`
- `runApp`
- `StatefulWidget`
- `build()`

#### egui
- `eframe::App`
- `App::update(ctx, frame)`
- `egui::CentralPanel`
- custom state struct
- state mutation in `update()`

Example:

```rust
struct MyApp {
    counter: i32,
}

impl eframe::App for MyApp {
    fn update(&mut self, ctx: &egui::Context, _frame: &mut eframe::Frame) {
        egui::CentralPanel::default().show(ctx, |ui| {
            if ui.button("+1").clicked() {
                self.counter += 1;
            }
            ui.label(format!("Counter: {}", self.counter));
        });
    }
}
```

#### Translation

- Flutter app state often lives in widgets.
- egui app state usually lives in a single app struct or a set of stateful values owned by the app.

---

### Layout and composition

#### Flutter
- `Column`, `Row`, `Stack`, `Center`, `Padding`, `Expanded`, `Flexible`
- `SizedBox`, `AspectRatio`

#### egui
- `ui.vertical`, `ui.horizontal`, `ui.allocate_ui`, `ui.scope`, `ui.columns`, `ui.separator`
- `egui::Rect`, `ui.max_rect`, `ui.available_size`
- custom layout helpers

```rust
ui.horizontal(|ui| {
    ui.label("Name:");
    ui.add_sized([180.0, 20.0], egui::TextEdit::singleline(&mut self.name));
});
```

#### Translation

Flutter layout is built around widget tree composition. egui layout is built via UI closures and sizing operations. You usually think in terms of current available space and emitted components.

---

### State management

#### Flutter
- `setState`
- `ValueNotifier`
- `ChangeNotifier`
- `Provider`
- `Riverpod`
- `Bloc`

#### Rust/egui
- `struct AppState`
- `Arc<Mutex<_>>` for shared state between threads/tasks
- `tokio::sync::watch` / `mpsc` for updates from async tasks
- state enums for screen transitions
- direct mutation inside `update()`

```rust
#[derive(Default)]
struct AppState {
    items: Vec<String>,
    input: String,
}
```

For asynchronous updates:

```rust
let (tx, rx) = std::sync::mpsc::channel();

// background task sends updates
// app receives updates and triggers repaint
ctx.request_repaint();
```

#### Translation

Flutter often hides state ownership behind widget lifecycles. egui makes state ownership explicit and local to the app or shared modules.

---

### Forms and input

#### Flutter
- `TextField`
- `TextFormField`
- `Form`
- validation callbacks
- `FocusNode`

#### egui
- `TextEdit::singleline`, `TextEdit::multiline`
- `ui.label`, `ui.checkbox`, `ui.radio_value`, `ui.slider`
- custom validation logic

```rust
ui.horizontal(|ui| {
    ui.label("Email:");
    ui.text_edit_singleline(&mut self.email);
});

if ui.button("Submit").clicked() {
    if self.email.contains('@') {
        // valid
    }
}
```

#### Translation

In Flutter, form widgets often come with a more framework-managed validation flow. In egui, validation is usually a direct `if` check in application logic.

---

### Buttons and interaction

#### Flutter
- `ElevatedButton`, `TextButton`, `IconButton`, `InkWell`
- callbacks and `onPressed`

#### egui
- `ui.button()`, `ui.small_button()`, `ui.add(Button::new(...))`
- `response.clicked`, `response.hovered`, `response.dragged`

```rust
if ui.button("Save").clicked() {
    self.save();
}
```

#### Translation

Flutter interactions are mostly declarative callback-driven. egui turns interaction into `Response` objects that let you inspect click, hover, drag, and focus state.

---

### Lists and scrolling

#### Flutter
- `ListView`
- `GridView`
- `CustomScrollView`

#### egui
- `ScrollArea::vertical()`
- loops with `ui.horizontal_wrapped` or `ui.vertical`
- custom virtualization if needed

```rust
egui::ScrollArea::vertical().show(ui, |ui| {
    for item in &self.items {
        ui.horizontal(|ui| {
            ui.label(item);
            if ui.button("Open").clicked() {
                // logic
            }
        });
    }
});
```

#### Translation

Flutter lists often emphasize widget composition and scroll performance at the framework level. egui leverages simple loops and explicit scroll container composition.

---

### Routing and navigation

#### Flutter
- `Navigator`
- `MaterialPageRoute`
- `RouteSettings`
- `push`, `pop`

#### egui
- `enum Screen { Home, Settings, About }`
- state variable or enum-driven UI switch
- custom modal and panel patterns

```rust
enum Screen {
    Home,
    Settings,
}

struct App {
    screen: Screen,
}

impl eframe::App for App {
    fn update(&mut self, ctx: &egui::Context, _frame: &mut eframe::Frame) {
        match self.screen {
            Screen::Home => {
                // home screen
            }
            Screen::Settings => {
                // settings screen
            }
        }
    }
}
```

#### Translation

In Flutter, navigation is part of the framework. In egui, navigation is often represented as app state and feature-specific UI branches.

---

### Styling and design systems

#### Flutter
- `ThemeData`
- `TextTheme`
- `ColorScheme`
- `IconTheme`

#### egui
- `egui::Visuals`
- `ui.style_mut()`
- `egui::FontId`
- `egui::Color32`
- custom theme structs

```rust
let mut style = (*ctx.style()).clone();
style.spacing.button_padding = egui::vec2(12.0, 6.0);
ctx.set_style(style);
```

#### Translation

Flutter has a richer, framework-native theme system. egui themes are lighter but powerful for custom visual identity.

---

### Async and background work

#### Flutter
- `FutureBuilder`
- `StreamBuilder`
- `async/await`
- `Timer`, `Delayed` etc.

#### egui
- `tokio`, `async-std`, `futures`
- `mpsc` or `watch` channels
- call `ctx.request_repaint()` when async state changes

```rust
let (tx, rx) = std::sync::mpsc::channel();

std::thread::spawn(move || {
    // work
    tx.send("done").unwrap();
});
```

#### Translation

In Flutter, async UI updates are often coupled to widget lifecycle. In egui, your app state owns the update loop, and async tasks inform the UI when something changes.

---

### Animation

#### Flutter
- `AnimationController`
- `Tween`
- `AnimatedBuilder`
- `Hero`, `Curves`

#### egui
- custom time-based animation with `Instant`
- `egui::Context::request_repaint()`
- interpolation math and redraw loops

```rust
let elapsed = self.start.elapsed().as_secs_f32();
let t = (elapsed * 0.5).sin() * 0.5 + 0.5;
```

#### Translation

Flutter animation is framework-assisted and builder-oriented. egui animation is typically more explicit and hand-rolled with the time domain and repaints.

---

### Testing

#### Flutter
- widget tests
- integration tests
- golden tests

#### egui
- unit tests for logic and state transitions
- app-state tests
- integration tests around pure UI logic

#### Translation

Flutter encourages UI testing at the widget layer. egui tends to lean on logic-level tests and custom state-driven validation because the UI is immediate and simpler.

---

## A "Flutter-to-egui" cheat sheet

| If you know Flutter... | Think in egui as... |
| --- | --- |
| `setState` | mutate app state and repaint |
| `build()` | `update()` emits UI every frame |
| `Widget` | list of immediate draw commands |
| `Scaffold` | `CentralPanel` + menus + side panels |
| `Container` | `ui.allocate_ui` and spacing |
| `TextField` | `TextEdit` + local `String` |
| `ListView` | `ScrollArea` + looped list |
| `ThemeData` | `Visuals` + style mutations |
| `FutureBuilder` | `async` task + state channel |
| `Navigator` | `enum Screen` + state machine |
| `Form` | custom validation logic in UI flow |
| `AnimationController` | time-based interpolation and repaint |
| `Provider` | app state + shared structs |

---

## Practical project structure

A Flutter developer working in egui often organizes code like this:

```text
src/
  main.rs
  app.rs
  state.rs
  screens/
    home.rs
    settings.rs
  widgets/
    button_row.rs
    stats_card.rs
  models/
    item.rs
  theme.rs
```

This mirrors the transition from widget-centric structure to app-centric structure.

---

## Typical translation examples

### Example 1: Flutter counter app

Flutter:

```dart
class MyHomePage extends StatefulWidget {
  @override
  State<MyHomePage> createState() => _MyHomePageState();
}

class _MyHomePageState extends State<MyHomePage> {
  int _counter = 0;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Center(
        child: Column(
          children: [
            Text('$_counter'),
            ElevatedButton(
              onPressed: () => setState(() => _counter++),
              child: Text('Add'),
            )
          ],
        ),
      ),
    );
  }
}
```

egui:

```rust
struct MyApp {
    counter: i32,
}

impl eframe::App for MyApp {
    fn update(&mut self, ctx: &egui::Context, _frame: &mut eframe::Frame) {
        egui::CentralPanel::default().show(ctx, |ui| {
            ui.vertical_centered(|ui| {
                ui.label(self.counter.to_string());
                if ui.button("Add").clicked() {
                    self.counter += 1;
                }
            });
        });
    }
}
```

### Example 2: Flutter form field

Flutter:

```dart
TextField(
  decoration: InputDecoration(labelText: 'Name'),
  onChanged: (value) => name = value,
)
```

egui:

```rust
ui.label("Name");
ui.text_edit_singleline(&mut self.name);
```

---

## Design philosophy comparison

### Flutter strengths
- polished material design ecosystem
- rich widget library and tree composition
- strong developer ergonomics for app UI
- mature reactive patterns

### egui strengths
- low overhead and fast UI development
- Rust-native safety and performance
- immediate mode simplicity
- excellent for desktop and tool-like interfaces

### Best-fit translations

- Flutter dashboards -> egui panels and data-heavy desktop UI
- Flutter forms -> egui input editors with custom validation
- Flutter mobile prototypes -> egui desktop or cross-platform UIs
- Flutter app state patterns -> Rust app state and async message loops

---

## Common pitfalls for Flutter developers switching to egui

1. Expecting a retained widget tree
   - egui does not keep a widget tree in the same way.

2. Re-rendering everything manually
   - In egui, you usually re-run the UI code each frame; state drives the result.

3. Treating callbacks like Flutter-style widget callbacks only
   - In egui, the response object is the direct interaction signal.

4. Ignoring app ownership and state flow
   - In Rust, data ownership and mutation boundaries matter.

5. Overusing global state
   - Keep state explicit and close to the app logic.

---

## Recommended mental model

Think of it like this:

- Flutter: a declarative UI model with a retained widget graph.
- egui: a state-driven immediate UI loop where you emit widgets from current state.

The art is not copying Flutter exactly. The art is understanding the shape of your app logic and re-expressing it in the idioms that fit Rust + egui.

---

## Quick reference

```text
Flutter concept -> Rust/egui concept
------------------------------------
StatefulWidget  -> app struct + update loop
BuildContext    -> UI context / mutable state access
Widget tree     -> instant UI emission each frame
setState        -> mutate state and trigger repaint
ThemeData       -> Visuals and style tweaks
FutureBuilder   -> async tasks + message channel
Navigator       -> enum-screen app state
ListView        -> ScrollArea + loop
Button          -> ui.button() + Response
TextField       -> TextEdit + String state
Layout          -> ui.horizontal(), vertical(), allocate_ui()
```

---

## Final takeaway

Flutter developers already understand app composition, layout, hierarchy, state, and event-driven interactions. Those skills translate well to egui once you shift from "widget tree thinking" to "state + immediate UI loop thinking".

This is not a replacement for Flutter. It is a translation layer: a map from your existing design instincts to the Rust and egui ecosystem.

---

## Contributing

Add comparisons, code snippets, and examples here as this catalog grows.

Suggested additions:
- Material widgets -> egui equivalents
- navigation patterns
- data table patterns
- form validation patterns
- async state synchronization
- cross-platform patterns

---

## License

This project is intended as a conceptual learning catalog. Use it as a practical reference and extension point for your own Rust/egui work.
