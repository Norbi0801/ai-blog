+++
title = "Zola - why I chose a Rust static site generator for my blog"
date = 2026-05-04
description = "A practical comparison of static site generators and a walkthrough of setting up Zola from scratch with templates, Sass, and GitHub Pages deployment."

[taxonomies]
tags = ["rust", "zola", "static-site-generator", "devops"]
+++

{% raw %}
I wanted a blog. Not a web app, not a CMS, not a React project with 400MB of node_modules. Just markdown files that turn into HTML pages fast enough that I never have to think about the build step.

That narrowed things down to static site generators. And after spending a weekend evaluating the big names - Hugo, Jekyll, Next.js, Astro, and Zola - I went with Zola. Here's why, and how to set it up from scratch.

<!-- more -->

## The SSG landscape in 2026

There are dozens of static site generators. For a developer blog with no dynamic content, these are the five that actually matter:

| Generator | Language | GitHub Stars | Single Binary | JS Required | Build Speed |
|-----------|----------|-------------|---------------|-------------|-------------|
| [Hugo](https://github.com/gohugoio/hugo) | Go | ~87k | Yes | No | Very fast |
| [Jekyll](https://github.com/jekyll/jekyll) | Ruby | ~49k | No | No | Slow |
| [Next.js](https://github.com/vercel/next.js) | JS/TS | ~132k | No | Yes | Medium |
| [Astro](https://github.com/withastro/astro) | JS/TS | ~58k | No | Optional | Fast |
| [Zola](https://github.com/getzola/zola) | Rust | ~17k | Yes | No | Very fast |

Stars don't tell the whole story. Next.js and Astro have massive numbers because they're full web frameworks, not just site generators. For a plain blog, most of their features are dead weight.

## Why not the others

**Jekyll** was the default choice for years, especially with GitHub Pages supporting it natively. But it requires Ruby, Bundler, and gem management. On a fresh machine, getting Jekyll running means installing Ruby version managers, dealing with native extension compilation, and hoping your gems resolve. For a tool that just converts markdown to HTML, that's a lot of ceremony. Build times for larger sites (500+ pages) routinely hit 30-60 seconds.

**Next.js** is great for building web applications with static export as a feature. But it pulls in React, webpack (or turbopack), and the entire Node.js ecosystem. Running `npx create-next-app` for a blog gives you a `node_modules` directory larger than most operating system installers. You get server components, API routes, image optimization pipelines - none of which a blog needs. It's the textbook case of picking a framework for its ecosystem when you need 5% of its capabilities. If you read my earlier post on [the build vs. buy decision framework](/blog/build-vs-buy---when-to-use-a-library-and-when-to-write-your-/), this falls squarely into "ecosystem gravity pulling you toward complexity you don't need."

**Astro** is the closest competitor for content-focused sites. Its island architecture and zero-JS-by-default philosophy align with what a blog needs. The template syntax is clean, the content collections API is well designed, and build times are reasonable. My issue: it still requires Node.js. You need `npm install`, a `package.json`, and the whole dependency tree. For a blog that's pure markdown and CSS, I didn't want a JavaScript runtime in the pipeline at all.

**Hugo** is the real alternative. Single binary, blazing fast, massive theme ecosystem, battle-tested by thousands of sites. If you're reading this and thinking "just use Hugo," that's a fair take. Two things pushed me away:

First, Go templates. Hugo uses Go's `text/template` and `html/template` packages, and the syntax is genuinely painful for anything beyond trivial layouts:

```html
{{ range where .Pages "Type" "posts" }}
  {{ if and (not .Draft) (gt .Date.Unix 0) }}
    <a href="{{ .Permalink }}">{{ .Title }}</a>
  {{ end }}
{{ end }}
```

That nested `{{ if and ... }}` pattern gets worse as logic grows. There's no pipe operator for chaining conditions, variable scoping is confusing (the meaning of `.` changes inside `range` blocks), and debugging template errors produces cryptic messages.

Second, Hugo's configuration surface area is enormous. The [documentation](https://gohugo.io/documentation/) covers page bundles, leaf bundles, branch bundles, headless bundles, content adapters, render hooks, shortcodes, partials, base templates, output formats, and more. For a simple blog, you need maybe 10% of this, but you still have to understand the rest to know what you can safely ignore.

## Why Zola

[Zola](https://www.getzola.org/) is a static site generator written in Rust by [Vincent Prouillet](https://github.com/Keats) (who also created the [Tera](https://github.com/Keats/tera) template engine and [jsonwebtoken](https://github.com/Keats/jsonwebtoken) crate). The current version is [v0.22.1](https://github.com/getzola/zola/releases/tag/v0.22.1), and while its ~17k GitHub stars look modest next to Hugo's 87k, the project is actively maintained and has everything a blog needs out of the box.

### Single binary, zero dependencies

```bash
# macOS
brew install zola

# Arch Linux
pacman -S zola

# Or grab the binary directly
wget https://github.com/getzola/zola/releases/download/v0.22.1/zola-v0.22.1-x86_64-unknown-linux-gnu.tar.gz
tar xzf zola-v0.22.1-x86_64-unknown-linux-gnu.tar.gz
./zola --version
```

That's it. No runtime, no package manager, no dependencies. The binary is statically linked. You can copy it to a server, a CI runner, or a USB stick and it works. If you've read my post on [Rust in production](/blog/rust-in-production---what-companies-actually-use-it-for/), Zola is a perfect example of the pattern - a focused tool where Rust's single-binary deployment model eliminates entire categories of "works on my machine" problems.

### Build speed

For a 50-page blog, Zola builds the entire site in roughly 30-40ms. [Benchmarks from tqdev.com](https://www.tqdev.com/2023-zola-ssg-is-4x-faster-than-hugo/) showed Zola building 4x faster than Hugo in comparable tests - 36ms vs 178ms for a 50-page site. On CPU load, the difference was even more dramatic, roughly 11x lower.

For a personal blog this barely matters - even Jekyll's 30-second builds are tolerable. But during development, when you're running `zola serve` with live reload, sub-50ms rebuilds mean your browser updates before you can switch to it. The feedback loop is instant.

The speed comes from Rust's zero-cost abstractions and the [syntect](https://github.com/trishume/syntect) library for syntax highlighting (the same library that powers syntax highlighting in [bat](https://github.com/sharkdp/bat) and parts of VS Code). There's no interpreter startup, no JIT warmup, no garbage collector pauses.

### Tera templates

Zola uses [Tera](https://keats.github.io/tera/), a template engine inspired by Jinja2 and Django templates. If you've used Python's Jinja2, Liquid, or Twig, Tera feels immediately familiar:

```html
{% extends "base.html" %}

{% block content %}
<article>
  <h1>{{ page.title }}</h1>
  <time datetime="{{ page.date | date(format='%Y-%m-%d') }}">
    {{ page.date | date(format="%B %e, %Y") }}
  </time>

  {% if page.taxonomies.tags %}
    <div class="tags">
      {% for tag in page.taxonomies.tags %}
        <a href="{{ get_taxonomy_url(kind='tags', name=tag) }}">#{{ tag }}</a>
      {% endfor %}
    </div>
  {% endif %}

  <div class="content">
    {{ page.content | safe }}
  </div>
</article>
{% endblock content %}
```

Compare that to the Hugo equivalent and the readability difference is significant. Variable names are explicit (`page.title` not `.Title`), filters use the pipe syntax (`| date(format="...")`), and block inheritance works the way you'd expect from any Jinja-family engine.

Tera also supports macros for reusable template fragments:

```html
{% macro post_card(page) %}
<article class="post-card">
  <a href="{{ page.permalink }}">
    <h2>{{ page.title }}</h2>
    <p>{{ page.description }}</p>
    <time>{{ page.date | date(format="%Y-%m-%d") }}</time>
  </a>
</article>
{% endmacro %}
```

You import and call macros with `{% import "macros.html" as macros %}` and `{{ macros::post_card(page=page) }}`. Clean, testable, composable.

### Sass built-in

Any `.scss` or `.sass` files in the `sass/` directory are compiled automatically. No `node-sass`, no `dart-sass`, no build step configuration:

```scss
// sass/style.scss
$font-stack: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
$color-bg: #1a1a2e;
$color-text: #e0e0e0;
$color-accent: #e94560;
$max-width: 720px;

body {
  font-family: $font-stack;
  background: $color-bg;
  color: $color-text;
  max-width: $max-width;
  margin: 0 auto;
  padding: 2rem;
  line-height: 1.7;
}

a {
  color: $color-accent;
  text-decoration: none;
  &:hover { text-decoration: underline; }
}

pre {
  background: darken($color-bg, 5%);
  padding: 1rem;
  border-radius: 4px;
  overflow-x: auto;
}
```

Set `compile_sass = true` in `config.toml` and Zola handles the rest. The compiled CSS lands in `public/style.css`. No config files, no plugins, no separate compilation step.

### Zero JavaScript in output

Zola generates pure HTML and CSS. No client-side JavaScript, no hydration, no framework runtime. The output is static files that any web server - Nginx, Caddy, GitHub Pages, a Raspberry Pi running busybox httpd - can serve directly.

This isn't just philosophical minimalism. It means:
- Perfect Lighthouse scores without effort
- No Content Security Policy headaches from inline scripts
- Pages work with JavaScript disabled
- Smaller transfer sizes (a typical blog page is 15-30KB total)
- No supply chain risk from client-side dependencies

If you need interactive elements, you can add `<script>` tags in your templates. Zola doesn't prevent JavaScript - it just doesn't force it on you.

### Built-in features that matter

Zola ships with syntax highlighting (via [syntect](https://github.com/trishume/syntect), supporting 200+ languages), an automatic table of contents generator, taxonomy support (tags, categories, or any custom taxonomy), Atom/RSS feed generation, sitemap generation, a search index builder (using [elasticlunr.js](http://elasticlunr.com/)), automatic slug generation, and a built-in link checker.

Each of these would be a plugin, a gem, or an npm package in other generators. In Zola, they're compiled into the binary.

## Setup from zero

Here's the full walkthrough. By the end you'll have a working blog with a custom template, styled with Sass, ready for deployment.

### Initialize the project

```bash
zola init myblog
cd myblog
```

This creates the directory structure:

```
myblog/
  config.toml
  content/
  sass/
  static/
  templates/
  themes/
```

Every directory has a purpose: `content/` holds your markdown, `templates/` holds your Tera templates, `sass/` holds your stylesheets, `static/` holds assets served as-is (images, fonts, favicons), and `themes/` holds third-party themes if you use one.

### config.toml

The only required field is `base_url`. Here's a practical starting configuration:

```toml
base_url = "https://yourusername.github.io"
title = "Your Blog"
description = "A dev blog about things that matter."
default_language = "en"
compile_sass = true
minify_html = true
generate_feeds = true
feed_filenames = ["atom.xml"]

[markdown]
highlight_code = true
smart_punctuation = false

[markdown.highlighting]
theme = "base16-ocean-dark"

[extra]
author = "Your Name"
```

A few notes on these choices:
- `compile_sass = true` enables the built-in Sass compiler
- `minify_html = true` strips whitespace from generated HTML (smaller files, free performance)
- `generate_feeds = true` with `feed_filenames = ["atom.xml"]` creates an Atom feed - Atom is more standardized than RSS 2.0 and better supported by modern readers
- `smart_punctuation = false` - I keep this off because I want full control over the output. When enabled, it converts straight quotes to curly quotes and `--` to en-dashes automatically
- `highlight_code = true` with a theme gives you syntax highlighting with no JavaScript (it's all done at build time using inline CSS)

The full configuration reference is at [getzola.org/documentation/getting-started/configuration](https://www.getzola.org/documentation/getting-started/configuration/).

### Base template

Create `templates/base.html`:

```html
<!DOCTYPE html>
<html lang="{{ lang }}">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>{% block title %}{{ config.title }}{% endblock title %}</title>
  <meta name="description" content="{% block description %}{{ config.description }}{% endblock description %}">
  <link rel="stylesheet" href="{{ get_url(path='style.css') }}">
  <link rel="alternate" type="application/atom+xml"
        title="{{ config.title }}"
        href="{{ get_url(path='atom.xml') }}">
</head>
<body>
  <header>
    <nav>
      <a href="{{ config.base_url }}">{{ config.title }}</a>
    </nav>
  </header>
  <main>
    {% block content %}{% endblock content %}
  </main>
  <footer>
    <p>&copy; {{ now() | date(format="%Y") }} {{ config.extra.author }}</p>
  </footer>
</body>
</html>
```

Key things happening here:
- `{% block title %}` and `{% block content %}` define override points for child templates
- `get_url(path='style.css')` generates the correct URL regardless of whether you're serving from a subdirectory
- `now() | date(format="%Y")` gives you the current year at build time - no JavaScript Date() needed
- The Atom feed link in `<head>` lets RSS readers auto-discover your feed

### Section and page templates

Zola has two core content types: **sections** (directories with `_index.md`) and **pages** (individual `.md` files). Create `templates/index.html` for the homepage:

```html
{% extends "base.html" %}

{% block content %}
<h1>Latest posts</h1>
{% set blog = get_section(path="blog/_index.md") %}
{% for page in blog.pages %}
  <article class="post-preview">
    <h2><a href="{{ page.permalink }}">{{ page.title }}</a></h2>
    <time datetime="{{ page.date | date(format='%Y-%m-%d') }}">
      {{ page.date | date(format="%B %e, %Y") }}
    </time>
    {% if page.description %}
      <p>{{ page.description }}</p>
    {% endif %}
    {% if page.taxonomies.tags %}
      <div class="tags">
        {% for tag in page.taxonomies.tags %}
          <span class="tag">#{{ tag }}</span>
        {% endfor %}
      </div>
    {% endif %}
  </article>
{% endfor %}
{% endblock content %}
```

And `templates/page.html` for individual posts:

```html
{% extends "base.html" %}

{% block title %}{{ page.title }} - {{ config.title }}{% endblock title %}
{% block description %}{{ page.description }}{% endblock description %}

{% block content %}
<article>
  <header>
    <h1>{{ page.title }}</h1>
    <time datetime="{{ page.date | date(format='%Y-%m-%d') }}">
      {{ page.date | date(format="%B %e, %Y") }}
    </time>
    {% if page.taxonomies.tags %}
      <div class="tags">
        {% for tag in page.taxonomies.tags %}
          <a href="{{ get_taxonomy_url(kind='tags', name=tag) }}">#{{ tag }}</a>
        {% endfor %}
      </div>
    {% endif %}
    {% if page.reading_time %}
      <span class="reading-time">{{ page.reading_time }} min read</span>
    {% endif %}
  </header>

  <div class="content">
    {{ page.content | safe }}
  </div>

  {% if page.lower or page.higher %}
  <nav class="post-nav">
    {% if page.lower %}
      <a href="{{ page.lower.permalink }}">&larr; {{ page.lower.title }}</a>
    {% endif %}
    {% if page.higher %}
      <a href="{{ page.higher.permalink }}">{{ page.higher.title }} &rarr;</a>
    {% endif %}
  </nav>
  {% endif %}
</article>
{% endblock content %}
```

Notice `page.reading_time` - Zola calculates this automatically from word count. No plugin needed.

### Create the blog section

```bash
mkdir -p content/blog
```

Create `content/blog/_index.md`:

```markdown
+++
title = "Blog"
sort_by = "date"
paginate_by = 10
+++
```

The `sort_by = "date"` ensures posts appear newest-first. `paginate_by = 10` splits the listing into pages of 10 posts.

### Write your first post

Create `content/blog/hello-world.md`:

```markdown
+++
title = "Hello, world"
date = 2026-04-10
description = "First post on the new blog."

[taxonomies]
tags = ["meta"]
+++

This is the first post. It supports **bold**, *italic*, `inline code`,
and fenced code blocks with syntax highlighting out of the box.

And that's about it. No configuration needed to make this work.
```

### Enable taxonomies

Add this to your `config.toml`:

```toml
taxonomies = [
    { name = "tags", feed = true },
]
```

Zola automatically generates `/tags/` listing all tags, and `/tags/rust/` listing all posts tagged with "rust". You can add as many taxonomies as you want - categories, series, authors - same syntax.

### Build and serve

```bash
zola serve
```

This starts a local server at `http://127.0.0.1:1111` with live reload. Edit a markdown file, save, and the browser updates in under 50ms. No hot module replacement complexity, no WebSocket configuration - it just watches the filesystem and rebuilds.

To generate the final output:

```bash
zola build
```

Everything lands in the `public/` directory. That directory is your entire website - copy it anywhere.

## The workflow

My daily workflow is three steps:

```
1. Write markdown in content/blog/new-post.md
2. Preview with `zola serve`
3. Push to GitHub -> Actions builds and deploys
```

That's the whole pipeline. No build tool configuration, no dependency updates, no `npm audit fix`. The blog is a git repo with markdown files, a few templates, and one config file.

## Deploying to GitHub Pages

Zola has an [official GitHub Action](https://github.com/getzola/github-pages) that handles the build and deployment. Create `.github/workflows/deploy.yml`:

```yaml
name: Build and deploy
on:
  push:
    branches:
      - main

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Build Zola site
        uses: getzola/github-pages@v1
        with:
          zola_version: "v0.22.1"

  deploy:
    runs-on: ubuntu-latest
    needs: build
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

In your repository settings, go to **Settings > Pages > Build and deployment** and set the source to **GitHub Actions**. Push to `main`, and your site deploys automatically. The entire build-and-deploy cycle typically finishes in under 30 seconds, most of which is GitHub Actions overhead, not Zola's build time.

## What Zola doesn't do

Fair disclosure on the trade-offs:

**Smaller ecosystem.** Hugo has [hundreds of themes](https://themes.gohugo.io/). Zola has [around 80](https://www.getzola.org/themes/). If you want a polished theme without writing CSS, Hugo gives you more options. I write my own templates, so this doesn't affect me.

**No plugin system.** Hugo has Go modules for extensibility. Astro has integrations. Zola has what's compiled into the binary and nothing else. If you need something Zola doesn't support (like image optimization or custom output formats), you either pre-process with external tools or pick a different generator.

**Smaller community.** 17k stars vs Hugo's 87k means fewer Stack Overflow answers, fewer blog posts about solving edge cases, and fewer people who'll review your PR if you find a bug. The [Zola discourse forum](https://zola.discourse.group/) is active but small.

**No incremental builds.** Zola rebuilds the entire site every time. For a blog with a few hundred posts this is still sub-second, but if you're generating tens of thousands of pages, Hugo's incremental builds matter.

## When to pick Zola

If you're a developer who:
- Wants a blog, not a web application
- Prefers Jinja2-style templates over Go templates
- Values a single binary with no runtime dependencies
- Doesn't need a JavaScript framework
- Is comfortable writing your own CSS or using a minimal theme
- Wants syntax highlighting, feeds, taxonomies, and Sass compilation without plugins

Then Zola is worth 30 minutes of your time to evaluate. Run `zola init`, write a template, create a post, run `zola serve`. You'll know within that half hour whether it fits.

The blog you're reading right now is built with Zola. The entire source - config, templates, sass, content - fits in a repository smaller than a typical `node_modules` directory. And when I push a new markdown file, it's live in under a minute.

Sometimes the best tool is the one that gets out of your way.
{% endraw %}
