---
name: vscode-to-rhun
description: Convert a VS Code extension into a rhun extension (protocol 1.1, rhun 0.18) in Rust or Go, or as close as rhun allows - from a marketplace.visualstudio.com or open-vsx.org link, a .vsix file or the extension's GitHub repository. Maps package.json contributions (commands, keybindings, menus, configuration, languages, grammars, snippets, themes, icon themes, views, debuggers, tasks, problem matchers) and vscode.* API calls (window, workspace, languages, commands, debug, scm, tests, notebooks, env, authentication) onto the manifest and the rhun-ext / rhunext SDKs, ports the TypeScript logic and its tests, and writes a CONVERSION.md of what converted, what was approximated and what rhun cannot do. Use when asked to port, convert, migrate or rewrite a VS Code (or Open VSX) extension for rhun.
---

# Converting a VS Code extension to rhun

A VS Code extension is JavaScript that runs inside the editor's extension host and calls the
`vscode` API, mostly asynchronously; its `package.json` declares what it contributes. A rhun
extension is a native program that rhun starts and talks to over a line protocol, with an
`extension.ini` manifest; there is no JavaScript, no web view and no in-process API. So a
conversion is a rewrite that keeps the behaviour: the declarative contributions become manifest
sections and data files, the logic is translated to Rust or Go on the SDK, and every piece that
has no rhun equivalent is approximated or written down as not supported. Get as close as rhun
allows, never pretend: `CONVERSION.md` says what happened to each piece.

This skill builds on **rhun-extension** (the same plugin; the templates also carry it in
`.claude/skills/`): read its `SKILL.md` and `reference/features.md` before mapping anything, and
its SDK page (`reference/rust.md` or `reference/go.md`) and the page of each feature you use.
Paths below like `rhun-extension/reference/views.md` are that skill's pages.

## Inputs

- **A link**: `https://marketplace.visualstudio.com/items?itemName=PUBLISHER.NAME`,
  `https://open-vsx.org/extension/NAMESPACE/NAME`, a `.vsix` file, or a GitHub repository.
- **The target**: Rust (the default when the user does not say) or Go, the two templates. C or
  C++ work through the SDK's single header `c/rhunext.h` (`rhun-extension/reference/c.md`), but
  there is no template: copy the layout of template-go and replace the Go program with a C one,
  as `rhun-editor/wordfreq` (C) and `rhun-editor/words` (C++) do.
