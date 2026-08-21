# qui-omarchy

Make [qui](https://github.com/autobrr/qui) change theme when your
[Omarchy](https://omarchy.org/) theme changes.

Switch Omarchy to Gruvbox, and the qui tab on your second monitor turns Gruvbox
too. It is one bash script and a text file with your API key.

## How it works

Omarchy runs every file in `~/.config/omarchy/hooks/theme-set.d/` after a theme
change. It gives each file the theme name as the first argument.

This hook reads a map of Omarchy theme names to qui theme ids. It then sends
the match to your instance with `PUT /api/themes/settings`. qui pushes the new
selection to open tabs over its activity stream, so they change theme without a
reload.

An Omarchy theme that is not in the map is ignored. qui keeps the theme it has.

## Before you start

You need three things:

- Omarchy.
- A qui instance with the theme settings API. Run `curl https://your-qui/api/themes` and look for a list of themes.
- A qui API key, from Settings, then API Keys.

18 of the 22 rows in the map point at Omarchy palettes from
[qui-community-themes](https://github.com/autobrr/qui-community-themes). Put
those CSS files in the themes directory of your instance. Sideloaded themes need
an active premium license, because `GET /api/themes/custom` returns 403 without
one, and the browser then has no CSS to apply.

Four rows work without a license: `catppuccin`, `catppuccin-latte`, `nord`, and
`tokyo-night`. They use themes that qui already has.

## Install

1. Clone this repository:

```bash
git clone https://github.com/s0up4200/qui-omarchy ~/.local/share/qui-omarchy
```

2. Link the hook into Omarchy:

```bash
mkdir -p ~/.config/omarchy/hooks/theme-set.d
ln -s ~/.local/share/qui-omarchy/qui-theme ~/.config/omarchy/hooks/theme-set.d/qui-theme
```

3. Write your instance URL and API key to `~/.config/omarchy/qui-theme.env`:

```bash
install -m 600 /dev/null ~/.config/omarchy/qui-theme.env
cat > ~/.config/omarchy/qui-theme.env <<'EOF'
QUI_URL=https://qui.example.com
QUI_API_KEY=your-key-here
EOF
```

The file is mode 600 and stays outside the repository. The hook does nothing
when the file is absent, so you can install the hook first and add the key
later.

4. Make sure that the map is correct:

```bash
QUI_DRY_RUN=1 ~/.config/omarchy/hooks/theme-set.d/qui-theme tokyo-night
```

The output is the payload, and nothing is sent:

```
tokyo-night -> {"themeId":"tokyo-night","mode":"dark"}
```

5. Change your Omarchy theme. The qui tab follows at once.

## Which themes map

Every row that is not one of those four points at a sideloaded Omarchy palette. The id of a sideloaded
theme is its file name, so dashes become underscores. `rose-pine` becomes
`custom:rose_pine`.

Light and dark are not in the map. The hook asks `omarchy-theme-color` for the
mode of the theme, so one row covers both.

Edit `QUI_THEMES` at the top of `qui-theme` to add your own. To add a theme:

1. Put the CSS file in the themes directory of your instance. `GET /api/themes/custom` shows the directory path.
2. Add a row. The key is the Omarchy theme name, and the value is `custom:` and the file name without `.css`.

Accent colors work the same way. `QUI_VARIATIONS` maps a qui theme id to an
accent name, and the default map sends `blue` for catppuccin. A theme with no
row keeps the accent that the browser last used.

## Failures are silent

The request has a 3 second limit, and the hook always exits 0.

The hook runs inside `omarchy-theme-set`. A slow request there stalls your
desktop, and a non-zero exit prints `Hook failed` into the theme switch. An
unreachable server, a bad key, and a missing key file all end the same way.
Omarchy changes theme, and qui does not.

To find out what happened, run the request yourself:

```bash
source ~/.config/omarchy/qui-theme.env
curl -i -X PUT "$QUI_URL/api/themes/settings" \
  -H "X-API-Key: $QUI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"themeId":"tokyo-night","mode":"dark"}'
```

A 400 means qui does not know that theme id. If the request returns 200 and the
tab does not change, the theme is a sideloaded one and the instance has no
premium license.

## Why the map is written down

One line of bash replaces 18 of the rows: swap dashes for underscores, and add
the `custom:` prefix. It also fires for themes your instance does not have. qui
does not validate `custom:` ids, so an Omarchy theme named `void` stores
`custom:void`, and a browser that opens fresh finds nothing to apply. The map is
what keeps an unknown theme a no-op.
