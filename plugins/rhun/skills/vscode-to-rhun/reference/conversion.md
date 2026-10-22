# CONVERSION.md and the licensing files

## CONVERSION.md

Start it at the inventory (every row open), fill it while mapping, finish it after the tests. It
ships in the repository (and in the release archive: `tests/layout.sh` copies it), so users see
what differs from the VS Code extension. Plain sentences; one row per contribution and per
distinct API use; status **converted**, **approximated** or **not supported**, and a reason for
every row that is not converted.

```markdown
# Conversion from VS Code

NAME is a port of [DISPLAY NAME](https://marketplace.visualstudio.com/items?itemName=PUB.NAME)
VERSION by AUTHOR (SOURCE REPOSITORY at TAG or COMMIT), LICENSE. Converted to a rhun extension in
Rust (rhun-ext 1.4.1, protocol 1.1, rhun 0.18) on DATE.

## Contributions

| VS Code | rhun | Status | Notes |
| --- | --- | --- | --- |
| command `sortLines.sortLines` "Sort lines (ascending, case sensitive)" | `[commands] sort_lines` | converted | |
| key `f9` when `editorTextFocus` | `[keys] F9 = sort_lines when editorFocus` | converted | shadows rhun's own F9 binding in the editor |
| menu `editor/context` submenu `sortLines` | `[menu.editor]` + `[submenu.sort_lines]` | converted | |
| setting `sortLines.filterBlankLines` (boolean) | `[settings] filter_blank_lines` (bool) | converted | renamed: rhun keys are `[a-z0-9_]` |
| … | | | |

## API

| VS Code | rhun | Status | Notes |
| --- | --- | --- | --- |
| `window.activeTextEditor.selection` | `selection`/`cursor` events | approximated | the last selection rhun reported |
| `TextEditor.edit` | `replace` | converted | one undo step, as before |
| … | | | |

## Behaviour that differs

- Each difference a user would notice, and why.

## Not converted

- Each piece left out, why, and what to use instead.

## Tests

- Which of the original's tests were ported (count, file), which were replaced by headless
  scripts, which were dropped and why; the commands that run them.
```

Every renamed id (commands, settings, context keys, views) is in the tables with both names, so
users can move their key bindings and settings: rhun has no import of VS Code's `keybindings.json`
or `settings.json`. Settings a user had in VS Code go on the extension's page in rhun (stored in
`~/.config/rhun/extensions.ini` under `[OWNER/NAME]` as `setting.KEY = VALUE`).

## License and credit

- `LICENSE`: the original license's full text, with the original copyright lines kept and a line
  for the port added (`Copyright (c) YEAR OWNER (rhun port)`). Same license as the original for a
  copyleft one; for a permissive one the original's license is the simple choice (a different
  license for the port is the user's decision, and the original's notice still stays).
- Apache-2.0: keep the original `NOTICE` and add the port's statement of changes (point to
  `CONVERSION.md`).
- `README.md`: what the extension does in rhun, its commands, keys and settings, a "Based on"
  line (the original's name, author, link, version) and anything a user must install (a tool, a
  language server). Do not use the original's name in a way that suggests its author made or
  endorses the port: "NAME for rhun, a port of …".
- Converted data files (grammars, themes, icons, fixtures) keep their own license headers or a
  `THIRD-PARTY-NOTICES` file next to them.
- The original's source (TypeScript, the bundle, `node_modules`) never goes into the repository.
- `extension.ini` `description` may say "Port of …"; `owner` is the user's account.

## What to tell the user

The extension's folder, how to build and try it (**Link a Folder (development)** on rhun's
Extensions page with that folder, after building `bin/NAME-TARGET`; **Restart** after each
build), the test results (unit tests, the checker, the headless tests), and `CONVERSION.md`'s
approximated and not supported rows, plus any rights question still open. Releasing (a `vX.Y.Z`
tag, the index pull request) is for the user to do (`rhun-extension/reference/distribution.md`).
