# Dart/Flutter to Rust/egui: Technical Translation Catalog

A technical field guide for mapping Flutter and Dart mental models to Rust and egui.

This repository is intended as a practical translation catalog: a reference for understanding how concepts from Flutter/Dart map onto Rust/egui architecture, rendering, input, state, layout, async work, and application structure.

The goal is to help Flutter developers map their existing mental model onto Rust/egui without pretending that the two frameworks are the same. The translation is conceptual and architectural: understand the problem, then express the solution in the idioms that fit egui and Rust.

---

## 1. Executive summary

Flutter and egui serve the same high-level purpose: interactive UI generation. They do this with completely different execution models.

Flutter is a retained, declarative UI framework built around a widget tree, element tree, render objects, and rebuild scheduling. Developers describe UI by composition, and the framework reconciles those descriptions into painting.

egui is an immediate-mode GUI library. Developers emit UI commands each frame from the current state, the backend renders those commands, and user interactions are interpreted using response objects.

The crucial translation is not:

- “Flutter widget X becomes egui widget Y.”

The crucial translation is:

- “Flutter state + rebuild model becomes Rust state + frame-driven UI emission.”

In practical terms, the mental model shifts from:

- widget tree + diff + rebuild

to:

- app state + frame loop + immediate UI output + `Response` interpretation.

That is the primary bridge between Dart/Flutter expertise and Rust/egui productivity.

---

## 2. The core mental model difference

### Flutter: retained widget graph

Flutter builds a conceptual tree of widgets, and the framework manages identity, rebuild timing, and reconciliation.

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
            )
          ],
        ),
      ),
    );
  }
}
```

The mental model is:

- widgets are definitions
- framework renders and reconciles them
- state changes trigger rebuilds

### egui: immediate UI emitted from state

Rust/egui is not a retained widget tree; it is a state-driven UI loop.

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

The mental model is:

- state is owned by the app
- each frame emits current UI
- input is read from `Response` objects
- repaint is intentional

---

## 3. High-confidence translation matrix

| Flutter / Dart concept | Rust / egui equivalent | Practical translation |
| --- | --- | --- |
| `runApp` | `eframe::run_native` | Bootstraps the UI program |
| `MaterialApp` | root `eframe::App` | App shell |
| `StatefulWidget` | app struct / screen state | Explicit state ownership |
| `build()` | `App::update()` | UI generation each frame |
| `setState` | mutate field + repaint | Re-render loop |
| `BuildContext` | `&egui::Context`, `&mut Ui` | Explicit context |
| `Widget` | immediate UI commands | Not an identity-retaining tree |
| `Row` / `Column` | `ui.horizontal()` / `ui.vertical()` | Layout closures |
| `Center` | `ui.centered_and_justified` / `ui.vertical_centered` | Layout helper |
| `Padding` | `ui.add_space`, spacing, `ui.scope` | Explicit spacing |
| `Text` | `ui.label()` | Immediate text output |
| `TextField` | `ui.text_edit_singleline()` | Bound to `String` |
| `Button` | `ui.button()` | `Response.clicked()` |
| `ListView` | `ScrollArea::vertical()` | Scroll container + loop |
| `GridView` | `ui.columns` / loops | Manual grid composition |
| `Expanded` / `Flexible` | allocation and layout constraints | `ui.allocate_ui` / sizing |
| `Container` | `ui.scope` or custom helper | Region-based rendering |
| `FutureBuilder` | async task + channel + repaint | Message-driven state |
| `StreamBuilder` | `watch` or `mpsc` | Stream updates into app state |
| `Provider` | app state or shared context | App-wide explicit state |
| `InheritedWidget` | state shared by root | Clear ownership model |
| `ThemeData` | `egui::Visuals` | Visual style system |
| `Navigator` | `enum Screen` / route stack | State machine |
| `AnimatedBuilder` | `Instant` + time interpolation | Custom animation loop |
| `GestureDetector` | `Response` hover/drag/click | Direct input handling |
| `AlertDialog` | modal UI state or custom overlay | App state-driven modal |
| `Form` | custom validation logic | Immediate UI validation |
| `DataTable` | custom table with rows loop | Rendered from data state |
| `ListTile` | `ui.horizontal` + `ui.label` | Manual row composition |
| `Card` | `ui.group()` / custom card function | Composed UI container |

---

## 4. Detailed pattern translation

### 4.1 App lifecycle and root architecture

Flutter:

- `runApp`
- root widget tree
- `StatefulWidget` hierarchy
- build lifecycle

Rust + egui:

- `eframe::run_native`
- root app struct implementing `eframe::App`
- explicit state fields in the app struct
- `update()` is the render lifecycle for each frame

```rust
struct MyApp {
    name: String,
    counter: i32,
}

