+++
title = "The command pattern in Rust - undo/redo and action queues"
date = 2026-03-03
description = "Turn actions into values you can store, queue, persist and reverse. A pragmatic walk through the command pattern in Rust with undo stacks, composite commands, and the inevitable comparison to closures."

[taxonomies]
tags = ["rust", "design-patterns", "architecture"]
+++

There is a small set of features that quietly demand the same shape of code no matter what application you are writing. Undo and redo. A queue of pending operations a worker drains in the background. A migration runner that can roll forward or back. A macro recorder. An action log you can replay to reconstruct state. Every one of these problems wants the same thing: an action that has been *captured* but not yet *executed*, sitting somewhere as a value, waiting for somebody to call it.

That is the entire idea behind the command pattern. You take a verb - "insert this character at offset 42", "charge this card", "move this file" - and you turn it into a noun. Once it is a noun, you can stick it in a `Vec`, write it to disk, send it across a channel, or hand it back to the user labelled "undo".

Rust gives you two reasonable ways to do this: a trait with `execute` and `undo` methods, or a closure. They are not the same tool, and the difference matters more than people give it credit for.

<!-- more -->

## The trait

The minimum useful version of a command in Rust is a trait with two methods. One does the thing. The other puts the world back the way it was.

```rust
pub trait Command {
    fn execute(&mut self, doc: &mut Document);
    fn undo(&mut self, doc: &mut Document);
}
```

`&mut self` is deliberate. A command often has to record state during `execute` so that `undo` can reverse it. The "delete word" command has no idea *which* word it is deleting until you run it against the actual document, but `undo` needs that text back. So the command stores it on the way through.

Take a text editor. Two commands cover most of what a buffer does:

```rust
pub struct Document {
    pub text: String,
    pub cursor: usize,
}

pub struct Insert {
    pub at: usize,
    pub what: String,
}

impl Command for Insert {
    fn execute(&mut self, doc: &mut Document) {
        doc.text.insert_str(self.at, &self.what);
        doc.cursor = self.at + self.what.len();
    }
    fn undo(&mut self, doc: &mut Document) {
        let end = self.at + self.what.len();
        doc.text.replace_range(self.at..end, "");
        doc.cursor = self.at;
    }
}

pub struct Delete {
    pub at: usize,
    pub len: usize,
    removed: String, // captured during execute
}

impl Command for Delete {
    fn execute(&mut self, doc: &mut Document) {
        let end = self.at + self.len;
        self.removed = doc.text[self.at..end].to_string();
        doc.text.replace_range(self.at..end, "");
        doc.cursor = self.at;
    }
    fn undo(&mut self, doc: &mut Document) {
        doc.text.insert_str(self.at, &self.removed);
        doc.cursor = self.at + self.removed.len();
    }
}
```

Notice that `Delete` only knows what it removed *after* it has run. The command is mutable for the life of the history entry. If you want strict immutability you can split the command into a `Plan` and a `Receipt`, but for almost every codebase I have shipped, mutating the command in place is the cleaner option.

## The history

Once commands are values, "undo" is a one-line operation: pop the last command off a stack and call `undo` on it. "Redo" is the same trick on a second stack.

```rust
pub struct History {
    done: Vec<Box<dyn Command>>,
    redo: Vec<Box<dyn Command>>,
}

impl History {
    pub fn apply(&mut self, mut cmd: Box<dyn Command>, doc: &mut Document) {
        cmd.execute(doc);
        self.done.push(cmd);
        self.redo.clear(); // any new action invalidates the redo stack
    }

    pub fn undo(&mut self, doc: &mut Document) {
        if let Some(mut cmd) = self.done.pop() {
            cmd.undo(doc);
            self.redo.push(cmd);
        }
    }

    pub fn redo(&mut self, doc: &mut Document) {
        if let Some(mut cmd) = self.redo.pop() {
            cmd.execute(doc);
            self.done.push(cmd);
        }
    }
}
```

`Vec<Box<dyn Command>>` is the workhorse here. Each entry is a fat pointer (16 bytes on a 64-bit system: one pointer to the heap-allocated command, one to the vtable for `Command`). `cmd.execute` is a virtual call - the compiler emits an indirect jump through the vtable, which a modern branch predictor handles fine but cannot inline. For an editor that runs maybe a thousand commands per second per user, the dispatch cost is invisible. For a tight inner loop processing millions of events you would think harder. We will come back to that.

The `redo.clear()` line is not optional. It is the rule that makes undo/redo behave the way every editor since vi has behaved: if you undo three things and then type something new, the three things you undid are gone forever. People expect this so deeply that they will file bugs if you do anything else.

## Macro commands

A composite command is a command that holds other commands. This is what "Replace All" is - one macro command made of N `Delete` and N `Insert` operations, presented to the history as a single undoable unit.

```rust
pub struct Macro {
    pub name: String,
    pub steps: Vec<Box<dyn Command>>,
}

impl Command for Macro {
    fn execute(&mut self, doc: &mut Document) {
        for step in &mut self.steps {
            step.execute(doc);
        }
    }
    fn undo(&mut self, doc: &mut Document) {
        // reverse order matters: last in, first out
        for step in self.steps.iter_mut().rev() {
            step.undo(doc);
        }
    }
}
```

