# 11tyZola

A minimal static-site starter built on **Zola**, with an **Eleventy-like**
workflow: layouts, partials, components, data files and collections — but with
**no Node, no package manager, no build chain**.

- **Zola** is the skeleton: content, templates, routing, static generation.
- **Alpine.js** is the logic: interactivity, state, and calling APIs from the browser.
- **CUBE CSS** is the looks: plain SCSS organised as Composition, Utility, Block, Exception. Zola compiles it with its built-in Sass support.

Everything is driven by one small bash script, `./zac`.

## Quick start

```bash
./zac setup   # download Zola and Alpine (once)
./zac dev     # dev server at http://127.0.0.1:1111
```

Edit `templates/index.html` or any file in `sass/` and save: Zola recompiles the
SCSS and reloads the browser.

## Commands

| Command        | What it does                                              |
| -------------- | --------------------------------------------------------- |
| `./zac setup`  | Downloads pinned versions of Zola and Alpine              |
| `./zac dev`    | `zola serve`, which also compiles and watches `sass/`     |
| `./zac build`  | `zola build`, output in `_site/`                          |
| `./zac clean`  | Removes `_site/`                                         |
| `./zac help`   | Shows usage                                               |

The script uses its own filename in messages, so you can rename it
(`git mv zac myproject`) without touching anything else.

For production builds with a specific base URL:

```bash
BASE_URL=https://example.com ./zac build
```

## Requirements

- Linux, x86_64 or ARM64 (macOS works too)
- `bash`, `curl`, `tar`

`setup` downloads Linux/macOS binaries, so native Windows is not supported
out of the box. WSL works; keep the project in the Linux filesystem
(`~/...`), not under `/mnt/c`, or file watching can silently fail.

## Project layout

```
.
├── zac                    # the one script
├── config.toml            # Zola config (compile_sass = true, output_dir = "_site")
├── content/               # your pages and sections (Markdown)
├── data/                  # global data files (TOML/JSON/YAML)
├── templates/
│   ├── index.html         # home page
│   ├── page.html          # single pages
│   ├── section.html       # collections / lists
│   ├── 404.html           # not-found page
│   ├── layouts/
│   │   └── base.html      # root layout
│   ├── partials/          # reusable fragments (header, footer, card, …)
│   └── components.html    # Tera components (Zola 0.23+)
├── sass/                  # CUBE CSS (Zola compiles it)
├── static/                # copied verbatim to _site/
└── _site/                 # build output (git-ignored)
```

## 11ty → 11tyZola mapping

If you come from Eleventy, the concepts map like this:

| Eleventy | 11tyZola |
|---|---|
| `layouts/base.njk` | `templates/layouts/base.html` (`{% extends %}`) |
| `{% include "header.njk" %}` | `{% include "partials/header.html" %}` |
| Collections (`collections.posts`) | Sections (`content/blog/`) + `section.pages` |
| Pagination | `{% paginate by N %}` in a section template |
| `_data/site.json` | `data/site.toml` (`load_data`) |
| Shortcodes | Tera components in `templates/components.html` |
| Filters | Tera built-in filters / macros |
| Permalinks | `path` / `slug` in front matter |
| Passthrough copy | `static/` |
| Global config | `config.toml` (`[extra]`) |

Notes specific to Zola 0.23:
- Shortcodes were removed; use **Tera components** (`{% component name(...) %}` …
  `{% endcomponent %}`, invoked as `{{ <name ... /> }}`).
- All content is also a Tera template, so wrap stray `{{`/`{%` in `{% raw %}`.

## How the CUBE layers work here

`_global.scss` declares the cascade layer order once:

```scss
@layer global, compositions, utilities, blocks, exceptions;
```

and each partial wraps its rules in its own `@layer`. Later layers win over
earlier ones whatever the selector specificity, so an exception such as
`.status[data-state="ok"]` always beats a block, and you never need
`!important`. Zola's Sass compiler passes `@layer` through untouched.

Compositions are tuned with custom properties instead of new classes, set
inline where needed:

```html
<div class="stack" style="--stack-space: var(--space-l)">
```

## Gotchas

- **Output name:** `sass/main.scss` becomes `/main.css`. Link it with
  `{{ get_url(path='main.css') }}`.
- **No cache busting:** `cache_bust=true` does not add a hash to compiled Sass
  output, so the template links `main.css` plainly. After a deploy, visitors
  may need a hard refresh unless your host sets sensible cache headers.
- **Partials:** files starting with `_` are not compiled on their own. Add a
  new one by creating `sass/_something.scss` and adding `@use "something";` to
  `main.scss`.
- **Curly braces:** Zola's template engine (Tera) treats `{{ }}`, `{% %}` and
  `{# #}` as its own syntax. In Alpine, prefer `x-text="value"` over
  `{{ value }}`, or wrap literal braces in `{% raw %}...{% endraw %}`.
- **`base_url`:** `zola dev` overrides it locally, but `zola build` uses the
  value in `config.toml` (or `BASE_URL`). Set your real URL before deploying.
- **Reserved templates:** `index.html`, `page.html`, `section.html` and
  `404.html` must live at the root of `templates/`. `layouts/` and `partials/`
  are for shared pieces.

## Calling APIs from Alpine

The site is static, so API calls run in the visitor's browser:

```html
<div x-data="{ stars: null, failed: false }"
     x-init="fetch('https://api.github.com/repos/getzola/zola')
               .then(r => r.json())
               .then(d => stars = d.stargazers_count)
               .catch(() => failed = true)">
  <span x-show="stars !== null" x-text="stars"></span>
  <span x-show="failed">Could not load.</span>
</div>
```

- **CORS:** the API must allow cross-origin requests from your site.
- **Secrets:** anything in the page is visible to every visitor. Never put a
  private API key in a template or in Alpine code. If an API needs a secret,
  put a small proxy in front of it.
- **CSP:** `_headers` ships a Content-Security-Policy tuned for Alpine
  (`'unsafe-eval'` is required). Adjust `connect-src` if you call external APIs.

## Versions

Pinned at the top of `zac`:

```bash
ZOLA_VERSION="v0.23.6"
ALPINE_VERSION="3.17.5"
```

Change the number and run `./zac setup` again to upgrade. Alpine is downloaded
from the npm registry tarball with `curl` and `tar` only; npm itself is not
needed.

## Deploying

`_site/` is plain static files, so any static host works (GitHub Pages,
Cloudflare Pages, Netlify, an nginx folder…).

For hosts that build for you:
- **Build command:** `./zac setup && ./zac build`
- **Output directory:** `_site`

GitHub Pages and Netlify configurations are included (`.github/workflows/`,
`netlify.toml`). The build honours the `BASE_URL` environment variable for
correct asset URLs on preview deploys.

## License

MIT. Add a `LICENSE` file with your name before publishing.