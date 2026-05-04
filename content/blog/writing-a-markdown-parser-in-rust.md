+++
title = "Writing a Markdown parser in Rust"
date = 2025-01-12
description = "Building a subset Markdown-to-HTML parser from scratch in ~280 lines of Rust, then comparing it to what pulldown-cmark does under the hood."

[taxonomies]
tags = ["rust", "parsing", "markdown"]
+++

Markdown looks simple. You read it, you write it, you barely think about it. But the moment you try to turn `**bold**` into `<strong>bold</strong>` programmatically, you realize there's a surprising amount of ambiguity hiding in that simplicity. That's exactly why I decided to write a parser from scratch - not to replace [pulldown-cmark](https://crates.io/crates/pulldown-cmark), but to understand what it actually has to deal with.

The result is around 280 lines of Rust that handle headings, paragraphs, code blocks, blockquotes, unordered lists, and inline formatting (bold, italic, code spans, links). It's a subset, intentionally. But building even a subset teaches you more about parsing than reading a hundred blog posts about parser theory.

<!-- more -->

## The architecture: two passes

Most Markdown parsers - including ours - use a two-pass approach. The [CommonMark spec](https://spec.commonmark.org/) itself recommends this in its parsing strategy appendix:

**Pass 1 - Block parsing:** Walk through the document line by line. Identify block-level structures: headings, code blocks, blockquotes, lists, paragraphs. Build a tree.

**Pass 2 - Inline parsing:** For each block that contains text (headings, paragraphs, list items), parse the inline formatting: bold, italic, code, links.

This split exists because block structure is determined by line prefixes (`#`, `` ``` ``, `>`, `- `), while inline structure is determined by delimiter characters within text (`*`, `` ` ``, `[`). Mixing them in a single pass would be painful.

## Data structures

Before any parsing logic, we need types. Rust enums are perfect here - a Markdown document is literally a sum type. A block is *either* a heading *or* a paragraph *or* a code block:

```rust
use std::fmt::Write;

#[derive(Debug)]
enum Block {
    Heading(u8, Vec<Inline>),       // level, content
    Paragraph(Vec<Inline>),
    CodeBlock(String, String),       // language, raw content
    Blockquote(Vec<Block>),          // recursive!
    UnorderedList(Vec<Vec<Inline>>), // list of items
}

#[derive(Debug)]
enum Inline {
    Text(String),
    Bold(Vec<Inline>),   // recursive - bold can contain italic
    Italic(Vec<Inline>),
    Code(String),        // not recursive - code is literal
    Link(String, String), // display text, url
}
```

A few things to notice. `Blockquote` contains `Vec<Block>` - blockquotes are recursive. A blockquote can contain headings, paragraphs, even nested blockquotes. Same idea with `Bold` and `Italic` containing `Vec<Inline>` - you can have `***bold and italic***` or `**bold with `code` inside**`.

But `Code` is just a `String`. Inside backticks, nothing is interpreted. `*not bold*` inside backticks stays as literal text. This is an important semantic distinction that falls naturally out of the type system.

You can check how much memory these enums take:

```rust
println!("Block: {} bytes", std::mem::size_of::<Block>());
println!("Inline: {} bytes", std::mem::size_of::<Inline>());
```

On 64-bit systems, `Block` is 56 bytes and `Inline` is 56 bytes. Rust sizes the enum to fit the largest variant plus the discriminant tag. `CodeBlock(String, String)` is two `String`s (24 bytes each = 48 bytes) plus alignment and discriminant - that drives the size. Every `Block::Heading` also occupies 56 bytes even though it only needs a `u8` and a `Vec` (25 bytes). That's the tradeoff with enums: uniform size means some variants waste space.

## Block parsing: line by line

The block parser walks through lines, matching prefixes to determine what each block is. The core is a `while` loop with index tracking:

```rust
fn parse_blocks(input: &str) -> Vec<Block> {
    let lines: Vec<&str> = input.lines().collect();
    let mut blocks = Vec::new();
    let mut i = 0;

    while i < lines.len() {
        let line = lines[i];

        if line.trim().is_empty() {
            i += 1;
            continue;
        }

        // Fenced code block: collect until closing ```
        if line.trim_start().starts_with("```") {
            let lang = line.trim_start().trim_start_matches('`').trim().to_string();
            let mut content = String::new();
            i += 1;
            while i < lines.len() && !lines[i].trim_start().starts_with("```") {
                if !content.is_empty() {
                    content.push('\n');
                }
                content.push_str(lines[i]);
                i += 1;
            }
            if i < lines.len() {
                i += 1; // skip closing ```
            }
            blocks.push(Block::CodeBlock(lang, content));
            continue;
        }

        // ATX heading: # through ######
        if line.starts_with('#') {
            let level = line.chars().take_while(|&c| c == '#').count() as u8;
            if level <= 6 {
                let text = &line[level as usize..];
                let text = text.trim_start();
                blocks.push(Block::Heading(level, parse_inlines(text)));
                i += 1;
                continue;
            }
        }

        // Blockquote
        if line.starts_with("> ") || line == ">" {
            let mut quote_lines = Vec::new();
            while i < lines.len()
                && (lines[i].starts_with("> ") || lines[i] == ">")
            {
                let stripped = if lines[i] == ">" { "" } else { &lines[i][2..] };
                quote_lines.push(stripped);
                i += 1;
            }
            let inner = quote_lines.join("\n");
            blocks.push(Block::Blockquote(parse_blocks(&inner)));
            continue;
        }

        // Unordered list
        if line.starts_with("- ") {
            let mut items = Vec::new();
            while i < lines.len() && lines[i].starts_with("- ") {
                items.push(parse_inlines(&lines[i][2..]));
                i += 1;
            }
            blocks.push(Block::UnorderedList(items));
            continue;
        }

        // Paragraph: everything else, collected until empty/special line
        let mut para_lines = Vec::new();
        while i < lines.len()
            && !lines[i].trim().is_empty()
            && !lines[i].starts_with('#')
            && !lines[i].starts_with("```")
            && !lines[i].starts_with("> ")
            && !lines[i].starts_with("- ")
        {
            para_lines.push(lines[i]);
            i += 1;
        }
        if !para_lines.is_empty() {
            let text = para_lines.join(" ");
            blocks.push(Block::Paragraph(parse_inlines(&text)));
        }
    }

    blocks
}
```

The order of checks matters. Code blocks must be detected before headings, because inside a code block `# this is not a heading` is just text. Paragraphs are the fallback - if a line doesn't match anything else, it's paragraph content.

