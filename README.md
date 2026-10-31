# Dart/Flutter to Rust/egui: Technical Translation Catalog

A technical field guide for mapping Dart/Flutter and Rust/egui mental models.

This repository is intended as a practical translation catalog: a reference for understanding how concepts from Dart/Flutter map onto Rust/egui architecture, rendering, input, state, layout, async work, and application structure.

The goal is to help Dart/Flutter developers map their existing mental model onto Rust/egui without pretending that the two frameworks are identical. The translation is conceptual and architectural:

- same problem domain
- different runtime model
- different ownership and scheduling rules

---

## 1. Executive summary

Dart/Flutter and Rust/egui serve the same high-level purpose: interactive UI generation. They do this through different runtime models.

Flutter is a retained, declarative UI framework built around a widget tree, element tree, render objects, and rebuild scheduling. Developers describe UI by composition, and the framework reconciles the tree against the previous render pass.

egui is an immediate-mode GUI library. Developers emit UI commands each frame from the current app state. User interactions are interpreted through response objects, and the app decides when to re-render and how to respond to inputs.

The crucial translation is not:

- "Flutter widget X becomes egui widget Y."

The crucial translation is:

- "Dart/Flutter state + rebuild model becomes Rust state + frame-driven UI emission."

In practical terms, the mental model shifts from:

- widget tree + diff + rebuild

to:

- app state + frame loop + immediate UI output + `Response` interpretation.

This is the primary bridge between Dart/Flutter expertise and productivity in Rust/egui.

---

## 2. The core mental model difference

### Dart/Flutter: retained widget graph

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
- the framework manages reconciliation
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

- state lives in the app struct
- each frame emits the current UI
- input is consumed through `Response` values
- repaint is explicit and intentional

### 2.1 Edge cases that feel different in practice

The basic translation works, but real-world UI bugs usually show up in edge cases where framework identity, lifecycle, and timing matter.

#### Dart/Flutter edge cases

- List reordering without keys: the framework can attach state to the wrong item when items move, insert, or delete.
- Rebuild loops caused by `setState` inside `build`, or by provider/store churn that triggers repeated refreshes.
- Build-time side effects: calling `setState` while a build is in progress or from a lifecycle callback can create difficult-to-debug loops.
- Async cancellation: `FutureBuilder`/`StreamBuilder` can emit after disposal; stale updates may need `mounted` checks or canceled subscriptions.
- Focus and text controllers: editing state is often split between widget state, `TextEditingController`, and the current focus tree.
- Theme, locale, and media-query changes: a configuration change can cascade into large rebuild trees and expensive recomputation.
- Dependencies and inherited state: a widget may read context from a parent unexpectedly, making architecture and testing harder.
- `RenderFlex` overflow and unbounded layout: nested rows/columns and scroll containers often expose intrinsic sizing problems.
- Hero and route transition timing: animations or navigation can end with stale routes or disappearing state.
- Gesture conflicts: taps, drags, and scroll gestures can fight each other if event handling is not explicit.
- Keyboard composition issues: IME input, emoji, and mobile text fields can produce surprising cursor and composition state behavior.
- Animation disposal and ticker lifecycle: a screen can be rebuilt while an animation is still active or disposed.
- Scroll restoration and page storage: reusing a widget tree can preserve stale scroll positions or selected items.
- Accessibility semantics timing: screen-reader announcements and focus changes can lag behind visual updates.

#### Rust/egui edge cases

- No retained widget tree: state is not automatically reattached to widgets, so identity must be managed deliberately via data and `egui::Id`.
- `egui::Id` collisions: reused IDs across panels or screens can cause stale widget state and unexpectedly shared controls.
- Input consumption timing: a `Response` is meaningful only for the current frame; reusing it later is a common source of stale-UI logic bugs.
- Repaint scheduling: if the app mutates state without `ctx.request_repaint()`, the UI may look frozen until another event causes a refresh.
- Layout without a fixed widget tree: overflow, clipping, and scrolling are usually solved by explicit `ScrollArea`, `allocate_ui`, or custom layout helpers.
- Async ownership: channels, watchers, and worker threads must be carefully integrated with the frame loop, or tasks may update state after the UI has moved on.
- Borrowing inside the UI closure: the immediate-mode model is ergonomic, but Rust borrowing rules can feel stricter when a closure mutates state while also reading it for rendering.
- Background-thread state mutation: updating shared state from a worker thread while the UI frame is rendering can cause race conditions or inconsistent output.
- Popup and modal focus ordering: custom windows and popups can steal focus in ways that feel surprising if the app is not managing interaction flow explicitly.
- Custom painting ordering: overlapping shapes, overlays, and labels can layer incorrectly if draw order is not controlled.
- DPI and scaling mismatches: window resizing, high-DPI displays, and custom font sizing can desynchronize layout and painting.
- High-frequency repaint loops: a loop that repaints on every frame without throttling can waste CPU and make simple interfaces feel sluggish.
- Multi-window and multi-panel state bugs: UI state can be accidentally shared across windows, side panels, or tabs.
- IME, clipboard, and multiline text issues: text editing needs explicit handling for paste, composition boundaries, and caret position updates.

