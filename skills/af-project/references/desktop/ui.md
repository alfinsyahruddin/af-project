# Desktop UI & Component Guide

This guide establishes the user interface architecture, window layout principles, GPUI Kit component standards, theme design tokens, keyboard navigation, and feedback patterns for native desktop applications.

---

## 1. Window Management & Workspace Layout

Desktop applications run in a unified, responsive native window with structured spatial ergonomics:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│ 60px App Header & Platform Window Chrome                                     │
│ [Traffic Lights / Menu]  [Title / Stock / Status]  [Global Modals & Theme]  │
├───────────────────┬───────────────────────────────┬─────────────────────────┤
│ Left Column (32%) │ Center Column (36%)           │ Right Column (32%)      │
│                   │                               │                         │
│ • Primary Lists   │ • Core Visualization / Depth  │ • Canvas Charts / Graph │
│ • Account Summary │ • Command & Form Entry        │ • Execution Feed        │
│                   │                               │                         │
│ (Scrollable area) │ (Scrollable depth + fit form) │ (Fixed chart + scroll)  │
└───────────────────┴───────────────────────────────┴─────────────────────────┘
```

### Window Dimension Rules
- **Default Resolution**: 1440 × 900 logical pixels.
- **Minimum Bounds**: Clamp window minimums (e.g., 1100 × 720). If the primary display is smaller, clamp to display workarea bounds.
- **Centering & Dimension Persistence**: On launch, reopen centered on the primary display using saved dimensions. Never save raw display coordinates across displays or multi-monitor setups.
- **Proportional Resizing**: Column dividers allow user drag-to-resize; persist proportions (e.g., 32% / 36% / 32%).
- **Scroll Boundaries**: Forms fit content height without internal scrolling; data tables, order books, and lists occupy remaining column height with explicit vertical scrollbars.

### Platform Chrome Integration
- **macOS**: Hide default OS titlebar (`titlebar: None`). Share the custom 60px header with GPUI Kit's `TitleBar` component to embed native traffic lights (close, minimize, zoom) while supporting window dragging and double-click zoom.
- **Windows & Linux**: Retain native window borders and decorations. Provide a top **Menu** button or standard menu bar for system actions, editing, and quit commands.

---

## 2. Design System: Themes, Colors & Typography

Desktop apps must support high-contrast, theme-aware visuals with dark and light variants.

### Palette Tokens

| Semantic Token | Dark Theme | Light Theme | Usage |
| :--- | :--- | :--- | :--- |
| `background` | `#0B0F19` (Deep Navy) | `#F8FAFC` (Cool Off-White) | Root window canvas background |
| `surface` | `#121826` (Navy Card) | `#FFFFFF` (Pure White) | Panels, cards, modal dialogs |
| `surface_hover` | `#1E293B` | `#F1F5F9` | Hovered list rows and button states |
| `border` | `#1F2937` | `#E2E8F0` | Pane dividers, card borders |
| `text_primary` | `#F9FAFB` | `#0F172A` | Primary headings, numbers, critical data |
| `text_muted` | `#94A3B8` | `#64748B` | Secondary labels, timestamps, units |
| `accent` | `#FFC700` (Amber Gold) | `#D97706` | Player identification, selected highlights |
| `success / bull` | `#32D02F` (Vibrant Green) | `#16A34A` | Buy orders, gains, connected state |
| `danger / bear` | `#FF4F4F` (Vibrant Red) | `#DC2626` | Sell orders, losses, error badges |

### Typography Guidelines
1. **Numeric & Tabular Data**: Always use tabular monospaced fonts (e.g., *IBM Plex Mono* or system monospaced) for prices, lot sizes, monetary balances, and tables to prevent number jitter during live updates.
2. **Interface Labels & Headings**: Use clean proportional fonts (e.g., *Rajdhani* or *Inter*) for navigation items, button labels, and section headings.
3. **Number Formatting**: Format all large financial or unit counts with localized digit grouping (e.g., `10.000` or `100,000,000`).

---

## 3. GPUI Kit Components & Interactive Controls

### Segmented Controls (with Spring Motion)
For mutually exclusive modes (e.g., Buy / Sell / Withdraw or 1m / 5m / 30m):
- Render a shared background pill that animates smoothly between options using spring physics.
- Respect OS accessibility settings: when **Reduced Motion** is active, skip spring transitions and jump immediately.

### Action Buttons
- Style submit and primary actions with explicit semantic intent:
  - Green for **BUY** / **Submit**
  - Red for **SELL** / **Delete**
  - Amber / Accent for **Withdraw** / **Apply** / **Primary Config**
- Disabled state must be visually distinct with reduced opacity and cursor styling; never trigger actions when invalid.