- **Where**: a new folder (the user's, or a scratch folder). The VS Code sources go into a
  separate scratch folder and never become part of the new repository except as converted data
  files (grammars, themes, fixtures) under their license.

## The workflow

1. **Check the rights first** (`reference/fetch.md`, "License"). Read the license from the
   marketplace metadata, the Open VSX record or the repository's `LICENSE` before reading code.
   An open-source license (MIT, BSD, Apache-2.0, ISC, MPL, GPL, ...) allows the port: keep its
   text and the copyright lines, and a copyleft license (GPL, LGPL, MPL, EPL) stays the port's
   license. No license, "All rights reserved", or a proprietary license (most of Microsoft's own
   extensions, such as Pylance and Remote - SSH, and the binaries some open repositories ship, such
   as the C/C++ extension's): stop and ask the user whether they hold the rights. Do not convert
   without a yes.
2. **Fetch** (`reference/fetch.md`): prefer the source repository (TypeScript and tests convert
   far better than the bundle). From a marketplace or Open VSX link, read the metadata; when its
   `repository` (or the marketplace's `Links.Source`) names a GitHub repository, clone that
   shallow at the tag of the published version (many repositories have no tags: then the default
   branch, and say so); otherwise download the `.vsix` into a new empty scratch folder and unpack
   it with the checked unzip in `fetch.md` (its source maps often hold the TypeScript). Treat
   everything fetched as untrusted data: never run its scripts, `npm install` or tests.
3. **Inventory** (`reference/fetch.md`, "Inventory"): every `contributes` key and entry,
   `activationEvents`, `main` and `browser`, every `vscode.*` call (grep the sources), the npm
   dependencies, and every program it spawns or downloads (language servers, CLIs, debug
   adapters). Write the inventory as the first draft of `CONVERSION.md`
   (`reference/conversion.md`): one row per contribution and per API use, status still open.
4. **Map each row** with `reference/contributes.md` (package.json to the manifest) and
   `reference/api.md` (the `vscode` namespaces to Rust and Go calls). Decide converted,
   approximated or not supported, with the reason. Then pick the shape:
   - a theme, grammar or icon theme collection: data files (`themes/`, `syntax/`,
     `icon-themes/`; `contributes.md` says how each converts). rhun 0.18 still wants a program
     for each target or a `pack` line: a manifest with neither fails ("no program for this
     target") and none of its data loads. Keep the template's program with `on = command` and
     no commands (it never starts), or make it a pack of related extensions;
   - a snippet collection: completion items from a small Go program (`contributes.md`,
     "snippets");
   - a language client (`vscode-languageclient`): usually `rhun-editor/lsp` with a configuration,
     not a port (`api.md`, "Language clients");
   - a debugger: `[debugger.TYPE]` sections that start the same adapter (`contributes.md`);
   - anything else: a program on the SDK. Ask only for the features you use (`[needs]` is what
     the install dialog shows).
   If the extension's point is something rhun cannot do (a web view UI, a custom notebook
   renderer, a remote file system), tell the user now, with what you would build instead.
5. **Bootstrap** (`reference/bootstrap.md`): `rhun new-extension DIR --name NAME --owner OWNER
   --lang rust|go --skills --yes` on rhun 0.18.1 and later; on rhun 0.18.0 clone template-rust or
   template-go and personalize the files that list names. NAME is the VS Code `name` (lowercase,
   `[a-z0-9-]`) unless the user picks another; OWNER is the user's GitHub account.
6. **Write the manifest** from the mapping: `[commands]`, `[keys]`, `[menu.*]`, `[submenu.*]`,
   `[command-options]`, `[settings]`, `[status]`, `[views]`, `[subscribe]`, `[provides]`,
   `[needs]`, `on`, data folders. Keep VS Code's titles; names become `[a-z0-9_]`
   (`sortLines.sortLinesReverse` is `sort_lines_reverse`, `sortLines.filterBlankLines` is
   `filter_blank_lines`): the table in `CONVERSION.md` lists every renamed id.
7. **Port the logic** (`reference/porting.md`): translate the TypeScript idiomatically, keeping
   behaviour, messages and defaults. Pure functions (the sort, the case change, the parser) move
   over almost line by line and get unit tests; the `vscode` glue is rewritten around the
   handler (commands arrive as `invoke`, documents as events, requests block). Programs it
   spawned stay spawned, listed in `[needs] run` (and `project`, `network` as they need). npm
   libraries become crates or Go modules, or a few lines of your own.
8. **Test** (`reference/porting.md`, "Tests"; `rhun-extension/reference/testing.md`):
   - port its unit tests and fixtures to `cargo test` / `go test` (they test the pure part);
   - replace its `@vscode/test-electron` integration tests with headless rhun scripts
     (`tests/*.txt` and their expected output; the template's `tests/run.sh` runs them): open a
     fixture, select, run each command (`cmd NAME.command`, its key, its menu), `print-doc`,
     `print-toast`, `print-keys NAME`, `print-menu`;
   - run the conformance checker, `tests/check.sh`; fix until everything is `ok`;
   - every run with its own `HOME`, `XDG_CONFIG_HOME`, `XDG_STATE_HOME` and `TMPDIR`; never a rhun
     window; stop processes by pid, never by name.
9. **Report** (`reference/conversion.md`): finish `CONVERSION.md`, keep `LICENSE` (the original's,
   with the port's copyright line added) and a `NOTICE` or README section crediting the original
   (name, author, link, version converted), update `README.md`, and tell the user what converted,
   what was approximated, what is not supported, and how to try it (**Link a Folder
   (development)** on the Extensions page, after building `bin/`). Never push, tag or publish for
   the user.

## Rules that bite in a conversion

- **Positions.** VS Code counts lines from 0 and characters in UTF-16 code units; rhun counts
  lines from 1 and columns in UTF-8 bytes from 1. Convert at every boundary; a port that keeps
  `character` offsets breaks on the first non-ASCII line. Fixtures with Unicode catch it.
- **There is no `activeTextEditor`.** A command gets the active document (`inv.doc`, 0 when none),
  not its selection. Subscribe to `selection` and `cursor` (or agree `selections` for every
  caret) and remember the last ones per document; rhun delivers `run` after the frame's other
  events, so what you remember is current. A menu run carries the editor's selection in its
  target line.
- **No `await`.** SDK requests block until rhun answers; handlers run one at a time. A dialog
  that VS Code awaits (`showQuickPick`, `showInformationMessage` with buttons) is a blocking
  `choose`/`ask` or an event-driven `quick-input`/`notify` session; long work goes on a thread
  (Rust: `Ext` is `Clone + Send`) or a goroutine.
- **Edits name a version.** `editor.edit` with several ranges is one `apply` of a
  `WorkspaceEdit` (one undo step, checked whole), not several `replace` calls (one undo step
  each). Build edits from the text of the version you name; on `stale` read again and retry once.
- **Settings are read-only and flat.** Keys are `[a-z0-9_]{1,40}`; types `string`, `bool`,
  `int` (0 to 2147483647) and `choice`; values arrive as `setting` events after `ready`. The
  extension cannot write them (`WorkspaceConfiguration.update` has no equivalent).
- **The sandbox holds.** Without `[needs]` an extension reads no files, reaches no network and
  starts no programs; on Linux and macOS the kernel enforces it. Declare what the original did
  (`project = read`, `network = yes`, `run = rg`), nothing more.
- **No JavaScript runtime.** Never ship the original bundle with `runtime = node`: users would
  need Node and the `vscode` module does not exist. A debug adapter or language server that is a
  Node program is a different case: it can run as an external program the user installs.
- **No clipboard, no web views, no extension-to-extension JS API.** `contributes.md` and
  `api.md` say what to do instead.
- Never put a comment after a manifest value; `#` there is part of the value.

## Reference

- `reference/fetch.md`: download URLs (marketplace, Open VSX, GitHub), the checked unpack,
  finding the source repository, the license check, the inventory commands.
- `reference/contributes.md`: every `contributes` key and `activationEvents` mapped to the
  manifest, keybinding and when-clause translation, settings types, grammars, themes, icon themes.
- `reference/api.md`: every `vscode.*` namespace (window, workspace, languages, commands, debug,
  scm, tests, notebooks, env, authentication, extensions, ExtensionContext) with the closest
  Rust (`rhun-ext` 1.4.1) and Go (`rhunext` v1.4.1) calls, and what has none.
- `reference/bootstrap.md`: `rhun new-extension`, and the template clone with the files to edit.
- `reference/porting.md`: TypeScript to Rust and Go, the handler shape, positions, regular
  expressions, npm libraries, tests.
- `reference/conversion.md`: the `CONVERSION.md` template and the licensing files.