#### The unifying rule

Both systems have the same class of problems, but they solve them with different primitives:

- Flutter keeps identity inside the framework and exposes it through widget tree semantics.
- egui keeps identity as a design choice of the app author and exposes it through state, IDs, and frame-driven logic.

In other words, Flutter hides a lot of identity management; egui requires the app to do it explicitly.

### 2.2 Shared edge-case taxonomy

The strongest way to translate these bugs is to look past syntax and map the underlying pattern.

- Identity management: `Key`, widget identity, and `egui::Id` all exist to preserve state across updates.
- Update timing: `setState` after build, stale async callbacks, and stale `Response` values all represent timing bugs.
- Asynchronous completion: `FutureBuilder`/`StreamBuilder` and channels both need lifecycle-aware handling.
- Layout constraints: `RenderFlex`, scroll areas, and explicit `allocate_ui()` all deal with the same stress under resize and overflow.
- Input ordering: click/drag/hover and focus transitions must be interpreted in the correct frame.
- State ownership: when data crosses screens, modals, layers, or windows, ownership must be explicit in both frameworks.
- Repaint and scheduling: visual output is either driven by rebuild scheduling or by frame repaint requests.
- Multi-layer interaction: navigation, overlays, dialogs, and popups all fail when states are not isolated properly.

---

## 3. Cross-language edge-case translation map

| Problem pattern | Dart/Flutter mental model | Rust/egui equivalent | Translation rule |
| --- | --- | --- | --- |
| unstable list identity | `Key` / `ValueKey` / `UniqueKey` | row state + explicit `egui::Id` or data ordering | preserve identity across frame updates |
| rebuild loop | repeated `setState` / provider churn | repeated state mutation + repaint requests | avoid work that retriggers itself |
| state update after async completion | `FutureBuilder` / `StreamBuilder` / `mounted` checks | channel/receiver + repaint guard | do not mutate stale UI from dead tasks |
| focus and input capture | `TextEditingController` + focus node | `TextEdit` state + `ui.memory()` + `Response` | keep current interaction state in app-owned data |
| modal flow | route builder + `Navigator` + `showDialog` | state enum + `egui::Window` + click handlers | turn navigation into state transitions |
| layout overflow | constraints, `RenderFlex`, `Expanded`, `Flexible` | `ScrollArea`, `ui.allocate_ui`, custom sizing | model layout as explicit allocation rules |
| route transition timing | navigator animation and route disposal | modal/window lifecycle + explicit state transitions | isolate old state from new state |
| animation | `AnimationController` + `Tween` + rebuild | time-based state + `ctx.request_repaint()` | drive motion from frame time, not widget identity |
| shared data across widgets | inherited widgets / provider / context | app state, shared structs, message channels | move shared state into explicit owners |
| gesture conflict | tap/drag/scroll event ordering | hover/click/drag response ordering | handle interaction ordering explicitly |
| accessibility and focus semantics | semantics tree + focus management | input state + UI memory + deliberate focus handling | keep mental model and visual state synchronized |
| overlay ordering | route stack and overlay management | modal windows and custom painter layers | manage layer order as explicit state |

The bridge between the two worlds is not “same API, different syntax.” It is “same UI problem, different runtime contract.”

---

## 4. High-confidence translation matrix

