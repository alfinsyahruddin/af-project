# Desktop Vertical Slice Feature

This reference implementation demonstrates how to build a complete vertical slice through every crate of a native desktop application: from the deterministic domain rule in `<app>-core`, to application command orchestration in `<app>-app`, atomic persistence in `<app>-storage`, and the reactive GPUI view in `<app>-desktop`.

---

## 1. Domain Model & Invariant Rule (`<app>-core/src/rules.rs`)

Pure business domain logic lives in `core`. No I/O, no UI, and zero `.unwrap()`:

```rust
use serde::{Deserialize, Serialize};
use thiserror::Error;

#[derive(Debug, Error, PartialEq, Eq)]
pub enum DomainError {
    #[error("broker fee percentage must be between 0 and 500 basis points (0.00% to 5.00%)")]
    FeeOutOfRange,
    #[error("maximum order lots must be greater than zero")]
    InvalidMaxLots,
}

#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub struct TradingPreferences {
    /// Basis points (e.g. 15 = 0.15%)
    pub buy_fee_bps: u32,
    pub sell_fee_bps: u32,
    pub max_lots_per_order: u32,
}

impl Default for TradingPreferences {
    fn default() -> Self {
        Self {
            buy_fee_bps: 15,
            sell_fee_bps: 25,
            max_lots_per_order: 50_000,
        }
    }
}

impl TradingPreferences {
    pub fn validate(&self) -> Result<(), DomainError> {
        if self.buy_fee_bps > 500 || self.sell_fee_bps > 500 {
            return Err(DomainError::FeeOutOfRange);
        }
        if self.max_lots_per_order == 0 {
            return Err(DomainError::InvalidMaxLots);
        }
        Ok(())
    }
}
```

---

## 2. Command Dispatch & Projection (`<app>-app/src/controller.rs`)

The `SessionController` handles incoming user commands, enforces state revisioning, updates projections behind `Arc`, and schedules background saves:

```rust
use std::sync::Arc;
use crate::types::{AppError, Command, SessionRepository};
use crate::entities::snapshot::Snapshot;
use <app>_core::rules::{TradingPreferences, DomainError};

#[derive(Debug, Clone)]
pub struct WorkspaceProjection {
    pub revision: u64,
    pub preferences: TradingPreferences,
    pub status_message: Option<String>,
}

pub struct SessionController {
    revision: u64,
    preferences: TradingPreferences,
    current_projection: Arc<WorkspaceProjection>,
    save_tx: crossbeam_channel::Sender<Snapshot>,
}

impl SessionController {
    pub fn update_preferences(&mut self, new_pref: TradingPreferences) -> Result<(), AppError> {
        // 1. Validate against pure core domain rules
        new_pref.validate().map_err(|e| AppError::Validation(e.to_string()))?;

        // 2. Mutate state and update monotonic revision
        self.preferences = new_pref.clone();
        self.revision += 1;

        // 3. Update immutable projection slice
        self.current_projection = Arc::new(WorkspaceProjection {
            revision: self.revision,
            preferences: new_pref,
            status_message: Some("Preferences updated successfully".into()),
        });

        // 4. Trigger non-blocking snapshot save to background coordinator
        let snapshot = Snapshot {
            schema_version: 4,
            revision: self.revision,
            preferences: self.preferences.clone(),
        };
        let _ = self.save_tx.try_send(snapshot);

        Ok(())
    }

    pub fn projection(&self) -> Arc<WorkspaceProjection> {
        Arc::clone(&self.current_projection)
    }
}
```

---

## 3. Storage Persistence Port & Adapter (`<app>-storage/src/lib.rs`)

The storage crate implements the port defined by the application layer. It handles safe disk writes with `.tmp` and `.recovery` files:

```rust
use std::fs;
use std::path::{Path, PathBuf};
use <app>_app::types::SessionRepository;
use <app>_app::entities::snapshot::Snapshot;
use thiserror::Error;

#[derive(Debug, Error)]
pub enum StorageError {
    #[error("filesystem error: {0}")]
    Io(#[from] std::io::Error),
    #[error("serialization error: {0}")]
    Serialization(#[from] serde_json::Error),
    #[error("unsupported snapshot schema version: {0}")]
    UnsupportedSchema(u32),
}

pub struct FileSessionRepository {
    base_dir: PathBuf,
}

impl FileSessionRepository {
    pub fn new(base_dir: impl Into<PathBuf>) -> Self {
        Self { base_dir: base_dir.into() }
    }

    pub fn save_atomic(&self, snapshot: &Snapshot) -> Result<(), StorageError> {
        let final_path = self.base_dir.join("session.json");
        let tmp_path = self.base_dir.join("session.json.tmp");
        let backup_path = self.base_dir.join("session.json.recovery");

        // 1. Serialize to temporary file
        let bytes = serde_json::to_vec_pretty(snapshot)?;
        fs::write(&tmp_path, bytes)?;

        // 2. Backup existing file if present
        if final_path.exists() {
            let _ = fs::copy(&final_path, &backup_path);
        }

        // 3. Atomic rename
        fs::rename(&tmp_path, &final_path)?;

        // 4. Cleanup old recovery file
        let _ = fs::remove_file(&backup_path);

        Ok(())
    }
}
```