### Custom Canvas Elements (Charts & Visual Depth)
- Render custom graphics (price charts, depth histograms, candlestick plots) using GPUI's canvas primitives (`cx.paint()`).
- Fit data automatically with sensible vertical padding.
- Prepend reference lines (e.g., previous close baseline) and split area fills at baseline crossings (e.g. green area above, red area below).
- Empty state: When no data is available, display a clean centered placeholder (e.g., `Waiting for trades…`) rather than an empty coordinate grid.

---

## 4. Retained State & View Reactivity

When building reactive GPUI views, follow strict retention principles:

1. **Single State Owner (`Workspace`)**:
   - The workspace model owns the session controller. Views read projections and dispatch actions.
2. **Retain In-Progress Input State**:
   - Background data updates or tick events must NEVER overwrite active text entry fields (e.g. typing a price or custom trader name).
   - Form inputs maintain local transient state until committed.
3. **Avoid Rebuilding Background Workers**:
   - View redraws triggered by `cx.notify()` must only re-evaluate render trees; they must never spawn background timers or reconnect channels.
4. **Read-Only Projections**:
   - Views consume `Arc<Projection>` slices. Never perform deep sorting or expensive aggregations in `render()`; precalculate them in the controller.

---

## 5. Keyboard Navigation & Shortcuts

Every primary desktop workflow must be executable without touching the mouse:

| Shortcut (macOS / Win & Linux) | Scope | Intended Action |
| :--- | :--- | :--- |
| `Cmd+Q` / `Ctrl+Q` | Global | Flush pending snapshot saves to disk and quit application cleanly |
| `Cmd+N` / `Ctrl+N` | Workspace | Focus primary input form (e.g., order lots or search field) |
| `Cmd+Enter` / `Ctrl+Enter` | Form / Modal | Submit primary form or confirm active modal |
| `Enter` | Active Input | Submit current input field |
| `Escape` | Modal / Dialog | Dismiss active modal, cancel inline edit, or return to workspace |
| `Tab` / `Shift+Tab` | Controls | Cycle focus forward and backward across focusable interactive elements |
| `Cmd+B` / `Ctrl+B` | Workspace | Switch mode / side to Buy / Mode 1 |
| `Cmd+S` / `Ctrl+S` | Workspace | Switch mode / side to Sell / Mode 2 |
| `Cmd+W` / `Ctrl+W` | Workspace | Switch mode to Withdraw / Mode 3 |

### Implementing Declarative Keybindings
```rust
use gpui::{actions, KeyBinding, AppContext};

actions!(workspace, [
    SubmitForm,
    CancelAction,
    FocusInput,
    ToggleTheme,
]);

pub fn bind_workspace_keys(cx: &mut AppContext) {
    cx.bind_keys([
        KeyBinding::new("enter", SubmitForm, Some("OrderForm")),
        KeyBinding::new("cmd-enter", SubmitForm, None),
        KeyBinding::new("escape", CancelAction, None),
        KeyBinding::new("cmd-n", FocusInput, None),
        KeyBinding::new("cmd-t", ToggleTheme, None),
    ]);
}
```

---

## 6. Modal Dialogs & Overlays

Modals are used for configuration, inspections, and game lifecycle states:

- **Backdrop**: 64% opaque black (`rgba(0, 0, 0, 0.64)`) backdrop dimming the background workspace.
- **Dismissal Behavior**:
  - Clicking outside on the backdrop closes the modal (unless an unsaved destructive confirmation is pending).
  - Pressing `Escape` closes the modal immediately.
- **Animation**: Smooth entrance using fade-and-slide motion; close immediately to keep the UI snappy.
- **Live Background Updates**: When inspection dialogs are open (e.g., portfolio details, leaderboard), the dialog should remain mounted and react live to background state changes without closing.

---

## 7. User Feedback & Notification Suppression

Desktop applications must balance informative feedback with non-intrusive presentation:

### Toast Notification Levels

| Level | Visual Style | Criteria to Trigger |
| :--- | :--- | :--- |
| **Success** | Green accent, check icon | Explicit user command completed (e.g. manual export, save) |
| **Update** | Neutral grey, info icon | Informational milestone |
| **Attention** | Amber accent, warning icon | Soft validation failure or non-critical state |
| **Error** | Red accent, alert icon | Hard failure, command rejection, network or I/O failure |

### Notification Suppression Rules
> [!IMPORTANT]
> - **Quiet Background Activity**: Automated background simulated agent activity or periodic tick cycles must NEVER spawn toast cards.
> - **Suppress Normal Partial Fills**: Executions that occur normally in the market should update tables, depth, and portfolios silently. Do NOT spam toasts for every fill event.
> - **Persistent Save Warnings**: If a background disk save fails, do not display a fleeting 3-second toast. Display a persistent bottom status banner with a **Retry save** button.
> - **Safe Quit Confirmation**: If unwritten changes exist when closing the app, present an explicit confirmation dialog: **Retry Save**, **Quit Without Saving**, or **Cancel**.
