+++
title = "Building a TUI app in Rust with ratatui"
date = 2025-03-12
description = "A hands-on walkthrough of building a keyboard-driven todo list for the terminal using ratatui, covering the event loop, layout system, widgets, and what production TUI apps look like under the hood."

[taxonomies]
tags = ["rust", "tui", "ratatui", "crossterm"]
+++

Most developers interact with the terminal every day but never think about building for it. The terminal is a rendering target just like a browser or a native window - it has pixels (cells), a coordinate system, input events, and a redraw loop. Ratatui gives you a framework for treating it that way.

Ratatui is a Rust crate (forked from the abandoned `tui-rs` in 2023) that implements immediate-mode rendering for terminal UIs. It has 19k+ GitHub stars, 22M+ downloads on crates.io, and powers production tools like [gitui](https://github.com/gitui-org/gitui), [bottom](https://github.com/ClementTsang/bottom), and [bandwhich](https://github.com/imsnif/bandwhich). The latest release is 0.30.0, which reorganized the crate into a modular workspace (`ratatui-core`, `ratatui-widgets`, backend crates) for faster compile times.

We're going to build a keyboard-driven todo list app from scratch. Not a toy example - something with selection state, visual feedback, multiple layout regions, and proper terminal cleanup on exit.

<!-- more -->

## How terminal UIs actually work

Before writing code, it helps to understand what ratatui is doing at the OS level.

When you call `ratatui::init()`, three things happen:

1. **Raw mode** is enabled via `tcsetattr()` (or the Windows console API). This disables line buffering and echo - your program gets every keystroke immediately instead of waiting for Enter.
2. **The alternate screen buffer** is activated via the `\x1b[?1049h` ANSI escape sequence. This is a second framebuffer that terminals maintain. When your app exits, the terminal switches back to the primary buffer and your shell history is untouched.
3. A **panic hook** is installed that calls `ratatui::restore()` before unwinding. Without this, a panic leaves your terminal in raw mode with the alternate screen still active - you'd have to `reset` to recover.

Ratatui then gives you a `Terminal` struct that wraps a backend (crossterm by default). Every frame, you call `terminal.draw()` with a closure that receives a `Frame`. Inside that closure, you describe what the screen should look like *right now*. Ratatui diffs the new buffer against the previous one and only emits escape sequences for cells that changed. This is why TUI apps feel snappy even over SSH - the diff keeps bandwidth low.

This is immediate-mode rendering. There's no retained widget tree, no virtual DOM. Every frame recomputes the entire UI from your application state. If your state says item 3 is selected, you render item 3 as highlighted. If the user presses Down, you update the state, and the next frame reflects it.

## Project setup

```toml
# Cargo.toml
[package]
name = "tui-todo"
version = "0.1.0"
edition = "2021"

[dependencies]
ratatui = "0.30"
crossterm = "0.28"
```

Ratatui enables the `crossterm` feature by default, but you need `crossterm` as a direct dependency too because you'll use its event types (`KeyCode`, `Event`) directly.

## The skeleton: init, loop, restore

Here's the minimal structure every ratatui app follows:

```rust
use std::io;
use crossterm::event::{self, Event, KeyCode, KeyEventKind};
use ratatui::{DefaultTerminal, Frame};

fn main() -> io::Result<()> {
    let mut terminal = ratatui::init();
    let result = run(&mut terminal);
    ratatui::restore();
    result
}

fn run(terminal: &mut DefaultTerminal) -> io::Result<()> {
    loop {
        terminal.draw(|frame| ui(frame))?;

        if let Event::Key(key) = event::read()? {
            if key.kind != KeyEventKind::Press {
                continue;
            }
            if key.code == KeyCode::Char('q') {
                return Ok(());
            }
        }
    }
}

fn ui(frame: &mut Frame) {
    frame.render_widget("Hello, terminal!", frame.area());
}
```

`ratatui::init()` returns a `DefaultTerminal` - a type alias for `Terminal<CrosstermBackend<Stdout>>`. The `run` function contains the event loop. `ratatui::restore()` always runs, even if `run` returns an error, because we call it unconditionally after.

A note on `event::read()`: this is a blocking call. The thread parks until the user presses a key or the terminal emits a resize event. That's fine for our todo app. If you need animations or background updates (think a progress bar or live data), you'd use `event::poll(Duration)` to check for events with a timeout and re-render periodically even without input.

One gotcha on Windows: crossterm sends both `KeyEventKind::Press` and `KeyEventKind::Release` events. On Linux and macOS, you only get `Press`. That `if key.kind != KeyEventKind::Press` guard prevents double-handling on Windows.

## Application state

Our todo app needs state: a list of items, which one is selected, and a way to toggle completion.

```rust
struct App {
    items: Vec<TodoItem>,
    state: ListState,
    should_quit: bool,
}

struct TodoItem {
    description: String,
    done: bool,
}

impl App {
    fn new() -> Self {
        let items = vec![
            TodoItem { description: "Write blog post about ratatui".into(), done: false },
            TodoItem { description: "Review PR #42".into(), done: true },
            TodoItem { description: "Fix CI pipeline".into(), done: false },
            TodoItem { description: "Update dependencies".into(), done: false },
            TodoItem { description: "Write integration tests".into(), done: false },
        ];
        let mut state = ListState::default();
        state.select_first();
        App { items, state, should_quit: false }
    }

    fn select_next(&mut self) {
        self.state.select_next();
    }

    fn select_previous(&mut self) {
        self.state.select_previous();
    }

    fn toggle_current(&mut self) {
        if let Some(i) = self.state.selected() {
            self.items[i].done = !self.items[i].done;
        }
    }

    fn delete_current(&mut self) {
        if let Some(i) = self.state.selected() {
            self.items.remove(i);
            // If we deleted the last item, move selection up
            if i >= self.items.len() && !self.items.is_empty() {
                self.state.select(Some(self.items.len() - 1));
            }
        }
    }
}
```

`ListState` is provided by ratatui. It tracks which item is selected and handles scroll offset automatically - when the selected item would be off-screen, it adjusts the viewport. The `select_next()` and `select_previous()` methods handle wrapping and bounds checking for you.

## The layout system

Ratatui's layout system splits a `Rect` (a rectangular area defined by x, y, width, height) into smaller `Rect`s using constraints. Think of it like CSS flexbox but for terminal cells.

```rust
use ratatui::layout::{Constraint, Direction, Layout};

let chunks = Layout::default()
    .direction(Direction::Vertical)
    .constraints([
        Constraint::Length(3),    // Title bar: exactly 3 rows
        Constraint::Min(5),       // Main content: at least 5, takes remaining space
        Constraint::Length(3),    // Status bar: exactly 3 rows
    ])
    .split(frame.area());
```

The six constraint types and what they actually do:

| Constraint | Behavior |
|---|---|
| `Length(n)` | Exactly `n` cells. Non-responsive - if the terminal is too small, content gets clipped. |
| `Percentage(n)` | `n`% of the parent area. |
| `Ratio(a, b)` | `a/b` of the parent area. Useful for equal splits: `Ratio(1, 3)`. |
| `Min(n)` | At least `n` cells. Grabs remaining space after fixed constraints are satisfied. |
| `Max(n)` | At most `n` cells. |
| `Fill(n)` | Takes excess space proportionally. `Fill(2)` gets twice the excess of `Fill(1)`. |

Layouts compose by nesting. Split vertically first, then split one of the resulting chunks horizontally:

```rust
let outer = Layout::default()
    .direction(Direction::Vertical)
    .constraints([
        Constraint::Length(3),
        Constraint::Fill(1),
        Constraint::Length(3),
    ])
    .split(frame.area());

// Split the middle section horizontally
let inner = Layout::default()
    .direction(Direction::Horizontal)
    .constraints([
        Constraint::Percentage(60),
        Constraint::Percentage(40),
    ])
    .split(outer[1]);

// outer[0] = top bar
// inner[0] = left panel (60% of middle)
// inner[1] = right panel (40% of middle)
// outer[2] = bottom bar
```

You can also use the shorthand constructors: `Layout::horizontal([...])` and `Layout::vertical([...])` skip the `direction()` call.

## Widgets: the building blocks

Ratatui ships with a set of built-in widgets. Each implements the `Widget` trait:

```rust
pub trait Widget {
    fn render(self, area: Rect, buf: &mut Buffer);
}
```

The `self` is consumed - widgets are ephemeral. You create them in the draw closure, they render into the buffer, and they're dropped. No persistent widget objects between frames.

For widgets that need to track state across frames (like a list's scroll position), there's `StatefulWidget`:

```rust
pub trait StatefulWidget {
    type State;
    fn render(self, area: Rect, buf: &mut Buffer, state: &mut Self::State);
}
```

You own the `State` in your app struct and pass a mutable reference each frame.

Let's look at the widgets we'll use.

### Block

`Block` is a container that draws borders and titles around other widgets:

```rust
use ratatui::widgets::{Block, Borders};

let block = Block::bordered()
    .title(" My App ")
    .border_style(Style::default().fg(Color::Cyan));
```

`Block::bordered()` is shorthand for `Block::default().borders(Borders::ALL)`. You can also use `Borders::TOP | Borders::BOTTOM` or any combination.

### Paragraph

`Paragraph` renders styled text with optional wrapping:

```rust
use ratatui::widgets::{Paragraph, Wrap};
use ratatui::text::{Line, Span};
use ratatui::style::{Color, Modifier, Style};

let text = vec![
    Line::from(vec![
        Span::styled("Status: ", Style::default().fg(Color::Gray)),
        Span::styled("5 items", Style::default().fg(Color::Green).add_modifier(Modifier::BOLD)),
    ]),
];

let paragraph = Paragraph::new(text)
    .block(Block::bordered().title(" Info "))
    .wrap(Wrap { trim: true });
```

`Span` is the atomic text unit - a string with a style. `Line` is a row of spans. `Text` is a list of lines. `Paragraph` renders a `Text`.

### List

`List` is a `StatefulWidget` - it works with `ListState` to handle selection and scrolling:

```rust
use ratatui::widgets::{List, ListItem, ListState};

let items: Vec<ListItem> = app.items.iter().map(|item| {
    let prefix = if item.done { "[x]" } else { "[ ]" };
    ListItem::new(format!("{} {}", prefix, item.description))
}).collect();

let list = List::new(items)
    .block(Block::bordered().title(" Todo "))
    .highlight_style(Style::default().bg(Color::DarkGray).add_modifier(Modifier::BOLD))
    .highlight_symbol(">> ");

frame.render_stateful_widget(list, area, &mut app.state);
```

`highlight_style` sets the style of the currently selected row. `highlight_symbol` prepends a marker to it. The `ListState` tracks selection and scroll offset - when you call `select_next()`, ratatui automatically scrolls the viewport to keep the selected item visible.

### Table

`Table` renders columnar data with optional row selection:

```rust
use ratatui::widgets::{Cell, Row, Table};

let header = Row::new(vec![
    Cell::from("Status"),
    Cell::from("Description"),
]).style(Style::default().fg(Color::Yellow)).bottom_margin(1);

let rows: Vec<Row> = app.items.iter().map(|item| {
    Row::new(vec![
        Cell::from(if item.done { "done" } else { "todo" }),
        Cell::from(item.description.as_str()),
    ])
}).collect();

let table = Table::new(rows, [Constraint::Length(6), Constraint::Fill(1)])
    .header(header)
    .block(Block::bordered().title(" Tasks "));
```

The second argument to `Table::new` specifies column widths as constraints. `Constraint::Fill(1)` makes the description column take all remaining space.

### Chart

`Chart` renders line or scatter plots. It's how [bottom](https://github.com/ClementTsang/bottom) draws CPU and memory graphs:

```rust
use ratatui::widgets::{Axis, Chart, Dataset, GraphType};
use ratatui::symbols;

let data = vec![(0.0, 1.0), (1.0, 3.0), (2.0, 2.0), (3.0, 5.0), (4.0, 4.0)];

let dataset = Dataset::default()
    .name("Progress")
    .marker(symbols::Marker::Braille)
    .graph_type(GraphType::Line)
    .style(Style::default().fg(Color::Cyan))
    .data(&data);

let chart = Chart::new(vec![dataset])
    .block(Block::bordered().title(" Chart "))
    .x_axis(
        Axis::default()
            .title("Time")
            .bounds([0.0, 4.0])
            .labels(vec!["0", "1", "2", "3", "4"]),
    )
    .y_axis(
        Axis::default()
            .title("Value")
            .bounds([0.0, 6.0])
            .labels(vec!["0", "3", "6"]),
    );
```

The `Braille` marker uses Unicode braille characters to pack multiple data points into a single terminal cell - each cell becomes a 2x4 dot grid, giving you 8x the effective resolution. This is why TUI charts can look surprisingly smooth.

## Putting it together: the full todo app

Let's wire everything up. Our app has three regions: a title bar, the todo list, and a help bar showing keybindings.

```rust
use std::io;

use crossterm::event::{self, Event, KeyCode, KeyEventKind};
use ratatui::{
    layout::{Constraint, Layout},
    style::{Color, Modifier, Style, Stylize},
    text::{Line, Span},
    widgets::{Block, List, ListItem, ListState, Paragraph},
    DefaultTerminal, Frame,
};

struct TodoItem {
    description: String,
    done: bool,
}

struct App {
    items: Vec<TodoItem>,
    state: ListState,
    should_quit: bool,
}

impl App {
    fn new() -> Self {
        let items = vec![
            TodoItem { description: "Write blog post about ratatui".into(), done: false },
            TodoItem { description: "Review PR #42".into(), done: true },
            TodoItem { description: "Fix CI pipeline".into(), done: false },
            TodoItem { description: "Update dependencies".into(), done: false },
            TodoItem { description: "Write integration tests".into(), done: false },
        ];
        let mut state = ListState::default();
        state.select_first();
        App { items, state, should_quit: false }
    }

    fn select_next(&mut self) {
        self.state.select_next();
    }

    fn select_previous(&mut self) {
        self.state.select_previous();
    }

    fn toggle_current(&mut self) {
        if let Some(i) = self.state.selected() {
            self.items[i].done = !self.items[i].done;
        }
    }

    fn delete_current(&mut self) {
        if let Some(i) = self.state.selected() {
            if !self.items.is_empty() {
                self.items.remove(i);
                if i >= self.items.len() && !self.items.is_empty() {
                    self.state.select(Some(self.items.len() - 1));
                }
                if self.items.is_empty() {
                    self.state.select(None);
                }
            }
        }
    }

    fn done_count(&self) -> usize {
        self.items.iter().filter(|i| i.done).count()
    }
}

fn main() -> io::Result<()> {
    let mut terminal = ratatui::init();
    let mut app = App::new();
    let result = run(&mut terminal, &mut app);
    ratatui::restore();
    result
}

fn run(terminal: &mut DefaultTerminal, app: &mut App) -> io::Result<()> {
    while !app.should_quit {
        terminal.draw(|frame| ui(frame, app))?;
        handle_events(app)?;
    }
    Ok(())
}

fn handle_events(app: &mut App) -> io::Result<()> {
    if let Event::Key(key) = event::read()? {
        if key.kind != KeyEventKind::Press {
            return Ok(());
        }
        match key.code {
            KeyCode::Char('q') | KeyCode::Esc => app.should_quit = true,
            KeyCode::Down | KeyCode::Char('j') => app.select_next(),
            KeyCode::Up | KeyCode::Char('k') => app.select_previous(),
            KeyCode::Char(' ') | KeyCode::Enter => app.toggle_current(),
            KeyCode::Char('d') | KeyCode::Delete => app.delete_current(),
            _ => {}
        }
    }
    Ok(())
}

fn ui(frame: &mut Frame, app: &mut App) {
    let chunks = Layout::vertical([
        Constraint::Length(3),
        Constraint::Fill(1),
        Constraint::Length(3),
    ])
    .split(frame.area());

    // Title bar
    let title = Paragraph::new(" Todo List")
        .style(Style::default().fg(Color::Cyan).add_modifier(Modifier::BOLD))
        .block(Block::bordered());
    frame.render_widget(title, chunks[0]);

    // Todo list
    let items: Vec<ListItem> = app
        .items
        .iter()
        .map(|item| {
            let style = if item.done {
                Style::default().fg(Color::DarkGray)
            } else {
                Style::default().fg(Color::White)
            };
            let prefix = if item.done { "[x]" } else { "[ ]" };
            ListItem::new(Line::from(vec![
                Span::styled(format!("{} ", prefix), style),
                Span::styled(&item.description, style),
            ]))
        })
        .collect();

    let done = app.done_count();
    let total = app.items.len();
    let list_title = format!(" Tasks ({}/{} done) ", done, total);

    let list = List::new(items)
        .block(Block::bordered().title(list_title))
        .highlight_style(
            Style::default()
                .bg(Color::DarkGray)
                .add_modifier(Modifier::BOLD),
        )
        .highlight_symbol(">> ");

    frame.render_stateful_widget(list, chunks[1], &mut app.state);

    // Help bar
    let help = Paragraph::new(Line::from(vec![
        Span::styled(" j/k", Style::default().fg(Color::Yellow)),
        Span::raw(" move  "),
        Span::styled("space", Style::default().fg(Color::Yellow)),
        Span::raw(" toggle  "),
        Span::styled("d", Style::default().fg(Color::Yellow)),
        Span::raw(" delete  "),
        Span::styled("q", Style::default().fg(Color::Yellow)),
        Span::raw(" quit"),
    ]))
    .block(Block::bordered());
    frame.render_widget(help, chunks[2]);
}
```

Run it with `cargo run` and you get a working todo list. Arrow keys or `j`/`k` to navigate, space to toggle, `d` to delete, `q` to quit. The terminal restores cleanly on exit.

## Under the hood: what `terminal.draw()` does

The `draw` method is where ratatui earns its keep. Here's what happens in a single frame:

1. A fresh `Buffer` is allocated matching the terminal dimensions. Each cell in the buffer stores a character, foreground color, background color, and modifiers.
2. Your closure runs. Every `render_widget` and `render_stateful_widget` call writes into this buffer.
3. Ratatui diffs the new buffer against the previous frame's buffer, cell by cell.
4. Only changed cells emit escape sequences to the backend. A cell that was `"a"` with white foreground and is still `"a"` with white foreground produces zero output.
5. The new buffer becomes the "previous" buffer for the next frame.

The buffer is a flat `Vec<Cell>` indexed as `y * width + x`. Each `Cell` is roughly 24 bytes (a `CompactString` for the character, plus color and modifier bitfields). For a standard 80x24 terminal, that's about 46 KB per buffer. Two buffers for diffing means ~92 KB total. Even a 300x100 terminal only uses ~1.4 MB. The cost is trivial.

You can see this in [ratatui's source](https://github.com/ratatui/ratatui/blob/main/ratatui-core/src/buffer/buffer.rs) - the `diff` method iterates both buffers and collects `(x, y, &Cell)` tuples for every cell that changed.

## Crossterm vs Termion vs Termwiz

Ratatui supports three terminal backends. The choice matters less than you'd think - the API you write against is ratatui's `Terminal` and `Frame`, not the backend directly. But there are real differences:

**Crossterm** (default, recommended):
- Cross-platform: Linux, macOS, Windows
- Pure Rust, no system library dependencies
- The most popular choice in the ecosystem - most examples and tutorials assume crossterm
- Async event support via `EventStream` (behind a feature flag)

**Termion**:
- Unix only (Linux, macOS). No Windows support.
- Lighter weight than crossterm - fewer dependencies, smaller binary
- Uses `stdin`-based event reading rather than system-specific APIs
- Good choice if you're targeting only Unix and want a smaller dependency tree

**Termwiz**:
- Developed by the author of [WezTerm](https://github.com/wez/wezterm)
- Best integration with WezTerm's advanced features
- Niche - only use it if your app specifically targets WezTerm users

Switching backends is a Cargo feature flag change:

```toml
# Use termion instead of crossterm
[dependencies]
ratatui = { version = "0.30", default-features = false, features = ["termion"] }
termion = "4"
```

The rest of your code stays the same. Your `ui()` function, your layout code, your widgets - none of it changes. Only the terminal initialization differs slightly.

In practice, just use crossterm. The ecosystem has standardized on it, and the Windows support means your tool works everywhere without conditional compilation.

## Styling: more than you'd expect

Terminal styling in ratatui goes deeper than bold and colors. The `Style` struct supports:

```rust
use ratatui::style::{Color, Modifier, Style, Stylize};

// Verbose way
let style = Style::default()
    .fg(Color::Rgb(255, 165, 0))    // True color (24-bit)
    .bg(Color::Black)
    .add_modifier(Modifier::BOLD | Modifier::ITALIC);

// Shorthand with Stylize trait
let span = "important".red().bold().on_black();
```

Color support depends on the terminal emulator:

- **16 colors**: `Color::Red`, `Color::Blue`, etc. Works everywhere.
- **256 colors**: `Color::Indexed(n)`. Works on most modern terminals.
- **True color (24-bit)**: `Color::Rgb(r, g, b)`. Requires terminal support (most terminals since ~2018).

The `Stylize` trait provides a fluent API that chains directly on strings, spans, and other types. It's much more readable than constructing `Style` objects manually.

Modifiers include `BOLD`, `DIM`, `ITALIC`, `UNDERLINED`, `REVERSED`, `CROSSED_OUT`, and `HIDDEN`. You can combine them with bitwise OR. `REVERSED` is particularly useful for selected items - it swaps foreground and background, so it works regardless of the user's color scheme.

## Patterns for larger apps

Our todo app is ~150 lines. Real TUI apps are 10-50x that. Here are patterns they use.

### Separate event handling from rendering

Our `handle_events` function mutates `App` directly. In larger apps, you'd use a message-passing pattern (sometimes called The Elm Architecture or TEA):

```rust
enum Message {
    MoveUp,
    MoveDown,
    ToggleItem,
    DeleteItem,
    Quit,
}

fn handle_events() -> io::Result<Option<Message>> {
    if let Event::Key(key) = event::read()? {
        if key.kind != KeyEventKind::Press {
            return Ok(None);
        }
        let msg = match key.code {
            KeyCode::Up | KeyCode::Char('k') => Some(Message::MoveUp),
            KeyCode::Down | KeyCode::Char('j') => Some(Message::MoveDown),
            KeyCode::Char(' ') => Some(Message::ToggleItem),
            KeyCode::Char('d') => Some(Message::DeleteItem),
            KeyCode::Char('q') => Some(Message::Quit),
            _ => None,
        };
        return Ok(msg);
    }
    Ok(None)
}

fn update(app: &mut App, msg: Message) {
    match msg {
        Message::MoveUp => app.select_previous(),
        Message::MoveDown => app.select_next(),
        Message::ToggleItem => app.toggle_current(),
        Message::DeleteItem => app.delete_current(),
        Message::Quit => app.should_quit = true,
    }
}
```

This separates "what happened" from "what to do about it." Testing becomes straightforward - you can unit test `update` by feeding it messages directly, no terminal needed.

### Component trait for nested UIs

When you have multiple panels with independent behavior (like a file tree + editor + terminal split), define a component trait:

```rust
trait Component {
    fn handle_key(&mut self, key: KeyCode);
    fn render(&self, frame: &mut Frame, area: Rect);
}
```

Each component manages its own state and rendering. The parent dispatches events to the focused component and calls `render` on all of them with their assigned areas.

### Async event loops

If your app fetches data or runs background tasks, you need a non-blocking event loop. The pattern uses `crossterm::event::EventStream` (enable the `event-stream` feature on crossterm) with tokio:

```rust
use crossterm::event::EventStream;
use futures::StreamExt;
use tokio::select;

async fn run_async(terminal: &mut DefaultTerminal, app: &mut App) -> io::Result<()> {
    let mut events = EventStream::new();

    loop {
        terminal.draw(|frame| ui(frame, app))?;

        select! {
            event = events.next() => {
                if let Some(Ok(Event::Key(key))) = event {
                    handle_key(app, key);
                }
            }
            // Add other futures here: network responses, timers, etc.
        }

        if app.should_quit {
            break;
        }
    }
    Ok(())
}
```

`select!` waits on multiple futures simultaneously. You can add a `tokio::time::interval` for periodic redraws, a channel receiver for background task results, or a network stream for live data.

## Custom widgets

You're not limited to built-in widgets. Implementing `Widget` is simple:

```rust
use ratatui::{
    buffer::Buffer,
    layout::Rect,
    style::{Color, Style},
    widgets::Widget,
};

struct ProgressBar {
    progress: f64, // 0.0 to 1.0
    label: String,
}

impl Widget for ProgressBar {
    fn render(self, area: Rect, buf: &mut Buffer) {
        if area.height == 0 || area.width == 0 {
            return;
        }

        let filled = (area.width as f64 * self.progress) as u16;

        for x in 0..area.width {
            let style = if x < filled {
                Style::default().bg(Color::Green).fg(Color::Black)
            } else {
                Style::default().bg(Color::DarkGray).fg(Color::White)
            };

            let ch = if x < self.label.len() as u16 {
                self.label.chars().nth(x as usize).unwrap_or(' ')
            } else {
                ' '
            };

            buf.set_string(area.x + x, area.y, ch.to_string(), style);
        }
    }
}
```

The `Buffer` is your canvas. You write characters with styles at specific coordinates. The `Rect` tells you your allocated area - never write outside it.

The [awesome-ratatui](https://github.com/ratatui/awesome-ratatui) repository lists third-party widgets: tree views, image renderers, markdown viewers, code editors, and more. Before building your own, check if someone already has.

## Production TUI apps: what they look like inside

The Rust TUI ecosystem has some impressive production tools. Looking at their architecture reveals patterns worth stealing.

**[gitui](https://github.com/gitui-org/gitui)** (10k+ stars) - a terminal git client. It uses ratatui with a component-based architecture. Each panel (status, diff, log, stash) is an independent component with its own event handler and rendering logic. The async git operations run on a separate thread and communicate back via channels. gitui benchmarks itself against the full Linux kernel repository (900k+ commits) and shows startup times under a second.

**[bottom](https://github.com/ClementTsang/bottom)** (10k+ stars) - a system monitor (`btm` command). It's the app that really pushes ratatui's `Chart` widget. CPU history, memory usage, network throughput - all rendered as line charts using braille markers. Bottom runs a background thread that samples `/proc` (or the macOS equivalent) at a configurable interval, and the UI thread re-renders whenever new data arrives. The layout adapts to terminal size, collapsing panels when space is tight.

**[bandwhich](https://github.com/imsnif/bandwhich)** - a network bandwidth monitor. Shows per-process, per-connection, and per-remote-address bandwidth usage in real time. It captures packets using `libpcap`, aggregates data in a background task, and renders tables with the results.

All three share common patterns:

1. **State lives outside the UI.** The application state is a plain struct. Widgets are created fresh each frame from that state.
2. **Events are translated to domain actions.** Key presses become messages like "scroll down" or "toggle view," not direct state mutations.
3. **Background work uses channels.** Expensive operations (git commands, system calls, packet capture) run on separate threads. The UI thread just reads the latest results.
4. **Terminal cleanup is bulletproof.** All of them install panic hooks and handle `SIGINT` to restore the terminal. A crash should never leave your terminal broken.

## Common mistakes

A few things that trip people up:

**Forgetting to filter key release events.** On Windows, every key press generates both a `Press` and a `Release` event. Without the `KeyEventKind::Press` guard, every action fires twice.

**Rendering outside allocated areas.** If your widget writes to coordinates outside its `Rect`, you corrupt other widgets. Always bounds-check. The built-in widgets handle this, but custom widgets need care.

**Blocking the event loop.** If you do a network request or file I/O inside `handle_events` or `ui`, the entire app freezes. Move anything that might block to a background task and communicate via channels.

**Not handling terminal resize.** Crossterm emits `Event::Resize(width, height)` when the terminal window changes size. Ratatui re-queries the terminal size on each `draw()` call, so layouts adapt automatically. But if you cache layout calculations between frames, you need to invalidate on resize.

## Where to go from here

The [ratatui website](https://ratatui.rs) has excellent tutorials - start with the counter app tutorial, then the JSON editor. The [examples directory](https://github.com/ratatui/ratatui/tree/main/examples) in the repo covers every widget and many architectural patterns.

For templates, check [ratatui/templates](https://github.com/ratatui/templates) - they provide starter projects with async event loops, component architectures, and proper error handling already wired up.

The ecosystem crate [tui-textarea](https://github.com/rhysd/tui-textarea) gives you a multi-line text editor widget with vim/emacs keybindings. [tui-input](https://github.com/sayanarijit/tui-input) handles single-line input fields. [ratatui-image](https://github.com/benjajaja/ratatui-image) renders actual images in the terminal using Kitty or Sixel protocols.

Terminal UIs feel anachronistic until you use a good one. Then you realize: no Electron overhead, no layout thrashing, instant startup, works over SSH, and you never leave your terminal. Ratatui makes building them surprisingly pleasant.