| Dart/Flutter concept | Rust / egui equivalent | Practical translation |
| --- | --- | --- |
| `runApp` | `eframe::run_native` | Bootstraps the UI program |
| `MaterialApp` | root `eframe::App` | App shell |
| `StatefulWidget` | app struct / screen state | Explicit state ownership |
| `build()` | `App::update()` | UI generation each frame |
| `setState` | mutate field + repaint | Frame-driven re-render |
| `BuildContext` | `&egui::Context`, `&mut Ui` | Explicit render context |
| `Widget` | immediate UI commands | Not an identity-retaining tree |
| `Row` / `Column` | `ui.horizontal()` / `ui.vertical()` | Layout closures |
| `Center` | `ui.centered_and_justified()` / `ui.vertical_centered()` | Layout helper |
| `Padding` | `ui.add_space()`, spacing, `ui.scope` | Explicit spacing |
| `Text` | `ui.label()` | Immediate text output |
| `TextField` | `ui.text_edit_singleline()` | Bound to `String` |
| `Button` | `ui.button()` | `Response.clicked()` |
| `ListView` | `ScrollArea::vertical()` | Scroll container + loop |
| `GridView` | `ui.columns()` / loops | Manual grid composition |
| `Expanded` / `Flexible` | allocation and layout constraints | `ui.allocate_ui()` / sizing |
| `Container` | `ui.scope()` or custom helper | Region-based rendering |
| `FutureBuilder` | async task + channel + repaint | Message-driven state |
| `StreamBuilder` | `watch` or `mpsc` | Stream updates into app state |
| `Provider` | app state or shared context | Explicit ownership |
| `InheritedWidget` | root/shared state | Shared state model |
| `ThemeData` | `egui::Visuals` | Visual style system |
| `Navigator` | `enum Screen` / route stack | State machine |
| `AnimatedBuilder` | `Instant` + interpolation + repaint | Custom animation loop |
| `GestureDetector` | `Response` hover/drag/click | Direct input handling |
| `AlertDialog` | modal UI state or custom overlay | App state-driven modal |
| `Form` | validation in app state | Immediate UI validation |
| `DataTable` | custom table row loop | Rendered from data state |
| `ListTile` | `ui.horizontal()` + labels/actions | Manual row composition |
| `Card` | `ui.group()` / custom card helper | Composed UI container |

---

## 5. App lifecycle and state translation

### 5.1 Dart/Flutter app structure

Flutter apps usually begin with a root widget and a rebuild-driven tree:

- root application
- stateful widgets
- `build()` returns UI as composition
- `setState` re-renders part of the tree

### 5.2 egui app structure

egui apps typically revolve around one app struct with frame-based rendering:

```rust
struct MyApp {
    title: String,
    counter: i32,
    is_expanded: bool,
}

impl eframe::App for MyApp {
    fn update(&mut self, ctx: &egui::Context, _frame: &mut eframe::Frame) {
        egui::CentralPanel::default().show(ctx, |ui| {
            ui.heading(&self.title);
            ui.checkbox(&mut self.is_expanded, "Expanded");

            if ui.button("Increment").clicked() {
                self.counter += 1;
            }

            ui.label(format!("Counter: {}", self.counter));
        });
    }
}
```

The translation is straightforward:

- Dart/Flutter widget tree -> Rust state + immediate UI output
- Dart/Flutter rebuild -> egui frame update
- Dart/Flutter `setState` -> mutate a field + re-render on next frame

---

## 6. Widget-to-UI mapping catalog

### Basic widgets

| Dart/Flutter | egui equivalent |
| --- | --- |
| `Text` | `ui.label()` |
| `RichText` | custom label with formatting or `egui::TextFormat` |
| `Icon` | custom font icon or label glyph |
| `Container` | `ui.group()` or custom helper |
| `SizedBox` | `ui.add_sized([w, h], ...)` |
| `Padding` | `ui.add_space()`, spacing groups |
| `Center` | `ui.centered_and_justified()` |
| `Align` | custom alignment layout |
| `Expanded` | layout allocation / `allocate_ui()` |
| `Wrap` | `ui.horizontal_wrapped()` |
| `Spacer` | `ui.add_space()` |
| `Divider` | `ui.separator()` |

### Input widgets

| Dart/Flutter | egui equivalent |
| --- | --- |
| `TextField` | `ui.text_edit_singleline()` / `text_edit_multiline()` |
| `TextFormField` | custom form state + validation |
| `Checkbox` | `ui.checkbox()` |
| `Switch` | custom toggle or checkbox style |
| `Radio` | `ui.radio_value()` |
| `Slider` | `ui.add(egui::Slider::new(...))` |
| `DropdownButton` | `ComboBox` / menu / custom popup |
| `TextButton` | `ui.button()` |
| `ElevatedButton` | styled button |
| `OutlinedButton` | styled button variant |
| `IconButton` | custom icon button |

### Layout / composition

| Dart/Flutter | egui equivalent |
| --- | --- |
| `Column` | `ui.vertical()` |
| `Row` | `ui.horizontal()` |
| `Stack` | overlay using rect/painter |
| `ListView` | `ScrollArea::vertical()` |
| `GridView` | loop + row/column layout |
| `PageView` | stateful page index |
| `Flow` | `ui.horizontal_wrapped()` |
| `CustomScrollView` | scroll area with grouped sections |
| `AnimatedContainer` | interpolation + repaint |

### Navigation / structure

| Dart/Flutter | egui equivalent |
| --- | --- |
| `Scaffold` | `CentralPanel` + `TopBottomPanel` + side panels |
| `AppBar` | top toolbar panel |
| `Drawer` | side panel or modal overlay |
| `BottomNavigationBar` | bottom panel with buttons |
| `TabBar` | tabs by state |
| `Navigator` | route enum / screen state |
| `AlertDialog` | custom modal window |
| `SnackBar` | toast state overlay |
| `ModalBottomSheet` | custom modal panel |