impl eframe::App for MyApp {
    fn update(&mut self, ctx: &egui::Context, _frame: &mut eframe::Frame) {
        egui::CentralPanel::default().show(ctx, |ui| {
            ui.horizontal(|ui| {
                ui.label("Name:");
                ui.text_edit_singleline(&mut self.name);
            });

            if ui.button("Increment").clicked() {
                self.counter += 1;
            }

            ui.label(format!("Counter: {}", self.counter));
        });
    }
}
```

### 4.2 Stateful semantics

Flutter uses widgets that maintain internal state and trigger rebuilds. egui typically uses a root app state and direct mutation.

This is the single most important conceptual change.

```rust
struct AppState {
    text: String,
    expanded: bool,
}

impl eframe::App for AppState {
    fn update(&mut self, ctx: &egui::Context, _frame: &mut eframe::Frame) {
        egui::CentralPanel::default().show(ctx, |ui| {
            ui.checkbox(&mut self.expanded, "Expanded");
            ui.text_edit_singleline(&mut self.text);
        });
    }
}
```

In Flutter, the state is often owned by widget objects. In egui, it is owned by the app and mutated directly.

---

## 5. Widget-to-UI mapping catalog

This is the practical cheat sheet many people want.

### Basic widgets

| Flutter | egui equivalent |
| --- | --- |
| `Text` | `ui.label()` |
| `RichText` | custom label with formatting or `egui::TextFormat` |
| `Icon` | custom icon widget or `ui.label("icon")` with font glyph |
| `Container` | `ui.group()` or custom layout helper |
| `SizedBox` | `ui.add_sized([w, h], ...)` |
| `Padding` | `ui.add_space()`, custom `ui.scope` wrappers |
| `Center` | `ui.centered_and_justified` or `ui.vertical_centered` |
| `Align` | `ui.with_layout` / custom alignment |
| `Expanded` | layout allocation and `ui.allocate_ui` |
| `Wrap` | `ui.horizontal_wrapped` |
| `Spacer` | `ui.add_space` |
| `Divider` | `ui.separator()` |

### Inputs

| Flutter | egui equivalent |
| --- | --- |
| `TextField` | `ui.text_edit_singleline` / `text_edit_multiline` |
| `TextFormField` | custom form state + input field |
| `Checkbox` | `ui.checkbox` |
| `Switch` | `ui.checkbox` or custom toggle |
| `Radio` | `ui.radio_value` |
| `Slider` | `ui.add(egui::Slider::new(...))` |
| `DropdownButton` | `ComboBox` / `ui.menu` / custom dropdown component |
| `Form` | custom app state + validation logic |
| `TextButton` | `ui.button` |
| `ElevatedButton` | `ui.button` with styling |
| `OutlinedButton` | custom button styling |
| `IconButton` | custom button with icon glyph |

### Layout

| Flutter | egui equivalent |
| --- | --- |
| `Column` | `ui.vertical` |
| `Row` | `ui.horizontal` |
| `Stack` | custom overlay using `ui.allocate_rect` and painter |
| `ListView` | `ScrollArea::vertical` |
| `GridView` | `ui.columns` or manual row/column grid |
| `PageView` | custom state with current page index |
| `Flow` | `ui.horizontal_wrapped` / custom wrapped layout |
| `CustomScrollView` | `ScrollArea` with grouped sections |
| `AnimatedContainer` | custom interpolation + repaint |

### Navigation + structure

| Flutter | egui equivalent |
| --- | --- |
| `Scaffold` | `CentralPanel` + `TopBottomPanel` + custom sidebars |
| `AppBar` | top panel or custom toolbar |
| `Drawer` | side panel or modal overlay |
| `BottomNavigationBar` | bottom panel with buttons |
| `TabBar` | tabs via selector UI |
| `Navigator` | enum screen or route stack |
| `AlertDialog` | custom modal panel or overlay |
| `SnackBar` | bottom toast state overlay |
| `ModalBottomSheet` | custom modal panel |

### Data / content

| Flutter | egui equivalent |
| --- | --- |
| `Card` | `ui.group` / custom card helper |
| `ListTile` | `ui.horizontal` with label and action |
| `DataTable` | custom table rows in a `ScrollArea` |
| `Table` | manual row/column rendering |
| `Chip` | custom label/button group |
| `Badge` | custom colored label |

---

## 6. Advanced patterns and deeper translations

### 6.1 Dialogs and modal flows

Flutter:

```dart
showDialog(
  context: context,
  builder: (_) => AlertDialog(
    title: const Text('Delete?'),
    actions: [
      TextButton(onPressed: () => Navigator.pop(context), child: const Text('Cancel')),
      TextButton(onPressed: () => Navigator.pop(context), child: const Text('Delete')),
    ],
  ),
);
```

Rust/egui:

```rust
enum ModalState {
    None,
    ConfirmDelete,
}

