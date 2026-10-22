# Porting the logic

Split the original in two before translating: the **pure part** (string and data work: the
sort, the case change, the parser, the formatter) and the **glue** (everything that touches
`vscode`). Translate the pure part function by function, keeping names (in the target's case
style), messages and defaults, with its tests. Rewrite the glue around the SDK's handler; do not
emulate the `vscode` API with a shim.

## The handler shape

| VS Code | Rust (`rhun-ext`) | Go (`rhunext`) |
| --- | --- | --- |
| `activate(context)` registering commands and listeners | `struct Ext1 { … }` + `impl Handler`; `fn main() { run(&[Feature::CommandArgs, …], Ext1::default()) }` | a struct with `Ready`, `Invoke`, `Run`, `Event`; `rhunext.RunWith([]string{rhunext.FeatureCommandArgs, …}, h)` |
| `commands.registerCommand('ext.sortLines', fn)` | `fn invoke(&mut self, ext: &Ext, inv: &Invocation) { match inv.name.as_str() { "sort_lines" => …, _ => {} } }` | `func (h *H) Invoke(ext *rhunext.Ext, inv rhunext.Invocation) { switch inv.Name() { case "sort_lines": … } }` |
| `workspace.onDidChangeConfiguration` + `getConfiguration` | `fn setting(&mut self, _: &Ext, k: &str, v: &str) { self.settings.update(k, v) }`, then `self.settings.bool("filter_blank_lines", false)` | in `Event`: `if h.settings.Update(ev) { return }`, then `h.settings.Bool("filter_blank_lines", false)` |
| `window.onDidChangeTextEditorSelection` | `fn event(&mut self, _: &Ext, ev: &Event)` with `ev.name == "selection"` / `"cursor"`, fields by `ev.num(i)` | `rhunext.ParseSelection(ev)`, `rhunext.ParseCursor(ev)` |
| a module-level variable | a field of the handler struct | a field of the handler struct (`Invoke` on a pointer receiver) |

Go: implement `Run(ext, command, doc)` too (a host without `command-args` calls it): forward to
`Invoke` as template-go does. Go handlers with a value receiver cannot keep state: use a pointer
(`rhunext.RunWith(features, &h)`).

## The editor's selection

VS Code commands read `editor.selection(s)`; rhun's command carries only the document. Subscribe
and remember:

```ini
[subscribe]
opened = *
cursor = *
selection = *
```

```rust
// per document: the last selection (anchor, cursor); a cursor event without a selection resets it
#[derive(Default)]
struct Sel { sel: std::collections::HashMap<Doc, (Pos, Pos)> }   // Doc is i64
fn event(&mut self, _ext: &Ext, ev: &Event) {
    match ev.name.as_str() {
        "selection" => { let p = |i| Pos::new(ev.num(i) as u32, ev.num(i + 1) as u32);
                         self.sel.insert(ev.num(0), (p(1), p(3))); }
        "cursor" => { let at = Pos::new(ev.num(1) as u32, ev.num(2) as u32);
                      let e = self.sel.entry(ev.num(0)).or_insert((at, at));
                      if e.1 != at { *e = (at, at); } }      // the caret moved off the selection
        "closed" => { self.sel.remove(&ev.num(0)); }
        _ => {}
    }
}
```

```go
case "selection":
	if doc, r, ok := rhunext.ParseSelection(ev); ok { h.sel[doc] = r } // r.Start anchor, r.End cursor
case "cursor":
	if doc, at, ok := rhunext.ParseCursor(ev); ok && h.sel[doc].End != at { h.sel[doc] = rhunext.Range{Start: at, End: at} }
case "closed":
	if doc, ok := rhunext.ParseDoc(ev); ok { delete(h.sel, doc) }
```

Check the field types against the SDK (`Pos` fields, `Doc` type) when you write it. The anchor
may come after the cursor: order the two before using them as a range. For several carets
(`editor.selections`) agree `selections` and keep `Handler::selections` / `ParseSelections`. A
menu run (`inv.from == "menu"`) carries the selection of the right click in `inv.target()`
(`editor VERSION L1 C1 L2 C2`): prefer it there.

## Positions and text

- VS Code `Position(line, character)`: 0-based line, UTF-16 units. rhun `Pos::new(line, col)`:
  1-based line, 1-based UTF-8 byte column. `line + 1`; for the column, convert within the line:

