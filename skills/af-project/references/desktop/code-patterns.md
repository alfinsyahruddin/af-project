# Desktop Architecture & Code Patterns

This guide defines the architectural standards, layer responsibilities, GPUI view patterns, and inviolable code rules for building native desktop applications in Rust.

---

## 1. Architectural Layers & Responsibilities

The 4-crate workspace enforces unidirectional flow and strict separation of concerns:

```text
┌────────────────────────────────────────────────────────────────────────┐
│ <app>-desktop (GPUI & OS Presentation)                                 │
│ ├─ Main window, workspace layout, split panes, title bar               │
│ ├─ GPUI Kit controls, custom vector graphics, theme bindings           │
│ └─ Keyboard shortcuts, action dispatch, toast & modal dialogs          │
└──────────────────────────────────┬─────────────────────────────────────┘
                                   │ Dispatches Commands / Reads Arc Projections
                                   ▼
┌────────────────────────────────────────────────────────────────────────┐
│ <app>-app (Application Orchestration & Session State)                  │
│ ├─ SessionController owns domain state & command routing               │
│ ├─ Derives read-only Projections behind Arc slices                     │
│ └─ Defines SessionRepository port & serializable Snapshot DTOs         │
└──────────────────┬─────────────────────────────────┬───────────────────┘
                   │ Implements Port                 │ Executes Rules
                   ▼                                 ▼
┌────────────────────────────────────┐ ┌─────────────────────────────────┐
│ <app>-storage (Persistence)        │ │ <app>-core (Domain Engine)      │
│ ├─ Versioned JSON snapshot I/O     │ │ ├─ Deterministic business logic │
│ ├─ Background save coordinator     │ │ ├─ Checked arithmetic & rules   │
│ └─ Profile locking & recovery .bak │ │ └─ Pure state machines & models │
└────────────────────────────────────┘ └─────────────────────────────────┘
```

### Layer Responsibilities Matrix

| Crate | Permitted Dependencies | Forbidden Imports | Primary Role |
| :--- | :--- | :--- | :--- |
| `<app>-core` | None (pure standard library, optional mathematical crates) | GPUI, UI crates, OS clocks, filesystem, network, storage | Pure domain models, deterministic rules, state machines |
| `<app>-app` | `<app>-core`, serde, chrono, thiserror | GPUI, raw filesystem I/O, concrete database drivers | State orchestration, command execution, projection caching |
| `<app>-storage` | `<app>-app`, `<app>-core`, serde_json, fs | GPUI, views, UI models, network clients | Snapshot persistence, backup recovery, profile file locks |
| `<app>-desktop` | `<app>-app`, `<app>-storage`, `<app>-core`, `gpui-kit` | Business calculation shortcuts that bypass controller | View rendering, input handling, platform chrome, modals |

---

## 2. Inviolable Desktop Rules: Do's & Don'ts