struct App {
    modal: ModalState,
}

impl eframe::App for App {
    fn update(&mut self, ctx: &egui::Context, _frame: &mut eframe::Frame) {
        if matches!(self.modal, ModalState::ConfirmDelete) {
            egui::Window::new("Delete?")
                .collapsible(false)
                .show(ctx, |ui| {
                    ui.label("Are you sure?");
                    ui.horizontal(|ui| {
                        if ui.button("Cancel").clicked() {
                            self.modal = ModalState::None;
                        }
                        if ui.button("Delete").clicked() {
                            self.modal = ModalState::None;
                        }
                    });
                });
        }

        egui::CentralPanel::default().show(ctx, |ui| {
            if ui.button("Delete").clicked() {
                self.modal = ModalState::ConfirmDelete;
            }
        });
    }
}
```

The translation is:

- `showDialog` -> application modal state with `egui::Window`
- dialog actions -> button click handlers changing app state

### 6.2 Forms and validation

Flutter patterns combine controller state, validation, and field focus. egui usually uses `String` + validation on submit.

```rust
struct FormState {
    name: String,
    email: String,
    status: String,
}

impl FormState {
    fn validate(&self) -> Result<(), String> {
        if self.name.trim().is_empty() {
            return Err("Name cannot be empty".to_string());
        }
        if !self.email.contains('@') {
            return Err("Invalid email".to_string());
        }
        Ok(())
    }
}

impl eframe::App for FormState {
    fn update(&mut self, ctx: &egui::Context, _frame: &mut eframe::Frame) {
        egui::CentralPanel::default().show(ctx, |ui| {
            ui.label("Create profile");
            ui.text_edit_singleline(&mut self.name);
            ui.text_edit_singleline(&mut self.email);

            if ui.button("Save").clicked() {
                match self.validate() {
                    Ok(()) => self.status = "Saved".to_string(),
                    Err(err) => self.status = err,
                }
            }

            ui.label(&self.status);
        });
    }
}
```

### 6.3 Data tables

Flutter:

```dart
DataTable(
  columns: const [
    DataColumn(label: Text('Name')),
    DataColumn(label: Text('Score')),
  ],
  rows: [
    DataRow(cells: [DataCell(Text('A')), DataCell(Text('10'))]),
  ],
);
```

egui:

```rust
struct RowData {
    name: String,
    score: i32,
}

struct TableState {
    rows: Vec<RowData>,
}

impl eframe::App for TableState {
    fn update(&mut self, ctx: &egui::Context, _frame: &mut eframe::Frame) {
        egui::CentralPanel::default().show(ctx, |ui| {
            egui::ScrollArea::both().show(ui, |ui| {
                ui.horizontal(|ui| {
                    ui.label("Name");
                    ui.separator();
                    ui.label("Score");
                });

                for row in &self.rows {
                    ui.horizontal(|ui| {
                        ui.label(&row.name);
                        ui.separator();
                        ui.label(row.score.to_string());
                    });
                }
            });
        });
    }
}
```

Translation:

- `DataTable` -> manual rows with a scrollable container
- `DataColumn` -> column labels in header row
- `DataRow` -> loop over data rows

### 6.4 Animation and motion

Flutter often relies on `AnimationController`, `Tween`, and `AnimatedBuilder`. In egui, animation is usually implemented using time values and repaint scheduling.

Flutter:

```dart
AnimationController(
  duration: const Duration(seconds: 1),
  vsync: this,
);
```

Rust/egui:

```rust
struct AnimationState {
    start: std::time::Instant,
    amplitude: f32,
}