### Data / content UI

| Dart/Flutter | egui equivalent |
| --- | --- |
| `Card` | `ui.group()` / custom card helper |
| `ListTile` | horizontal row with label + actions |
| `DataTable` | custom rows in a `ScrollArea` |
| `Table` | manual row/column rendering |
| `Chip` | custom label/button |
| `Badge` | custom colored label |

---

## 7. Pattern translations that matter in practice

### 7.1 Dialogs and modal flows

Flutter uses `showDialog` with a builder function. In egui, modal behavior is expressed as app state plus an `egui::Window`.

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

Translation:

- `showDialog` -> modal state + `egui::Window`
- dialog actions -> click handlers that mutate state

### 7.2 Forms and validation

Flutter form patterns usually combine controllers and validation logic. In egui, the common approach is to keep the form values in app state and validate on submit.

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

### 7.3 Data tables

Flutter uses `DataTable` for structured records. In egui, the equivalent is usually a manual row loop with a scroll area.

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

### 7.4 Async + reactive state updates

Flutter commonly uses `FutureBuilder` or `StreamBuilder` to react to asynchronous data. In egui, the equivalent is usually a channel-based or watch-based state update + repaint.

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

This is roughly the same design intention as a Flutter `FutureBuilder`, but expressed with explicit Rust state and runtime control.

### 7.5 Animation and motion

Flutter relies on `AnimationController`, `Tween`, and rebuild-driven animation. In egui, animation is usually implemented with time-state and repaint requests.

```rust
struct AnimationState {
    start: std::time::Instant,
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

- `AnimationController` -> time-based state + explicit repaint
- `Tween` -> interpolation logic in Rust
- `AnimatedBuilder` -> frame-by-frame rendering from live state

---

## 8. State architecture patterns in real applications

A Dart/Flutter developer usually thinks in terms of widget trees. The Rust equivalent is usually an explicit state model.

### Common patterns

1. Single app state object
   - simplest pattern
   - works well for desktop tools and medium-sized apps
   - all state lives in the app struct

2. Screen-level state
   - each screen or panel owns local state
   - good for isolation and feature boundaries

3. Domain state + view state separation
   - business data and UI state are separate
   - easier to test and reason about

4. Message-driven state
   - app handles events with enum variants
   - rendering is a pure function of current state

```rust
enum AppEvent {
    Increment,
    ToggleExpanded,
    LoadData,
    DataLoaded(String),
}
```

This pattern maps naturally to the mental model of a Dart/Flutter app that reacts to user events, but it is implemented with explicit Rust enums and state transitions instead of widget rebuild callbacks.

---

## 9. Designing a Dart/Flutter-like abstraction over egui

This is the most valuable next step beyond raw translation.

The key idea is not to clone Flutter exactly. It is to create a Dart/Flutter-like developer interface that compiles down to egui runtime primitives.

### Why this matters

- familiar APIs for Dart/Flutter developers
- preserves Rust ownership rules
- keeps egui as the real rendering runtime
- lowers migration friction for teams moving from Dart/Flutter

### Abstraction layers worth designing

1. egui runtime layer
   - `Context`, `Ui`, `Response`, `Rect`, inputs

2. Dart/Flutter-like app shell
   - `App`, `MaterialApp`, `Scaffold`

3. Stateful widget layer
   - `State<T>`, `StatefulWidget`, `BuildContext`

4. Layout layer
   - `Column`, `Row`, `Stack`, `Padding`, `SizedBox`, `Center`

5. Navigation layer
   - route enum / route stack / screen state

6. Async layer
   - `FutureBuilder`-like and `StreamBuilder`-like patterns

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

This is intentionally “Dart/Flutter-like” at the API boundary, while using immediate-mode rendering under the hood.

### Design principle

The framework should mimic the shape of Flutter, but not its internal runtime architecture.

That is the sweet spot: familiar mental structure, Rust-native execution model.

---

## 10. Final takeaway

The central truth is simple:

Dart/Flutter developers already know the difficult parts of UI development: composition, layout, state, interaction, navigation, theming, forms, async updates, and architecture. Those ideas all transfer to egui, but the runtime model changes.

The only thing that changes is the runtime model:

- Dart/Flutter: widget tree + rebuild system
- egui: app state + immediate UI emissions + response-driven input

If this repository teaches that shift clearly, it will be highly valuable for developers moving from Dart/Flutter into Rust GUI development.

---

## 11. License

This repository is intended as a learning and translation reference for developers moving from Dart/Flutter to Rust/egui.