```rust
fn utf16_to_byte(line: &str, units: usize) -> usize {      // 0-based UTF-16 offset to 0-based byte offset
    let mut u = 0;
    for (b, c) in line.char_indices() { if u >= units { return b; } u += c.len_utf16(); }
    line.len()
}
fn byte_to_utf16(line: &str, byte: usize) -> usize { line[..byte].encode_utf16().count() }
```

```go
func utf16ToByte(line string, units int) int {
	u := 0
	for b, r := range line { if u >= units { return b }; u += utf16.RuneLen(r) }
	return len(line)
}
```

- `document.getText()` is `ext.text(doc)` → bytes and the version; lines end in LF (rhun drops CR
  and puts it back on save). `lineAt(i).text` is the i-th LF-separated piece; `lineCount` is the
  number of pieces (a final LF makes an empty last line, as in VS Code).
- A whole-line range from line A to line B (1-based) is `Range::new(Pos::new(A, 1),
  Pos::new(B + 1, 1))` when line B has an LF, else up to `Pos::new(B, len(line B) + 1)`;
  `rhun_ext::whole_range(&text)` / `rhunext.WholeRange(text)` is the whole document.
- VS Code's selection that ends at column 0 of a line does not include that line (many
  line-based extensions check `selection.end.character === 0`): the same test is `end.col == 1`.
- `string.length`, `substring`, `charAt` count UTF-16 units: in Rust and Go count bytes or chars
  deliberately, and use the fixtures' Unicode lines to check that lengths match the original
  (`s.encode_utf16().count()` in Rust, `len(utf16.Encode([]rune(s)))` in Go, where the original
  measured `.length`).

## Edits

```rust
let Ok((text, version)) = ext.text(doc) else { return };
let new = transform(&text, &self.settings);                  // the pure part
if new != text {
    if let Err(e) = ext.replace(doc, version, range, &new) {  // stale: the user typed meanwhile
        eprintln!("NAME: {e}");
    }
}
```

`replace` takes `&str` in Rust (`String::from_utf8_lossy` the bytes or work on `String`), `[]byte`
in Go. Several ranges in one undo step:
`WorkspaceEdit::new().doc(doc, version, vec![TextEdit::new(r, s), …])` and `ext.apply(&e)`
(Go `e.AddDoc(doc, version, edits...)`, `ext.Apply(e)`); the ranges must not overlap and all
refer to the text at `version`. `[needs] documents = write`.

## JavaScript semantics that change