Two details worth pointing out. First, `undo` walks the steps in reverse. If a macro inserts then deletes, undo must put the deletion back *before* removing the insertion, otherwise the indices stop lining up. Second, `Macro` is itself a `Command`, so you can nest macros inside macros without changing the history code at all. That is the nicest property of the pattern in practice.

For partial-failure semantics - what if step 5 of 10 panics - you have a choice. Either each command is required to succeed (you validate up front), or `execute` returns `Result<(), Error>` and `Macro` rolls back the steps it already ran. Adding `Result` to the trait is straightforward but worth doing on day one. Retrofitting it later is irritating because every implementor needs touching.

## Persistence: making commands durable

A `Box<dyn Command>` lives in memory. You cannot serde it. The moment you want to save the undo history across editor restarts, or persist a queue of pending jobs to SQLite, the trait-object approach hits a wall. There is no `Serialize for dyn Command` because the deserializer would not know which concrete type to instantiate.

The fix is to keep the trait for runtime polymorphism *and* add an enum that names every command the system knows how to serialize. The enum becomes the wire format.

```rust
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize)]
pub enum CommandKind {
    Insert { at: usize, what: String },
    Delete { at: usize, len: usize, removed: String },
    Macro { name: String, steps: Vec<CommandKind> },
}

impl CommandKind {
    pub fn into_boxed(self) -> Box<dyn Command> {
        match self {
            CommandKind::Insert { at, what } => Box::new(Insert { at, what }),
            CommandKind::Delete { at, len, removed } => {
                Box::new(Delete { at, len, removed })
            }
            CommandKind::Macro { name, steps } => Box::new(Macro {
                name,
                steps: steps.into_iter().map(|s| s.into_boxed()).collect(),
            }),
        }
    }
}
```

Now your history file is just `serde_json::to_writer(file, &history_kinds)?`. On startup you read the JSON, map every `CommandKind` back into a `Box<dyn Command>`, and the application has no idea anything was ever on disk. This is also exactly how `sqlx` migrations, `refinery`, `diesel` and `flyway` work under the hood: each migration is a command with `up` and `down` methods, the schema-history table records which ran in what order, and the deserializer is the migration runner that knows how to find each command by name.

The same shape covers task queues. A row in a `jobs` table holds a `kind` discriminator and a JSON payload. The worker reads the row, deserializes into a `JobCommand` enum, calls `execute`. If it fails, the queue retries or moves to a dead-letter table. There is no special "queue framework" here - it is the command pattern with a database for storage instead of a `Vec`.

## Closures: the tempting alternative

You can do most of this with closures. `Box<dyn FnOnce(&mut Document)>` is, in a real sense, a command. Capturing variables in the closure replaces fields on a struct. For one-shot fire-and-forget tasks, a closure is shorter and clearer:

```rust
let queue: Vec<Box<dyn FnOnce(&mut Document)>> = vec![
    Box::new(|d| d.text.push_str("hello")),
    Box::new(|d| d.cursor = 0),
];
```

For undo, closures fall apart. A closure has no `undo` method - it is a single `call_once` and that is the whole interface. You could carry a pair of closures, `(do_it, undo_it)`, but now you are reinventing the trait with worse ergonomics: you cannot serialize the pair, you cannot ask "what kind of command was this", you cannot pattern-match on the history to coalesce two consecutive `Insert`s into one entry the way real editors do.

The line I draw: closures for transient action queues that never need to be persisted, inspected, or reversed. Trait objects (with a serializable enum mirror) for everything else. You see this split in real Rust code. `tokio::task::spawn` takes a future, which is essentially a closure-shaped command - fire it and forget. `sqlx::migrate!` generates an enum of named, serializable commands - because migrations need names, ordering, and a record of what ran.

## What it costs

Every `Box<dyn Command>` is a heap allocation. For a 64-bit target, the box itself is 8 bytes for the pointer and 8 bytes for the vtable pointer, plus whatever the command struct holds. An `Insert { at: 42, what: "x".to_string() }` is 8 + 24 = 32 bytes for the data plus the heap allocation for the one-byte string buffer plus the box. A 10,000-entry history of single-character inserts is on the order of 600 KB of heap. Real editors coalesce - a run of one-character inserts becomes one `Insert { at, what: "the quick brown fox" }` after a 500 ms idle timer - and the size collapses.

Dispatch through a vtable is one indirect call. If you benchmark a tight loop that runs a million commands you can measure the difference against monomorphised generics, but in any application where commands represent *user-meaningful actions* the cost is dwarfed by everything else. If you really need monomorphisation, an enum-based command (`match cmd { Insert {..} => ..., Delete {..} => ... }`) is the same shape as the serializable mirror and lets the compiler inline. You give up open extensibility - third parties cannot add new commands without modifying the enum - but you get a static dispatch and a built-in serialization story for free.

The pattern earns its keep the first time a feature lands that needs it: an undo button, a "save unsent jobs to disk", a migration roll-back, a macro recorder. None of those features are hard if your codebase already speaks in commands. All of them are an architectural rewrite if it speaks in direct method calls. You can build the whole thing in roughly the amount of code shown above, which is why I treat it as one of the cheapest architectural insurance policies Rust offers.