Notice how blockquotes use recursion: we strip the `> ` prefix from each line, rejoin them, and call `parse_blocks` on the result. This means a blockquote containing a heading, a list, and a code block all just works. The same parsing logic handles both top-level content and nested content.

One subtle detail: paragraph collection joins lines with spaces (`para_lines.join(" ")`). This means a paragraph written across multiple lines in the source gets merged into one string before inline parsing. That's the standard Markdown behavior - line breaks within a paragraph are soft wraps, not hard breaks.

## Inline parsing: character by character

This is where it gets interesting. Block parsing is mostly prefix matching, but inline parsing requires tracking delimiter pairs across the text:

```rust
fn parse_inlines(input: &str) -> Vec<Inline> {
    let mut result = Vec::new();
    let chars: Vec<char> = input.chars().collect();
    let mut i = 0;
    let mut buf = String::new();

    while i < chars.len() {
        // Bold: **...**
        if i + 1 < chars.len() && chars[i] == '*' && chars[i + 1] == '*' {
            if !buf.is_empty() {
                result.push(Inline::Text(std::mem::take(&mut buf)));
            }
            i += 2;
            let mut inner = String::new();
            while i + 1 < chars.len()
                && !(chars[i] == '*' && chars[i + 1] == '*')
            {
                inner.push(chars[i]);
                i += 1;
            }
            if i + 1 < chars.len() {
                i += 2; // skip closing **
            }
            result.push(Inline::Bold(parse_inlines(&inner)));
            continue;
        }

        // Italic: *...*
        if chars[i] == '*' {
            if !buf.is_empty() {
                result.push(Inline::Text(std::mem::take(&mut buf)));
            }
            i += 1;
            let mut inner = String::new();
            while i < chars.len() && chars[i] != '*' {
                inner.push(chars[i]);
                i += 1;
            }
            if i < chars.len() {
                i += 1; // skip closing *
            }
            result.push(Inline::Italic(parse_inlines(&inner)));
            continue;
        }

        // Inline code: `...`
        if chars[i] == '`' {
            if !buf.is_empty() {
                result.push(Inline::Text(std::mem::take(&mut buf)));
            }
            i += 1;
            let mut code = String::new();
            while i < chars.len() && chars[i] != '`' {
                code.push(chars[i]);
                i += 1;
            }
            if i < chars.len() {
                i += 1;
            }
            result.push(Inline::Code(code));
            continue;
        }

        // Link: [text](url)
        if chars[i] == '[' {
            if !buf.is_empty() {
                result.push(Inline::Text(std::mem::take(&mut buf)));
            }
            i += 1;
            let mut text = String::new();
            while i < chars.len() && chars[i] != ']' {
                text.push(chars[i]);
                i += 1;
            }
            if i + 1 < chars.len() && chars[i] == ']' && chars[i + 1] == '(' {
                i += 2;
                let mut url = String::new();
                while i < chars.len() && chars[i] != ')' {
                    url.push(chars[i]);
                    i += 1;
                }
                if i < chars.len() {
                    i += 1;
                }
                result.push(Inline::Link(text, url));
            } else {
                buf.push('[');
                buf.push_str(&text);
                if i < chars.len() {
                    buf.push(chars[i]);
                    i += 1;
                }
            }
            continue;
        }

        buf.push(chars[i]);
        i += 1;
    }

    if !buf.is_empty() {
        result.push(Inline::Text(buf));
    }

    result
}
```

The critical ordering decision: check `**` before `*`. If you check single `*` first, `**bold**` gets parsed as "empty italic, text `bold`, empty italic" - completely wrong. Two-character delimiters must be tested before their single-character subsets.

The `std::mem::take(&mut buf)` pattern is worth calling out. When we hit a delimiter, any accumulated plain text needs to be flushed to the result before we start parsing the delimited content. `std::mem::take` replaces `buf` with an empty `String` and gives us the old value - no cloning, no allocation. The existing `String`'s heap buffer moves directly into the `Inline::Text` variant.

The recursive call in `Bold` and `Italic` (`parse_inlines(&inner)`) is what enables nested formatting. `**bold with *italic* inside**` first gets the inner content `bold with *italic* inside` extracted, then that string gets parsed again, finding the italic markers.

For links, there's a fallback path. If we see `[` but don't find the `](url)` pattern, we treat the whole thing as plain text. This matters because square brackets appear in normal text too.

## HTML rendering

With the AST built, rendering is straightforward pattern matching:

```rust
fn render_html(blocks: &[Block]) -> String {
    let mut out = String::new();
    for block in blocks {
        match block {
            Block::Heading(level, inlines) => {
                write!(out, "<h{}>", level).unwrap();
                render_inlines(&mut out, inlines);
                writeln!(out, "</h{}>", level).unwrap();
            }
            Block::Paragraph(inlines) => {
                out.push_str("<p>");
                render_inlines(&mut out, inlines);
                out.push_str("</p>\n");
            }
            Block::CodeBlock(lang, content) => {
                if lang.is_empty() {
                    out.push_str("<pre><code>");
                } else {
                    write!(out, "<pre><code class=\"language-{}\">",
                        escape_html(lang)).unwrap();
                }
                out.push_str(&escape_html(content));
                out.push_str("</code></pre>\n");
            }
            Block::Blockquote(inner) => {
                out.push_str("<blockquote>\n");
                out.push_str(&render_html(inner));
                out.push_str("</blockquote>\n");
            }
            Block::UnorderedList(items) => {
                out.push_str("<ul>\n");
                for item in items {
                    out.push_str("<li>");
                    render_inlines(&mut out, item);
                    out.push_str("</li>\n");
                }
                out.push_str("</ul>\n");
            }
        }
    }
    out
}