| JavaScript | Rust | Go |
| --- | --- | --- |
| `Array.prototype.sort` (stable since ES2019), comparator returning a number | `sort_by` (stable) with an `Ordering` | `slices.SortStableFunc` (`sort.Slice` is not stable) |
| `a < b` on strings, default `sort()` | UTF-16 unit order: equals `str` byte order except for characters above U+FFFF vs U+E000-U+FFFF; compare `encode_utf16()` iterators when it matters | the same caveat: `strings.Compare` is byte order |
| `localeCompare`, `Intl.Collator` (`numeric: true`, `sensitivity: 'base'`) | no ICU in std: write the comparison (lowercase, then natural digit runs), or the `icu_collator` crate; say **A** | `golang.org/x/text/collate` (`collate.New(language.Und, collate.Loose, collate.Numeric)`) |
| `toUpperCase`, `toLowerCase` | `to_uppercase`, `to_lowercase` (full Unicode, `ß` → `SS` as in JS) | `strings.ToUpper` keeps `ß`; `cases.Upper(language.Und)` from `golang.org/x/text/cases` matches JS |
| `.length`, `charCodeAt` | UTF-16: `encode_utf16().count()` | `utf16.Encode([]rune(s))` |
| `RegExp` | `regex` crate: no lookaround or backreferences (`fancy-regex` has them); `\d`, `\w` are Unicode: `(?-u:\d)` for JS's ASCII meaning | `regexp` (RE2): no lookaround or backreferences; `\d`, `\w` ASCII as in JS |
| `str.split(/\r?\n/)` | `text.split('\n')` (rhun's text has no CR) | `strings.Split(s, "\n")` |
| `Math.random` | a small PRNG of your own seeded from time, or `fastrand`/`rand` | `math/rand/v2` |
| `JSON.parse` of JSONC (comments, trailing commas) | strip comments first, then `serde_json` | strip, then `encoding/json` |
| `setTimeout`, `setInterval`, debounce | a thread with `sleep`, calling a clone of `Ext` | `time.AfterFunc`, a goroutine |
| `async`/`await`, `Promise.all` | blocking calls; threads for parallel work | blocking calls; goroutines and `sync.WaitGroup` |
| exceptions shown with `showErrorMessage` | `Result`, then `ext.toast` / `notify` with the message | `error`, then `Toast` / `Notify` |

## Node and npm

- `fs`, `path`, `os`: `std::fs`, `std::path` / `os`, `path/filepath`. Files outside the state
  folder need `[needs] project` or `home`; `HOME` is the extension's state folder.
- `child_process.spawn('tool', args)`: `std::process::Command::new("tool")` / `exec.Command`,
  with `[needs] run = tool`. Keep the same arguments; read both pipes; the tool inherits the
  sandbox. A tool the user must install is named in the README and in an error toast when missing.
- npm libraries: a crate or Go module when a small, maintained one does the same (say which in
  `CONVERSION.md`), else port the few functions used, with their license if you copy code.
  Prefer no dependency for a hundred lines of logic. Keep the template's toolchain: template-go
  says `go 1.25` (and its CI sets up 1.25), and `go get MODULE@latest` can raise that line and
  fetch a newer toolchain. Run `GOTOOLCHAIN=local go get MODULE@VERSION` with a version whose
  `go.mod` asks no more (`golang.org/x/text` v0.33.0 for Go 1.25), and check `go.mod`'s `go` line
  afterwards; in Rust keep the edition and MSRV the template builds with.
- Native Node addons (`.node`), WebAssembly blobs inside the bundle: a crate or module that does
  the job, or **N** with the reason.

## Tests

1. **Unit tests** from the original's suites (mocha `suite`/`test`, jest `describe`/`it`):
   one Rust `#[test]` (in `#[cfg(test)] mod tests`) or Go `func TestX(t *testing.T)` per case,
   on the pure functions, with the same inputs and expected outputs. Copy fixtures into
   `tests/fixtures/` (excluded from the release archive by `tests/layout.sh`) with their license.
   A test that drove `vscode.commands.executeCommand` and read the document back is a headless
   test (next).
2. **Headless tests**: rhun runs a script and its output must equal an expected file
   (`rhun-extension/reference/testing.md`). The template's `tests/run.sh` runs
   `tests/basic.txt`; extend it to run every `tests/*.txt` against `tests/NAME.expected`, and
   write the project's files (the fixtures) into `$work/proj`. Drive what a user does:

```text
open proj/unsorted.txt
wait-ext NAME running
# select lines 2 to 4: the caret to line 2, then shift+Down three times
key ctrl+Home
key Down
key shift+Down
key shift+Down
key shift+Down
cmd NAME.sort_lines
wait 300
wait 300
print-doc
cmd undo
# a setting, as the extension's page changes it
ext-setting NAME filter_blank_lines yes
# the key binding and the menu
print-keys NAME
key F9
click 500 200 right
print-menu
```

   `cmd NAME.COMMAND` runs a command as the palette does; `key …` presses its binding;
   `click X Y right` then `menu-pick LABEL` uses the editor menu (a submenu's row reads `TITLE ›`
   in `print-menu` and is picked by that label, `menu-pick Sort lines ›`, then its item by its
   title); `ext-setting NAME KEY VALUE` changes a setting as its page does (a `wait 300` before
   the next command lets the `setting` event arrive); `print-doc`, `print-toast`, `print-notes`,
   `print-status`, `print-view`, `print-problems` show the effect; `print-contrib-problems
   OWNER/NAME` prints nothing when every manifest entry was accepted. A selection made with
   `shift+Down` ends at column 1 of the next line, which VS Code's line commands count unless
   they check `end.character === 0`: keep that rule as the original had it. Write the expected
   file from a first run (`UPDATE=1`) and read every line before keeping it.
3. **The checker**: `sh tests/check.sh` (`rhun-ext-check`): every line `ok`.

Every headless run gets its own `HOME`, `XDG_CONFIG_HOME`, `XDG_STATE_HOME` and `TMPDIR` (the
template's `tests/run.sh` does it); never run rhun with a window for a test, and stop a stuck
run by its pid, never by name.