impl eframe::App for AnimationState {
    fn update(&mut self, ctx: &egui::Context, _frame: &mut eframe::Frame) {
        let elapsed = self.start.elapsed().as_secs_f32();
        let t = (elapsed * 1.5).sin() * 0.5 + 0.5;

        egui::CentralPanel::default().show(ctx, |ui| {
            let size = 100.0 + t * 80.0;
            let rect = egui::Rect::from_center_size(
                ui.max_rect().center(),
                egui::vec2(size, size),
            );
            ui.painter().circle_filled(rect.center(), size / 2.0, egui::Color32::WHITE);
        });

        ctx.request_repaint();
    }
}
```

Translation:

- `AnimationController` -> time-based state and repaint requests
- `Tween` -> interpolation logic using time values
- `AnimatedBuilder` -> UI rendered from interpolated state each frame

### 6.5 Async + reactive state updates

Flutter common pattern:

- `FutureBuilder` or `StreamBuilder`
- app reacts to state changes

Rust/egui equivalent:

- `mpsc` / `watch` channels
- `tokio` tasks or threads
- app updates the state and requests repaint

```rust
use std::sync::mpsc;

struct AsyncState {
    loading: bool,
    status: String,
    rx: Option<mpsc::Receiver<String>>,
}

impl AsyncState {
    fn request_update(&mut self) {
        let (tx, rx) = mpsc::channel();
        self.rx = Some(rx);
        self.loading = true;

        std::thread::spawn(move || {
            let payload = "Loaded".to_string();
            tx.send(payload).unwrap();
        });
    }
}

impl eframe::App for AsyncState {
    fn update(&mut self, ctx: &egui::Context, _frame: &mut eframe::Frame) {
        if let Some(rx) = &self.rx {
            if let Ok(value) = rx.try_recv() {
                self.status = value;
                self.loading = false;
                self.rx = None;
                ctx.request_repaint();
            }
        }

        egui::CentralPanel::default().show(ctx, |ui| {
            if self.loading {
                ui.spinner();
            } else {
                ui.label(&self.status);
            }

            if ui.button("Load").clicked() {
                self.request_update();
            }
        });
    }
}
```

This is a strong and direct translation from Flutter’s `FutureBuilder` mental model.

---

## 7. State architecture patterns in actual applications

A Flutter developer usually sees an application as a tree of widgets. The Rust equivalent is almost always an explicit state ecosystem.

### Common state architecture patterns

1. Single app state object
   - simplest approach
   - good for small/medium tools
   - all state lives in one app struct

2. Screen-level state
   - each screen or panel owns its state
   - good for feature isolation

3. Domain state + UI state separation
   - domain models and services are separate
   - UI state is the visible representation

4. Message-driven architecture
   - app receives events via enum variants
   - commands mutate state and render next frame

```rust
enum AppEvent {
    Increment,
    ToggleExpanded,
    LoadData,
    DataLoaded(String),
}
```

A Flutter-like app event system can be expressed cleanly in Rust.

---

## 8. Designing a Flutter-like abstraction over egui

This is the most valuable next step beyond plain translation.

The key idea is not to clone Flutter. It is to create a Flutter-like developer interface that compiles down to egui runtime primitives.

### Why the abstraction is useful

- Familiar API for Flutter developers
- preserves Rust ownership rules
- stays efficient because egui is still the real runtime
- reduces startup friction for teams migrating from Flutter

### Good abstraction layers

1. Raw egui runtime
   - `Context`, `Ui`, `Response`, `Visuals`, `Rect`, input handling

2. Flutter-like app shell
   - `App`, `MaterialApp`, `Scaffold`

3. Stateful widget layer
   - `State<T>`, `StatefulWidget`, `BuildContext`

4. Layout layer
   - `Column`, `Row`, `Stack`, `Padding`, `SizedBox`, `Center`

5. Navigation layer
   - route enums and screen states

6. Async layer
   - `FutureBuilder`, `StreamBuilder`-like builders

### Example conceptual API

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

This is intentionally “Flutter-like” at the API boundary, while still using immediate-mode rendering under the hood.

### Design principle

The framework should mimic the shape of Flutter, but not its internal runtime architecture.

That is the sweet spot.

---

## 9. Final takeaway

The central truth is simple:

Flutter developers already know the difficult parts of UI development: composition, layout, state, interaction, navigation, theming, forms, async updates, and architecture. Those ideas all transfer to Rust/egui.

The only thing that changes is the runtime model:

- Flutter: widget tree + rebuild system
- egui: app state + immediate UI emissions + response-driven input

If the repo teaches that shift clearly, it will be highly valuable to developers trying to move from Dart/Flutter into Rust GUI development.

---

## 10. License

This repository is intended as a learning and translation reference for developers moving from Dart/Flutter to Rust/egui.
