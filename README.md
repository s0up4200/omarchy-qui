# omarchy-qui

Make [qui](https://github.com/autobrr/qui) follow your
[Omarchy](https://omarchy.org/) theme.

Switch Omarchy to Gruvbox and the open qui tab turns Gruvbox.

## How it works

Omarchy runs every file in `~/.config/omarchy/hooks/theme-set.d/` after a theme
change, passing the theme name. The hook looks that name up in a map and PUTs
the match to `/api/themes/settings`. qui pushes it to open tabs over its
activity stream, so nothing needs a reload.

Themes missing from the map are ignored.

## Requirements

- Omarchy
- A qui instance serving `/api/themes`
- A qui API key, from Settings, API Keys

18 of the 22 rows point at Omarchy palettes from
[qui-community-themes](https://github.com/autobrr/qui-community-themes). Put
those CSS files in your instance's themes directory.

`catppuccin`, `catppuccin-latte`, `nord` and `tokyo-night` work on any instance.

Sideloading is a qui premium feature. See the
[custom themes docs](https://getqui.com/docs/features/custom-themes).

## Install

```bash
git clone https://github.com/s0up4200/omarchy-qui ~/.local/share/omarchy-qui
mkdir -p ~/.config/omarchy/hooks/theme-set.d
ln -s ~/.local/share/omarchy-qui/qui-theme ~/.config/omarchy/hooks/theme-set.d/qui-theme

install -m 600 /dev/null ~/.config/omarchy/qui-theme.env
cat > ~/.config/omarchy/qui-theme.env <<'EOF'
QUI_URL=https://qui.example.com
QUI_API_KEY=your-key-here
EOF
```

Check the map before switching anything:

```bash
$ QUI_DRY_RUN=1 ~/.config/omarchy/hooks/theme-set.d/qui-theme tokyo-night
tokyo-night -> {"themeId":"tokyo-night","mode":"dark"}
```

## The map

`QUI_THEMES`, at the top of `qui-theme`. Bare ids are qui's own themes.
`custom:` ids are sideloaded files named after the CSS, so `rose-pine` is
`custom:rose_pine`.

To add a theme, put the CSS in your instance's themes directory
(`GET /api/themes/custom` gives the path) and add a row.

Light and dark are not in the map. `omarchy-theme-color` resolves the mode from
the theme, so one row covers both.

`QUI_VARIATIONS` sets the accent per qui theme id, and ships `blue` for
catppuccin. Without a row, qui keeps whatever accent that browser last used.

## Failures are silent

3 second timeout, always exits 0. The hook runs inside `omarchy-theme-set`, so
a slow request stalls your desktop and a non-zero exit prints `Hook failed` mid
switch.

To see the error, send it yourself:

```bash
source ~/.config/omarchy/qui-theme.env
curl -i -X PUT "$QUI_URL/api/themes/settings" \
  -H "X-API-Key: $QUI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"themeId":"tokyo-night","mode":"dark"}'
```

400 means qui does not know that id. 200 with no visible change means a
sideloaded theme on an instance without premium.

## Why the map is hardcoded

Deriving it would cover 18 of the rows: swap dashes for underscores, prefix
`custom:`. It would also fire for themes your instance lacks. qui does not
validate `custom:` ids, so `void` stores `custom:void` and a fresh browser finds
no CSS. The map is what keeps an unknown theme a no-op.