---

## 4. GPUI Settings Modal View (`<app>-desktop/src/settings_modal.rs`)

The desktop view renders UI using GPUI Kit, binds user input, and dispatches commands to the model:

```rust
use gpui::prelude::*;
use gpui::{div, px, Render, ViewContext, WindowContext, InteractiveElement};
use gpui_kit::button::{Button, ButtonVariants};
use gpui_kit::theme::Theme;
use <app>_core::rules::TradingPreferences;
use crate::app_model::AppModel;

pub struct SettingsModalView {
    app_model: AppModel,
    buy_fee_input: String,
    sell_fee_input: String,
    error_message: Option<String>,
}

impl SettingsModalView {
    pub fn new(app_model: AppModel, cx: &mut ViewContext<Self>) -> Self {
        let pref = app_model.projection(cx).preferences.clone();
        Self {
            app_model,
            buy_fee_input: pref.buy_fee_bps.to_string(),
            sell_fee_input: pref.sell_fee_bps.to_string(),
            error_message: None,
        }
    }

    fn save_preferences(&mut self, cx: &mut ViewContext<Self>) {
        let buy_fee: u32 = match self.buy_fee_input.parse() {
            Ok(v) => v,
            Err(_) => {
                self.error_message = Some("Invalid buy fee".into());
                cx.notify();
                return;
            }
        };

        let sell_fee: u32 = match self.sell_fee_input.parse() {
            Ok(v) => v,
            Err(_) => {
                self.error_message = Some("Invalid sell fee".into());
                cx.notify();
                return;
            }
        };

        let new_pref = TradingPreferences {
            buy_fee_bps: buy_fee,
            sell_fee_bps: sell_fee,
            max_lots_per_order: 50_000,
        };

        match self.app_model.update_preferences(new_pref, cx) {
            Ok(_) => {
                self.error_message = None;
                // Close modal or notify parent
            }
            Err(err) => {
                self.error_message = Some(err.to_string());
                cx.notify();
            }
        }
    }
}

impl Render for SettingsModalView {
    fn render(&mut self, cx: &mut ViewContext<Self>) -> impl IntoElement {
        let theme = cx.global::<Theme>();

        div()
            .w(px(400.0))
            .p_4()
            .bg(theme.colors.surface)
            .rounded_md()
            .flex()
            .flex_col()
            .gap_3()
            .child(
                div()
                    .text_base()
                    .font_weight(gpui::FontWeight::BOLD)
                    .child("Settings")
            )
            .when_some(self.error_message.as_ref(), |this, err| {
                this.child(div().text_xs().text_color(theme.colors.danger).child(err.clone()))
            })
            .child(
                Button::new("save-button")
                    .label("Save Changes")
                    .variant(ButtonVariants::Primary)
                    .on_click(cx.listener(|this, _, cx| this.save_preferences(cx)))
            )
    }
}
```

---

## 5. Verification: Tests Across the Slice

Every crate has dedicated test boundaries verifying its behavior in isolation:

### Unit Test in `<app>-core` (`tests/rules_test.rs`)
```rust
use <app>_core::rules::{TradingPreferences, DomainError};

#[test]
fn test_validates_excessive_fee() {
    let pref = TradingPreferences {
        buy_fee_bps: 600, // exceeds 500 bps limit
        sell_fee_bps: 25,
        max_lots_per_order: 100,
    };
    assert_eq!(pref.validate(), Err(DomainError::FeeOutOfRange));
}
```

### Workflow Test in `<app>-app` (`tests/workflows.rs`)
```rust
#[test]
fn test_update_preferences_emits_projection() {
    let (mut controller, rx) = create_test_controller();
    let pref = TradingPreferences {
        buy_fee_bps: 20,
        sell_fee_bps: 30,
        max_lots_per_order: 10_000,
    };

    assert!(controller.update_preferences(pref.clone()).is_ok());
    assert_eq!(controller.projection().preferences, pref);
    assert_eq!(controller.projection().revision, 1);
}
```

### Persistence Test in `<app>-storage` (`tests/persistence.rs`)
```rust
#[test]
fn test_atomic_save_and_recovery() {
    let temp_dir = tempfile::tempdir().unwrap();
    let repo = FileSessionRepository::new(temp_dir.path());
    let snapshot = Snapshot::sample_v4();

    assert!(repo.save_atomic(&snapshot).is_ok());
    assert!(temp_dir.path().join("session.json").exists());
    assert!(!temp_dir.path().join("session.json.tmp").exists());
}
```