| Area | Strictly Required (DO) | Invariant Violation (DON'T) |
| :--- | :--- | :--- |
| **Layer Isolation** | Maintain strict acyclic boundaries (`desktop -> app -> core`, `storage -> app`). | **Never** import GPUI or storage in `core`. Never import GPUI in `app`. |
| **Error Handling** | Return typed `Result<T, DomainError/AppError/StorageError>` and propagate with `?`. | **Never** call `.unwrap()` or `.expect()` in production paths. |
| **Thread Discipline** | Run all mutations on UI executor; run all disk I/O on background workers. | **Never** perform synchronous file or network I/O on the UI thread. |
| **Domain Arithmetic** | Checked integer arithmetic only for business/financial units (prices, IDs, counts). | **Never** use floating-point numbers (`f32`/`f64`) for financial or critical accounting. |
| **State Sharing** | Share projections with views as immutable `Arc<Projection>`. | **Never** wrap the entire state machine in a shared blocking mutex read by views. |
| **Toolchain & Locks** | Build using `--locked` with pinned `rust-toolchain.toml` and dependencies. | **Never** change toolchain versions or upgrade unpinned UI crates ad-hoc. |
| **Commit Style** | Use descriptive natural language summaries without conventional prefixes. | **Never** use `feat:`, `fix:`, `chore:` prefixes. |
| **Persistence Safety** | Write to `.tmp`, flush to disk, backup existing to `.recovery`, then atomic swap. | **Never** directly overwrite an existing snapshot without backup. |

---

## 3. GPUI View & Component Patterns

### Modern GPUI View Implementation
Views implement `gpui::Render` and consume immutable projections:

```rust
use gpui::prelude::*;
use gpui::{div, rgb, px, Render, ViewContext, WindowContext, InteractiveElement, StatefulInteractiveElement};
use gpui_kit::theme::Theme;
use std::sync::Arc;
use crate::app_model::AppModel;

pub struct OrderBookView {
    app_model: AppModel,
}

impl OrderBookView {
    pub fn new(app_model: AppModel, _cx: &mut ViewContext<Self>) -> Self {
        Self { app_model }
    }
}

impl Render for OrderBookView {
    fn render(&mut self, cx: &mut ViewContext<Self>) -> impl IntoElement {
        let projection = self.app_model.projection(cx);
        let theme = cx.global::<Theme>();

        div()
            .flex()
            .flex_col()
            .size_full()
            .bg(theme.colors.surface)
            .border_r_1()
            .border_color(theme.colors.border)
            .child(
                div()
                    .h(px(36.0))
                    .px_3()
                    .flex()
                    .items_center()
                    .text_sm()
                    .font_weight(gpui::FontWeight::SEMIBOLD)
                    .text_color(theme.colors.text_primary)
                    .child("Order Book")
            )
            .child(
                div()
                    .flex_1()
                    .overflow_y_scroll()
                    .children(projection.book_levels.iter().map(|level| {
                        div()
                            .h(px(24.0))
                            .px_3()
                            .flex()
                            .justify_between()
                            .items_center()
                            .text_xs()
                            .child(format!("{}", level.price))
                            .child(format!("{} lots", level.quantity))
                    }))
            )
    }
}
```

### Keyboard Actions & Scoped Shortcuts
Bind actions declaratively using GPUI actions:

```rust
use gpui::{actions, KeyBinding, AppContext};

actions!(workspace, [SubmitOrder, CancelOrder, ToggleTheme, OpenSettings]);

pub fn register_keybindings(cx: &mut AppContext) {
    cx.bind_keys([
        KeyBinding::new("enter", SubmitOrder, Some("OrderForm")),
        KeyBinding::new("escape", CancelOrder, Some("OrderForm")),
        KeyBinding::new("cmd-t", ToggleTheme, None),
        KeyBinding::new("cmd-,", OpenSettings, None),
    ]);
}
```

---

## 4. Single State Owner & Background Persistence Pattern

The `SessionController` coordinates state mutations and emits serializable snapshots to the storage coordinator:

```rust
pub struct SessionController {
    state: TradingState,
    revision: u64,
    save_tx: crossbeam_channel::Sender<Snapshot>,
}

impl SessionController {
    pub fn execute_command(&mut self, cmd: Command) -> Result<Outcome, AppError> {
        // 1. Validate and execute in pure core engine
        let outcome = self.state.apply_command(cmd)?;
        
        // 2. Increment monotonic session revision
        self.revision += 1;
        
        // 3. Queue debounced snapshot save (non-blocking)
        let snapshot = self.state.create_snapshot(self.revision);
        let _ = self.save_tx.try_send(snapshot);
        
        Ok(outcome)
    }
}
```

### Storage Save Coordinator (Background Thread)
```rust
pub fn spawn_save_coordinator(
    rx: crossbeam_channel::Receiver<Snapshot>,
    repository: Arc<dyn SessionRepository>,
) {
    std::thread::Builder::new()
        .name("storage-save-coordinator".into())
        .spawn(move || {
            let mut latest_snapshot: Option<Snapshot> = None;
            let debounce_duration = std::time::Duration::from_millis(250);

            loop {
                // Drain channel and keep only the latest revision
                match rx.recv_timeout(debounce_duration) {
                    Ok(snapshot) => {
                        latest_snapshot = Some(snapshot);
                    }
                    Err(crossbeam_channel::RecvTimeoutError::Timeout) => {
                        if let Some(snapshot) = latest_snapshot.take() {
                            if let Err(e) = repository.save_atomic(&snapshot) {
                                log::error!("Failed background snapshot save: {e:?}");
                            }
                        }
                    }
                    Err(crossbeam_channel::RecvTimeoutError::Disconnected) => break,
                }
            }
        })
        .expect("failed to spawn save coordinator thread");
}
```

---

## 5. Safe Persistence & Atomic Swap

Never overwrite a valid user snapshot directly. Use atomic multi-phase writing with recovery fallback:

1. **Serialize to Temporary File**: Write serialized JSON to `<profile>.json.tmp`.
2. **Sync File to Storage Medium**: Ensure OS disk caches are flushed (`file.sync_all()`).
3. **Backup Existing Profile**: Rename current `<profile>.json` to `<profile>.json.recovery` (preserving prior working state).
4. **Atomic Rename**: Rename `<profile>.json.tmp` to `<profile>.json`.
5. **Clean Recovery**: Remove old `.recovery` only after atomic swap completes successfully.
