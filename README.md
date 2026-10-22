# Skills for rhun extensions

Two skills for coding agents that write extensions for the [rhun](https://rhun.app) editor, the
same for Claude Code and Codex:

- **rhun-extension**: the workflow, the protocol in one page, the Rust, Go and C SDKs (v1.4.0)
  with recipes, a page per area of protocol 1.1 (rhun 0.18), and how to test with the
  conformance checker and headless rhun.
- **vscode-to-rhun**: converts a VS Code extension, given its Marketplace, Open VSX or GitHub
  link, into a rhun extension in Rust or Go, or as close as rhun allows. It checks the license,
  fetches the source (or unpacks the `.vsix`), maps every contribution and `vscode` API call onto
  the manifest and the SDKs, ports the logic and its tests, and writes a `CONVERSION.md` of what
  converted, what was approximated and what rhun cannot do.

rhun-extension covers everything an extension can do: commands with arguments, keys and chords,
menus, status items and when clauses; workspace edits and snippets; trees, logs, forms, tables,
Markdown and picture views, notifications, progress and quick input; every language feature and
the presentation ones (semantic tokens, inlay hints, code lens, decorations); tasks, debuggers
and test controllers; source control panels, notebooks, custom editors, agents and tools;
secrets, sign-in, virtual documents; remote windows, WebAssembly modules, packs and releases.

## Claude Code

```
/plugin marketplace add rhun-editor/skills
/plugin install rhun@rhun-editor
```

or, without the plugin system: `cp -R plugins/rhun/skills/* ~/.claude/skills/`.
The templates ([Rust](https://github.com/rhun-editor/template-rust),
[Go](https://github.com/rhun-editor/template-go)) carry rhun-extension in `.claude/skills/` already.

## Codex

```sh
codex plugin marketplace add rhun-editor/skills
codex plugin add rhun@rhun-editor
```

or `cp -R plugins/rhun/skills/* ~/.codex/skills/`. The templates' `AGENTS.md`
gives Codex the same rules.

## Then

Ask for what you want: "write a rhun extension in Rust that charts the size of each file in my
project", "add a TODO tree to the sidebar", "run my project's tests from the Testing section",
"add completion from my language server to my extension". The skill picks the features, starts
from a template, declares them in `hello` and the manifest, and tests until the conformance
checker and the headless test pass.

To port a VS Code extension: "convert https://marketplace.visualstudio.com/items?itemName=PUB.NAME
to a rhun extension in Go". The agent asks before converting an extension without an open-source
license, starts from a template (`rhun new-extension` on rhun 0.18.1 and later), and leaves the
result in a folder of its own with `CONVERSION.md`; it never publishes anything.
