# Skills for rhun extensions

A skill for coding agents that write extensions for the [rhun](https://rhun.app) editor: the
workflow, the protocol in one page, the Rust, Go and C SDKs with recipes, and how to test with
the conformance checker and headless rhun. The same skill for Claude Code and Codex.

## Claude Code

```
/plugin marketplace add rhun-editor/skills
/plugin install rhun@rhun-editor
```

or, without the plugin system: `cp -R plugins/rhun/skills/rhun-extension ~/.claude/skills/`.
The templates ([Rust](https://github.com/rhun-editor/template-rust),
[Go](https://github.com/rhun-editor/template-go)) carry it in `.claude/skills/` already.

## Codex

```sh
codex plugin marketplace add rhun-editor/skills
codex plugin add rhun@rhun-editor
```

or `cp -R plugins/rhun/skills/rhun-extension ~/.codex/skills/`. The templates' `AGENTS.md`
gives Codex the same rules.

## Then

Ask for what you want: "write a rhun extension in Rust that charts the size of each file in my
project", "add completion from my language server to my extension". The skill starts from a
template, writes the manifest first, and tests until the conformance checker and the headless
test pass.
