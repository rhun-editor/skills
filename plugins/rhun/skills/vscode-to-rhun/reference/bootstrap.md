# Starting the rhun extension

NAME is the extension's name (`[a-z0-9][a-z0-9-]*`, equal to its GitHub repository's name;
usually the VS Code `name`), OWNER the user's GitHub account (ask; never the VS Code publisher's
unless it is the user). DIR must not exist yet, or be an empty folder.

## rhun 0.18.1 and later: `rhun new-extension`

```sh
rhun --version                       # 0.18.1 or later
rhun new-extension DIR --name NAME --owner OWNER --lang rust --skills --yes   # or --lang go
```

It downloads the template's main branch (template-rust or template-go), checks and unpacks it,
renames it (below), with `--skills` copies the rhun-extension skill into `.claude/skills/` and
`.agents/skills/`, moves it into DIR and runs `git init`. `--yes` asks nothing. It prints the
build line for this machine. What it renames: `name`, `owner` and `title` in `extension.ini` and
the `run.TARGET` program names; `go.mod`'s module (`github.com/OWNER/NAME`) or `Cargo.toml`'s and
`Cargo.lock`'s package name; `OWNER/NAME` and `NAME.command` in `tests/*`; the README's title. The
rest is still the template's demo: go on with "Then" below.

## rhun 0.18.0: clone the template

```sh
git clone --depth 1 https://github.com/rhun-editor/template-rust DIR    # or template-go
rm -rf DIR/.git && git -C DIR init -q
```

Then rename by hand, replacing `hello` (the template's name) and `rhun-editor` (its owner):

| File | Change |
| --- | --- |
| `extension.ini` | `name`, `owner`, `title`, `description`, `version`, `color`; `run.linux-x86_64 = bin/NAME-linux-x86_64`, `run.macos-arm64 = bin/NAME-macos-arm64`, `run.windows-x64 = bin/NAME-windows-x64.exe` |
| `Cargo.toml` (Rust) | `name = "NAME"`, `version`, `license` (the original's SPDX id); `Cargo.lock` follows at the next `cargo build` |
| `go.mod` (Go) | `module github.com/OWNER/NAME` |
| `tests/basic.txt` | the header comment (`DIR/OWNER/NAME`), `cmd NAME.COMMAND`, `wait-ext NAME running` |
| `tests/basic.expected` | rewritten from a first run (`UPDATE=1 sh tests/run.sh`), then read line by line |
| `tests/run.sh` | the status item's pattern `owner=rhun-editor\/hello` (or remove that block with the status item), and the sample files it writes into `proj/` (put the converted fixtures there) |
| `tests/run.sh` `run()` | add `TMPDIR=$work/tmp` (and `mkdir` it) beside `HOME` and the `XDG_` variables: the templates leave it out |
| `README.md` | the whole text (`conversion.md`) |
| `src/main.rs` / `main.go` | the top comment and the log prefixes (`hello:`) |

`tests/check.sh`, `tests/layout.sh` and the workflows read the names from `extension.ini`.

## Then (both ways)

1. **SDK 1.4.1**: the templates pin 1.4.0. Rust: `rhun-ext = { git =
   "https://github.com/rhun-editor/extension-protocol", tag = "rust-v1.4.1" }` in `Cargo.toml`;
   Go: `go get github.com/rhun-editor/extension-protocol@v1.4.1 && go mod tidy`.
2. **Remove the demo**: the `say`, `upper`, `greet` commands, the `[keys]` chord, `[menu.editor]`,
   `[status]`, `[language.Shell]` and their handler code, and the matching lines of
   `tests/basic.txt`. template-go's `scripts/` (a shell-script variant) ships in the release
   archive through `tests/layout.sh`: delete it unless you use it.
3. **Licence and icon**: replace `LICENSE` with the original's text plus the port's copyright line
   (`conversion.md`); use the original's icon when its license allows (`icon.png`, square PNG,
   at most 256 px and 64 KB).
4. **Data folders**: converted `syntax/`, `themes/`, `icon-themes/`, `icons/`, `debug/` go at the
   top of the repository; `tests/layout.sh` ships every top-level file and folder except the
   source, tests and build files, so a new source folder (`src/` is already excluded in
   template-rust; Go packages such as `internal/` are too) goes on its exclusion list, and the
   VS Code sources never belong in the repository.
5. **Agents**: keep `AGENTS.md`, `CLAUDE.md` and `.claude/skills/rhun-extension`; add a line to
   `AGENTS.md` that the extension is a port and `CONVERSION.md` lists what changed.
6. Build and check the empty shell once before porting: Rust `cargo build --release && mkdir -p
   bin && cp target/release/NAME bin/NAME-TARGET`, Go `go build -o bin/NAME-TARGET .` (TARGET
   `macos-arm64`, `linux-x86_64`), `sh tests/check.sh`.

## C or C++

There is no template. Start from template-go's layout (manifest, `tests/`, `.github/`), replace
the Go program with one source file built against `c/rhunext.h` of
rhun-editor/extension-protocol at tag v1.4.1 (`rhun-extension/reference/c.md`), and change
`tests/run.sh`, `tests/check.sh` and the release workflow's build lines to `cc`/`c++`
(`rhun-editor/wordfreq` is a C extension laid out this way, `rhun-editor/words` a C++ one).