fn render_inlines(out: &mut String, inlines: &[Inline]) {
    for inline in inlines {
        match inline {
            Inline::Text(t) => out.push_str(&escape_html(t)),
            Inline::Bold(inner) => {
                out.push_str("<strong>");
                render_inlines(out, inner);
                out.push_str("</strong>");
            }
            Inline::Italic(inner) => {
                out.push_str("<em>");
                render_inlines(out, inner);
                out.push_str("</em>");
            }
            Inline::Code(c) => {
                out.push_str("<code>");
                out.push_str(&escape_html(c));
                out.push_str("</code>");
            }
            Inline::Link(text, url) => {
                write!(out, "<a href=\"{}\">", escape_html(url)).unwrap();
                out.push_str(&escape_html(text));
                out.push_str("</a>");
            }
        }
    }
}

fn escape_html(input: &str) -> String {
    input
        .replace('&', "&amp;")
        .replace('<', "&lt;")
        .replace('>', "&gt;")
        .replace('"', "&quot;")
}
```

The `write!` macro on `String` never actually fails (it returns `Err` only if the `Write` impl does, and `String`'s implementation is infallible), so the `.unwrap()` calls are safe. Some people prefer `write!(...).ok();` to suppress the warning, but I'd rather make the infallibility explicit.

`escape_html` deserves mention. If you skip this, you have an XSS vector. Markdown that contains `<script>alert(1)</script>` must render as escaped text, not as an actual script tag. The replacement order matters too - `&` must be replaced first, otherwise you'd double-escape: `&lt;` would become `&amp;lt;`.

## Putting it together

```rust
fn main() {
    let input = r#"# Hello World

This is a **bold** and *italic* paragraph with `inline code`.

## Links

Check out [Rust](https://www.rust-lang.org) for more info.

```rust
fn main() {
    println!("Hello from a code block!");
}
```

> Blockquotes can contain **formatted** text
> and span multiple lines.

- First item
- Second with **bold**
- Third with `code` and a [link](https://example.com)
"#;

    let blocks = parse_blocks(input);
    let html = render_html(&blocks);
    println!("{html}");
}
```

Running this produces clean HTML:

```html
<h1>Hello World</h1>
<p>This is a <strong>bold</strong> and <em>italic</em> paragraph
with <code>inline code</code>.</p>
<h2>Links</h2>
<p>Check out <a href="https://www.rust-lang.org">Rust</a> for more info.</p>
<pre><code class="language-rust">fn main() {
    println!("Hello from a code block!");
}</code></pre>
<blockquote>
<p>Blockquotes can contain <strong>formatted</strong> text
and span multiple lines.</p>
</blockquote>
<ul>
<li>First item</li>
<li>Second with <strong>bold</strong></li>
<li>Third with <code>code</code> and a <a href="https://example.com">link</a></li>
</ul>
```

280 lines. Handles the most common Markdown constructs. But there are cracks.

## Where this parser breaks

Try this input:

```markdown
This is *italic with **bold inside** it*
```

Our parser handles it fine because of the recursive `parse_inlines` call. But what about:

```markdown
This is **bold with *italic* and more bold**
```

Also fine. Now try:

```markdown
*foo **bar* baz**
```

What should this produce? According to [CommonMark](https://spec.commonmark.org/0.31.2/#emphasis-and-strong-emphasis), the answer involves the "delimiter run" algorithm - a 17-rule specification for how `*` and `_` characters interact based on surrounding whitespace and punctuation, left-flanking vs right-flanking runs, and the "multiple of 3" rule for preventing certain nestings. Our simple "find the next matching delimiter" approach gets the easy cases right but fails on these edge cases.

Other things we skip entirely:

- **Setext headings** (underlined with `===` or `---`)
- **Ordered lists** and nested lists with indentation
- **Reference-style links** (`[text][id]` with `[id]: url` defined elsewhere)
- **Images** (`![alt](src)`)
- **Tight vs loose lists** (whether list items get wrapped in `<p>` tags depends on blank lines between them)
- **HTML blocks** (raw HTML passthrough - the CommonMark spec defines seven different types with different rules)
- **Link titles** (`[text](url "title")`)
- **Autolinks** (`<https://example.com>`)
- **Hard line breaks** (trailing two spaces or backslash)

Each of these adds complexity. The loose list rules alone are responsible for a significant chunk of parser bugs across implementations. And the full emphasis algorithm is why [pulldown-cmark's source](https://github.com/pulldown-cmark/pulldown-cmark) weighs in at roughly 15,000 lines of Rust.

## How pulldown-cmark does it differently

[pulldown-cmark](https://github.com/pulldown-cmark/pulldown-cmark) (v0.13.3 at time of writing) takes a fundamentally different approach from our AST-based parser.

**Pull-based event stream.** Where we build `Vec<Block>` containing `Vec<Inline>` - a full tree in memory - pulldown-cmark implements `Iterator<Item = Event>`. You get events like `Start(Heading(H1))`, `Text("hello")`, `End(Heading(H1))` one at a time. This means you can process a 10MB Markdown file without holding the whole document tree in memory. The consumer controls the pace.

```rust
use pulldown_cmark::{Parser, Event, Tag};

let input = "# Hello **world**";
let parser = Parser::new(input);

for event in parser {
    match event {
        Event::Start(tag) => print!("<{}>", tag_name(&tag)),
        Event::End(tag) => print!("</{}>", tag_name(&tag)),
        Event::Text(text) => print!("{}", text),
        _ => {}
    }
}
```

**Zero-copy strings.** Our parser builds owned `String` values everywhere. pulldown-cmark uses `CowStr` - a copy-on-write string that's usually a borrowed slice of the original input. When the parser yields `Event::Text("hello")`, that "hello" is a pointer into your source string, not a new heap allocation. It only allocates when the text needs transformation (like entity decoding `&amp;` into `&`). For a 10KB Markdown file, this is the difference between dozens of allocations and nearly zero.

**SIMD scanning.** On x86_64, pulldown-cmark uses SIMD instructions to scan for special characters (`*`, `_`, `` ` ``, `[`, `<`, `&`, `\n`). Instead of checking one byte at a time, it checks 16 bytes simultaneously. This is behind a feature flag (`simd`), but it's one of the reasons pulldown-cmark can process around 500,000 characters per second.

**Two-pass internally.** Despite the pull-based external API, pulldown-cmark internally uses the same two-pass strategy we do. `firstpass.rs` builds a block-level tree, then inline parsing happens lazily as events are requested. The architecture is the same - the difference is that ours materializes everything upfront while pulldown-cmark streams it.

If you're building a real tool that processes Markdown - a static site generator, a documentation renderer, a note-taking app - use pulldown-cmark. And if you've read the [adapter pattern post](/blog/the-adapter-pattern-in-rust-wrapping-external-apis/), you know the right way to integrate it: wrap it behind a trait so your domain code doesn't depend on pulldown-cmark's types directly.

## What you learn from writing your own

The point of building this was never to compete with production parsers. It was to understand what parsing actually involves:

**Ordering matters everywhere.** Check `**` before `*`. Check code blocks before headings. Check empty lines before paragraph continuation. Get any of these wrong and the output is subtly broken in ways that are hard to debug. If you ever get stuck debugging parser state, the `dbg!` macro from the [debugging post](/blog/debugging-rust-beyond-println/) is your best friend here - drop a `dbg!(&blocks)` after `parse_blocks` and you'll see exactly what the parser produced.

**Recursion falls naturally out of the problem.** Blockquotes containing blocks, bold containing inlines - these aren't design choices, they're reflections of how Markdown actually nests. Rust enums make this recursive structure type-safe in a way that's hard to achieve in languages where you'd use a generic tree node.

**The gap between "works on my examples" and "handles all valid input" is enormous.** Our parser is ~280 lines. pulldown-cmark is ~15,000. That 50x ratio mostly comes from edge cases - the delimiter algorithm, loose lists, HTML blocks, entity decoding, source mapping.

**Separation of parsing and rendering pays off immediately.** Because we have a clean AST (`Vec<Block>`), adding a new output format - say, terminal-colored output or LaTeX - means writing a new render function without touching the parser. pulldown-cmark takes this further with the event stream: the [pulldown-cmark-to-cmark](https://crates.io/crates/pulldown-cmark-to-cmark) crate converts events back to Markdown, and various other crates convert to different formats.

If you want to push this further, try adding ordered lists (you'll discover the indentation tracking problem), or images (easy - almost identical to links), or the full emphasis algorithm from the [CommonMark spec](https://spec.commonmark.org/0.31.2/#emphasis-and-strong-emphasis) (hard - you'll understand why it took years to get right).

The full parser code fits in a single file. Copy it, run it, break it, add to it. That's how you learn what Markdown actually is - not by reading the spec, but by watching your parser fail on inputs you thought were simple.
