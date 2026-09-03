# Perxel WP Plugin UI

A small, server-rendered admin-UI layer that every Perxel WordPress plugin
bundles. It provides one master layout (feature sidebar + main content) and a
handful of components built **on top of** native wp-admin CSS - not a
replacement for it.

- **Server-rendered PHP + a few lines of vanilla JS. No build step, no framework.**
- Distributed as a tagged tarball, vendored into each plugin at
  `vendor/perxel-ui/` - the same model as
  [Action Scheduler](https://actionscheduler.org/): committed to the plugin
  repo, shipped in the plugin zip, refreshed with a script.
- The runtime loader keeps the **highest version wins** across every active
  plugin that ships a copy, so a stale copy in one plugin can never fatal or
  restyle another.

Current version: **0.18.0** - see [`CHANGELOG.md`](CHANGELOG.md).

---

## Installing it in a plugin

### 1. Vendor the kit

Drop `bin/update-ui.sh` into the consuming plugin:

```sh
#!/usr/bin/env bash
# Refresh vendor/perxel-ui/ from a tagged release of perxel/wp-plugin-ui.
set -euo pipefail

VERSION="${1:?usage: bin/update-ui.sh <version>   e.g. 0.16.0}"
DEST="$(cd "$(dirname "$0")/.." && pwd)/vendor/perxel-ui"
URL="https://github.com/perxel/wp-plugin-ui/archive/refs/tags/v${VERSION}.tar.gz"

tmp="$(mktemp -d)"
trap 'rm -rf "$tmp"' EXIT

curl -fsSL "$URL" | tar -xz -C "$tmp" --strip-components=1

rm -rf "$DEST"
mkdir -p "$DEST"
cp -R "$tmp"/. "$DEST"/

# Not needed at runtime.
rm -f "$DEST"/README.md "$DEST"/CHANGELOG.md "$DEST"/CHECKLIST-wordpress-org.md \
      "$DEST"/.gitignore "$DEST"/AGENTS.md "$DEST"/CLAUDE.md

echo "vendor/perxel-ui/ is now at v${VERSION}"
```

```sh
bin/update-ui.sh 0.18.0
```

Commit `vendor/perxel-ui/` (add a `.gitignore` exception if `vendor/` is
ignored) so it ships in the plugin zip and the WordPress.org SVN checkout -
there is no Composer step on the build server.

### 2. Register the loader

In the plugin's main file, after its own constants:

```php
require_once __DIR__ . '/vendor/perxel-ui/loader.php';
Perxel_UI_Loader::register(
    '0.18.0',
    __DIR__ . '/vendor/perxel-ui',
    plugins_url( 'vendor/perxel-ui', __FILE__ )
);
```

The version string passed here is what the "highest wins" loader compares -
keep it equal to the tag you vendored (currently 0.18.0).

### 3. Use it in an admin-page callback

```php
Perxel_UI::enqueue(); // on admin_enqueue_scripts for the page

Perxel_UI_Layout::open( array(
    'title'   => __( 'Dashboard', 'my-plugin' ),
    'plugin'  => 'My Plugin',
    'version' => MY_PLUGIN_VERSION,
    'menu'    => array( '' => array(
        'my-plugin'          => __( 'Dashboard', 'my-plugin' ),
        'my-plugin-settings' => __( 'Settings', 'my-plugin' ),
    ) ),
    'current' => 'my-plugin',
    'base'    => 'admin.php',
) );

include __DIR__ . '/views/dashboard.php'; // plugin-owned main content

Perxel_UI_Layout::close();
```

---

## Versioning & the "overwrite is safe" guarantee

- The kit is versioned independently of any plugin (see `CHANGELOG.md`). Bump it
  when the kit changes, not when a consumer does.
- The loader keeps the **highest** registered copy when several plugins are
  active; the others are inert (`class_exists` guard). Two versions never
  collide.
- **Within a major version the public API is additive-only.** A newer vendored
  copy dropped into an older plugin, or vice versa, can never fatal and never
  changes plugin behaviour - at worst a plugin that needs a newer kit shows an
  admin notice (`Perxel_UI_Loader::require_version()`).
- A breaking change = major bump, and every plugin must re-vendor before
  shipping it.
- `loader.php` itself must stay backwards compatible **forever** - it is the one
  file an old plugin still runs when a newer copy wins.

---

## Public API

`Perxel_UI_Layout`

| Method | Purpose |
| --- | --- |
| `open( array $args )` | `.wrap` → shell → sidebar (sticky brand bar: `plugin`) → `<main>` (sticky title bar: `<h1>` + `actions`). Args: `title`, `plugin`, `version`, `menu`, `current`, `base`, `links`, `author`, `actions`, `wrap_class`, `text_domain`. `actions` is trusted HTML pinned to the right of the title bar - the house home for a page's Save button; wire it to the page's `<form>` with the HTML5 `form="<form-id>"` attribute. `author` (`[ 'name' => …, 'url' => … ]`) and `version` show left in the footer; `links` (`[ label => url ]`) show right. |
| `close()` | Renders the footer, then closes what `open()` opened. |
| `set_page_titles( array $map, $plugin = '' )` | `[ page_slug => page name ]` + a plugin name. Own the browser `<title>` for the kit's screens: the tab reads `Site • Page • Plugin` instead of the bare " ‹ Site - WordPress" a `remove_submenu_page()`d screen is left with. Call on `admin_menu`. Additive, idempotent. |

`Perxel_UI` (each returns an HTML string - `echo` it)

| Method | Purpose |
| --- | --- |
| `enqueue()` | Registers the kit CSS/JS under the shared `perxel-ui` handle. |
| `notice( $type, $html, $args )` | `success\|warning\|error\|info`, on WP `.notice`. `$args`: `dismissible`, `inline`. |
| `progress_bar( $pct, $args )` | Standalone bar. `$args`: `id`, `label`. |
| `card( $args )` | `title`, `body`, `actions`, `id`, `class`. |
| `rows( $groups )` | iOS-style grouped settings list. Flat row list, or groups `[ 'title' => …, 'rows' => [ … ] ]`. Row: `label` left, `content` right, plus `sub`, `tone`, `icon` (`good`/`warn`/`bad`/`muted` status dot, or trusted-HTML glyph). A row with a `summary` key is a native `<details>` disclosure. A group takes `title_action` (trusted HTML pinned right of the title), `'danger' => true` (destructive zone), and `note` (muted footnote). |
| `toggle( $args )` | A checkbox rendered as an iOS switch. `name`, `checked`, `value`, `id`, `form`, `label`. |
| `checkbox_group( $args )` | A "pick several" list rendered as selectable pills. `options`, `name`, `form`, `selected`. |
| `code( $text, $args )` | Read-only preformatted block - scrolls sideways. `$args`: `label`, `id`. |
| `spinner()` | Inline CSS loading spinner. |

### Escaping contract

The helpers escape their own structural markup and the `title` / `label` fields.
`body`, `actions`, `value`, `content`, `sub` are treated as **trusted HTML** -
the caller escapes their dynamic parts:

```php
echo Perxel_UI::rows( $groups ); // phpcs:ignore WordPress.Security.EscapeOutput.OutputNotEscaped -- Perxel_UI escapes internally.
```

---

## UI rules

- **No em dashes** (or en dashes) anywhere - strings, comments, docs. Plain
  hyphen, spaced when it joins clauses.
- **A row with an input does not also carry an action button.** Space in the
  value slot is tight. Either auto-run the action a short debounce after the
  user stops typing, or lift the action to the group's `title_action`.
- Configuration lives on its own screen, away from the action it configures, so
  a client can be told "go here, click this, done". One primary action per
  screen (pinned in the sticky title bar via `open()`'s `actions`).

## What belongs in the kit

- **In the kit:** anything another Perxel plugin could plausibly reuse - layout,
  notices, progress bars, cards, row groups, tokens.
- **In the plugin:** anything specific to that plugin's domain (its own widgets,
  domain-specific tables, dialogs). Plugin CSS/JS may be inline or in the
  plugin's own `assets/`.
- **Grey area:** start plugin-local; promote to the kit when a second plugin
  needs it, and bump the kit version.

## Showcase

`showcase/` renders every component in the real layout - the review surface
after any kit change. A plugin hosts it as one of its own hidden screens: define
`PERXEL_UI_SHOWCASE_HOSTED` (truthy, before the kit boots), then echo
`Perxel_UI_Showcase::body()` between your own `Perxel_UI_Layout::open()` /
`close()`. Strip `showcase/` from the distributed build (`.distignore`); the
loader tolerates its absence.

## Consumers

| Plugin | Vendored version |
| --- | --- |
| [wp-ai-translate](https://github.com/perxel/wp-ai-translate) | 0.18.0 (first consumer) |
| [wp-image-optimizer](https://github.com/perxel/wp-image-optimizer) | 0.15.0 (copied `ui/`; migrates later) |

## License

GPL-2.0-or-later.
