# package.json to extension.ini

Each row: the VS Code key, what it becomes in rhun 0.18 (protocol 1.1), and the status to write
in `CONVERSION.md` (**C** converted, **A** approximated, **N** not supported). The pages named are
the rhun-extension skill's (`rhun-extension/reference/…`). Names: commands, settings, views and
menu items are `[a-z0-9_]{1,40}`; context keys a lowercase letter, then letters, digits, `_`
(40 bytes). Convert `ext.doThing` to `do_thing` and list every rename in `CONVERSION.md`.

## Top-level keys

| VS Code | rhun | |
| --- | --- | --- |
| `name` | `name` (`[a-z0-9][a-z0-9-]*`, equal to the repository's name) | C |
| `publisher` | `owner` is the user's GitHub account, not the publisher; credit the publisher in README/NOTICE | A |
| `displayName`, `description`, `version` | `title`, `description` (one line, 200 bytes), `version` (start at the original's or at 0.1.0; say which) | C |
| `icon` (PNG) | `icon` (square PNG, 128 px, at most 256 px and 64 KB) and `color` | C |
| `license` | keep `LICENSE` (`conversion.md`) | C |
| `main` | the program (`run.TARGET`, built from Rust or Go) | C |
| `browser` | nothing: one native program (or a WebAssembly module, `run.wasm32-wasip1`, for every system) | A |
| `engines.vscode` | `protocol = 1`; the 1.1 features it asks for in `hello` | C |
| `extensionDependencies`, `extensionPack` | `pack = OWNER/NAME …` of rhun extensions that do the same jobs (a pack only installs them) | A |
| `capabilities.untrustedWorkspaces`, `virtualWorkspaces` | nothing: `[needs]` and rhun's trust pause for folders opened from links | A |
| `l10n`, `package.nls.*.json` | resolve the default `package.nls.json` strings into the manifest; no localization | A |
| `categories`, `keywords`, `galleryBanner`, `badges`, `qna`, `sponsor`, `pricing`, `preview` | README; the index entry (`rhun-editor/extensions`) | A |

## activationEvents to `on`

`on` lists conditions, space separated; any one starts the extension. Running one of its
commands always starts it, and a `[provides]` pattern starts it when a matching file opens.

| activationEvents | rhun |
| --- | --- |
| `onCommand:ID`, none (VS Code infers them from contributions) | `on = command` |
| `onLanguage:ID` | `on = lang:NAME` (rhun's grammar name, below) |
| `workspaceContains:GLOB` | `on = contains:NAME` (a file name or `*.ext` in a workspace folder) |
| `onStartupFinished`, `*` | `on = startup` (the default) |
| `onView:ID`, `onWebviewPanel` | a declared `[views]` view starts it when shown |
| `onDebug*` | `[debugger.TYPE]` needs no program; `resolve = yes` starts it for `debug-resolve` |
| `onUri` | feature `urls`, `link` (`api.md`) |
| `onFileSystem:SCHEME` | `[provides] schemes` (read-only virtual documents) |
| `onCustomEditor`, `onNotebook` | `[provides] editor`, `notebook` |
| `onTaskType`, `onSearch`, `onAuthenticationRequest`, `onChatParticipant`, `onTerminalProfile` | `on = project` or `startup`, and say why |
| (open files of a pattern) | `on = open:*.ext` |

## contributes

| contributes | rhun | | Page |
| --- | --- | --- | --- |
| `commands` | `[commands] NAME = Title`; `category` goes in front of the title ("Sort: Reverse"), rhun appends the extension's title; `icon` (codicon or SVG) is `[command-options] NAME.icon` with a built-in icon name or `ext:NAME` from `[icons]` (PNG); `enablement` is `NAME.when` | C/A | contributions.md |
| `keybindings` | `[keys]` (below) | C/A | contributions.md |
| `menus` | `[menu.LOCATION]` and `[submenu.NAME]` (below) | C/A | contributions.md |
| `submenus` | `[submenu.NAME] title = …`; no submenu inside a submenu | C | contributions.md |
| `configuration` | `[settings]` (below) | C/A | manifest.md, resources.md |
| `configurationDefaults` | nothing: an extension cannot change rhun's or another extension's settings; document the recommended values | N | |
| `languages` | a grammar `syntax/NAME.syn` for a new language (`files`, `first_line`, comments, brackets); `[language.NAME]` keys for a language rhun has (below) | C/A | contributions.md |
| `grammars` (TextMate) | a `.syn` grammar, by hand (below); injections and embedded languages have no equivalent | A | |
| `snippets` | completion items with snippet fields (below) | A | snippets.md |
| `themes` (color) | `themes/NAME.theme` (below) | A | |
| `iconThemes` | `icon-themes/NAME.icons` with PNGs (below) | A | contributions.md |
| `productIconThemes` | none: rhun's interface icons are fixed | N | |
| `icons` (font-based) | `[icons]` PNGs for `ext:NAME` in status items, menus and buttons | A | contributions.md |
| `colors` (theme color ids) | none: use theme color names (`warning`, `accent`, `c1`-`c8`, `#rrggbb`, `DARK\|LIGHT`) in decorations and pictures | A | presentation.md |
| `views`, `viewsContainers` | `[views] KEY.title/place/kind`: places are `sidebar` (a section under the Explorer), `panel` (a bottom tab) and `side` (beside Agents); no activity-bar container of your own | A | views.md |
| `viewsWelcome` | the view's `state empty TEXT` with up to 3 action keys | A | views.md |
| `debuggers` | `[debugger.TYPE]` (below) | C/A | debug.md |
| `breakpoints` | nothing to declare: breakpoints are set in any file; a debugger's `languages` offers it for F5 | C | debug.md |
| `taskDefinitions` | none: tasks are published at run time with `tasks-set` (`api.md`, tasks) | A | tasks.md |
| `problemMatchers`, `problemPatterns` | a matcher per task: built-ins `go`, `gcc`, `rustc`, `tsc`, `msbuild`, or patterns with `{file}` `{line}` `{col}` `{message}` fields (below) | A | tasks.md |
| `customEditors` | `[provides] editor`, `editor.title`, `editor.model`, `editor.priority`: drawn as a picture by the extension, not HTML | A | custom-editors.md |
| `notebooks` | `[provides] notebook`, `notebook.title`, `notebook.kernel` | C | notebooks.md |
| `notebookRenderer` | none: outputs are text, ANSI, PNG, JPEG, GIF | N | notebooks.md |
| `semanticTokenTypes`, `semanticTokenModifiers`, `semanticTokenScopes` | `semantic-legend` with `NAME=EFFECT` (a class name, `strike`, `underline`, `faded`) | A | presentation.md |
| `jsonValidation`, `yamlValidation` | rhun-editor/lsp's JSON/YAML servers with schema settings in its `settings.json` | A | api.md |
| `typescriptServerPlugins` | none | N | |
| `terminal.profiles` | a task or `terminal-open` with a command (feature `terminals`) | A | tasks.md |
| `walkthroughs` | the README, or a `doc` view with the steps | A | views.md |
| `authentication` (a provider) | none: other extensions cannot ask yours for a session; sign-in is per extension (`api.md`) | N | |
| `resourceLabelFormatters`, `remoteHelp`, `localizations`, `continueEditSession` | none | N | |
| `chatParticipants`, `languageModelTools`, `mcpServerDefinitionProviders` | `[tools]` (other extensions and agents call them), `agents` rows | A | ai.md |
| `codeActions` (metadata), `documentation.refactoring` | nothing to declare: `[provides] actions` | C | language.md |

## Keybindings to `[keys]`

```json
{ "command": "ext.doThing", "key": "ctrl+shift+u", "mac": "cmd+shift+u", "when": "editorTextFocus", "args": "x" }
```
becomes `ctrl+shift+u = do_thing x when editorFocus`.

- **Strokes**: `ctrl`, `shift`, `alt` as they are; `cmd`, `meta`, `win` are `super`. On macOS
  rhun reports Command as `ctrl`, so a binding whose `key` is `ctrl+X` and `mac` is `cmd+X` is one
  line. A `mac` binding on the Control key (VS Code `ctrl` under `mac`) has no separate stroke;
  keep the cross-platform one and note it. Other per-system keys: prefix `linux:`, `mac:`,
  `windows:` (`mac:ctrl+alt+o = …`).
- **Key names**: `f1`-`f12` are `F1`-`F12`; `enter` `Enter`, `escape` `Escape`, `tab` `Tab`,
  `backspace` `Backspace`, `delete` `Delete`, `up/down/left/right` `Up` … `Right`, `home` `Home`,
  `end` `End`, `pageup` `PageUp`, `pagedown` `PageDown`, `insert` `Insert`, `space` `Space`,
  `numpad_add` `KP_Add` (X keysym names). A key with `=` is written by name (`ctrl+equal`).
- **Chords**: `ctrl+k ctrl+c` stays; the first stroke needs ctrl, alt or super.
- **Typing keys** (a character, Enter, Tab, Backspace, Delete, Escape, arrows, Home, End,
  PageUp, PageDown, Insert) without ctrl, alt or super are refused unless the clause has
  `!textInputFocus`, `viewFocus` or `view == …` as a top-level `&&` term. A VS Code binding of
  `tab` or `enter` in the editor (with `editorTextFocus`) cannot be kept: rebind with a modifier.
- **args**: a JSON value becomes the ARGS text after the command (up to ` when `); the handler
  reads it from `inv.body` (`from == "key"`). Pass a string as is, an object as compact JSON.
- **Removals** (`"command": "-ext.x"`) and bindings to VS Code's own commands (`editor.action.*`,
  `workbench.*`): map to `rhun.NAME` built-ins where one exists, else drop with a reason.
- Precedence: the user's `[keys]` > extensions' `[keys]` > rhun's built-ins > `[commands]` default
  keys. A `[keys]` line takes a built-in's key (F9 is Toggle Breakpoint, F12 Go to Definition):
  check `print-keys NAME` and say in the README which built-in it shadows, or pick another key.

## When clauses

rhun's clauses use the same operators (`&&`, `||`, `!`, `==`, `!=`, `<`, `in`, `not in`), but
`=~` takes a glob in quotes (`'*.test.ts'`), not a regular expression, and values are literals.
`rhun-ext-check` and `when CLAUSE` (headless) compile them.

| VS Code key | rhun key |
| --- | --- |
| `editorTextFocus`, `editorFocus` | `editorFocus` |
| `textInputFocus`, `inputFocus` | `textInputFocus` |
| `editorHasSelection`, `editorHasMultipleSelections`, `editorReadonly` | the same |
| `activeEditorIsDirty` | `editorDirty` |
| `editorLangId`, `resourceLangId` | `resourceLang`, with rhun's grammar names (below) |
| `resourceExtname`, `resourceFilename`, `resourceDirname`, `resourcePath`, `resourceScheme` | `resourceExt` (`.js`, with the dot), `resourceFilename`, `resourceDirname`, `resourcePath`, `resourceScheme` (`file` or `untitled`) |
| `isLinux`, `isMac`, `isWindows` | the same; `isWeb` is always false: drop the branch |
| `sideBarVisible`, `panelVisible` / `terminal.active` | `sidebarVisible`, `terminalVisible` |
| `terminalFocus`, `inDebugMode`, `debugState`, `debugType` | the same |
| `findWidgetVisible`, `suggestWidgetVisible`, `parameterHintsVisible`, `inSnippetMode` | `findVisible`, `completionVisible`, `signatureVisible`, `snippetActive` |
| `focusedView`, `view`, `listFocus`, `treeViewFocus` | `view` (check its value with `print-context view`), `viewFocus` |
| `viewItem`, `scmProvider`, `scmResourceGroup` | the same; `scmResourceState` is `scmResourceStatus` (a letter) |
| `explorerResourceIsFolder`, `explorerResourceIsRoot` | the same |
| `workspaceFolderCount > 0`, `workbenchState != empty` | `projectOpen` |
| `gitOpenRepositoryCount > 0` | `gitRepo` |
| `remoteName` | `remoteName` (`ssh`, `docker`, `podman`, `wsl`) |
| `config.EXT.SETTING` (its own) | `NAME.setting.KEY` (read-only) |
| `config.editor.X` | `config.SECTION.KEY` as in rhun's config (`config.editor.word_wrap`) |
| a key set with `setContext` | `NAME.KEY`, set with `set_context` (feature `context-keys`), defaults in `[context]` |
| `activeWebviewPanelId`, `notebook*`, `comment*`, `timeline*`, `isInDiffEditor`, … | none: drop the term and say what the item now shows for |

## Menus to `[menu.LOCATION]`

| VS Code menu | rhun location |
| --- | --- |
| `editor/context` | `editor` (the target line has the selection: `editor VERSION L1 C1 L2 C2`) |
| `explorer/context` | `explorer` (`file PATH`, `folder PATH`, `root PATH`) |
| `editor/title`, `editor/title/run` | `editor_title` (icon buttons, 3 then `…`) |
| `editor/title/context` | `tab` |
| `view/title` | `view_title` (icon buttons) |
| `view/item/context` | `view_item`; items whose `group` is `inline` go to `view_item_inline` |
| `scm/title`, `scm/resourceState/context`, `scm/resourceGroup/context` | `scm_title`, `scm_resource`, `scm_group` |
| `timeline/item/context` | `history_commit`, `history_file` |
| `terminal/context` | `terminal` |
| `extension/context` | `extension` (replaces the title-bar button's command list) |
| `commandPalette` with `when: false` | `[command-options] NAME.palette = no`; with a clause, `NAME.when` |
| a `submenu` entry | `ITEM.submenu = NAME` and `[submenu.NAME]` |
| `debug/*`, `testing/*`, `notebook/*`, `comments/*`, `webview/*`, `statusBar/*`, `editor/lineNumber/context`, `menuBar/*` | none |

Each entry becomes `ITEM.command`, `ITEM.when`, `ITEM.group`, `ITEM.order`, `ITEM.args`
(`ITEM` defaults to the command's name). In a menu the `editor*` and `resource*` keys describe
the menu's target, so `editorTextFocus &&` in an `editor/context` clause is dropped (the menu is
the editor's own): `editorTextFocus && !editorReadonly` becomes `!editorReadonly`.
`"group": "navigation@2"` is `group = navigation`, `order = 2`. `alt` commands have no
equivalent. Labels are the commands' titles; a submenu's row shows its `title` followed by ` ›`.
At most 16 items per extension per location.

## configuration to `[settings]`

| VS Code type | rhun `KEY.type` | |
| --- | --- | --- |
| `boolean` | `bool` (default `yes`/`no`) | C |
| `string` | `string` | C |
| `string` with `enum` | `choice` with `KEY.options = a b c` (space separated: values with spaces must be renamed, and the code maps them back) | C/A |
| `integer` 0 to 2147483647 | `int` with `min`, `max` | C |
| `number` (fractions), negative integers | `string`, parsed by the extension; say so in the description | A |
| `array`, `object` | `string` (a space- or comma-separated list), or a file in `$RHUN_STATE` or the project (`.rhun/NAME.ini`) that the extension reads | A |
| `markdownDescription`, `description` | `KEY.description` (plain text) | C |
| the property's last segment | `KEY.title` (humanized: `filterBlankLines` is "Filter Blank Lines") | C |
| `scope: resource`, `language-overridable` | `KEY.scope = workspace` or `folder`, `KEY.languages = yes` (feature `scoped-settings`) | C/A |
| `scope: machine`, `application` | `user` (the default) | C |
| `deprecationMessage`, `order`, `pattern`, `minLength`, `editPresentation` | none; check values in code | A |

Keys: `sortLines.filterBlankLines` becomes `filter_blank_lines` (the section prefix goes: the
extension's settings already belong to it). A setting that names a program or a path must stay
`scope = user`: workspace values come from project files.

## languages and language-configuration.json

VS Code language ids to rhun grammar names (`resourceLang`, `lang:`, `[language.NAME]`, `_` for
a space): `javascript` JavaScript, `typescript` and `typescriptreact` TypeScript,
`javascriptreact` JavaScript, `python` Python, `go` Go, `rust` Rust, `c` C, `cpp` C++, `csharp`
C#, `java` Java, `shellscript` Shell, `powershell` PowerShell, `markdown` Markdown, `json` and
`jsonc` JSON, `yaml` YAML, `toml` TOML, `html` HTML, `css` CSS, `xml` XML, `dockerfile`
Dockerfile, `makefile` Makefile, `ruby` Ruby, `php` PHP, `lua` Lua, `sql` SQL, `swift` Swift,
`kotlin` Kotlin, `vb` Visual_Basic. The full list is in rhun's guide (https://rhun.app/guide.md,
"Languages"); `print-state` shows a document's grammar.

A language rhun has: put the configuration in `[language.NAME]`. A new one: a grammar
`syntax/NAME.syn` with `name`, `files = *.ext Filename`, `first_line` (a word for
`firstLine`), `icon`, and the same keys.

| language-configuration.json | rhun key |
| --- | --- |
| `comments.lineComment`, `comments.blockComment` | `comment`, `block` (`toggle_comment` when the comment counts only at a line's start) |
| `brackets` | `brackets` (two-byte pairs, 16) |
| `autoClosingPairs` (with `notIn: ["string", "comment"]`) | `autoclose` (pairs of tokens up to 8 bytes, 16), `autoclose_code` for those that close only in code |
| `surroundingPairs` | `surround` (one-byte pairs) |
| `indentationRules.increaseIndentPattern`, `decreaseIndentPattern` (regular expressions) | `indent_open`, `indent_close`, `indent_middle`, `indent_dedent_after` (tokens: read the expressions and list their words) |
| `onEnterRules` | `continue` (line markers Enter repeats), `doc_block` (`/** * */`) |
| `folding.markers` | `fold_markers` (`#region #endregion`); `folding.offSide` is `fold = indent` |
| `wordPattern` | `ident` (extra word characters) |
| `colorizedBracketPairs`, `autoCloseBefore` | none |

## grammars (TextMate) to `.syn`

A `.syn` grammar is regions (strings, comments, embedded blocks) plus word lists, not regular
expressions. Convert by reading the `.tmLanguage.json` (or `.plist`/YAML):

| TextMate | `.syn` |
| --- | --- |
| `fileTypes`, the language's `extensions`/`filenames` | `files = *.ext Name` |
| `firstLineMatch` | `first_line = WORD` |
| comment rules (`comment.line`, `comment.block`) | `comment = //`, `block = /* */` |
| string rules with `begin`/`end` quotes and an escape | `string = " \`, `mstring = """ \` (multiline) |
| other `begin`/`end` rules with literal delimiters | `region = START END CLASS [multiline] [bol] [escape=X]` |
| `\b(if\|else\|…)\b` with `keyword.*` / `storage.type` / `constant.language` / `support.function` | `keywords`, `types`, `constants`, `builtins` word lists |
| sigils (`$var`, `@decorator`, `#pragma`) | `prefix = $v @a #p` |
| `(?i)` keywords | `case = insensitive` |
| capitalized type names (`entity.name.type` by case) | `captypes = yes` |

Scope to class: `keyword.*` keyword, `storage.type` and `entity.name.type` type,
`entity.name.function` function, `string.*` string, `constant.numeric` number, `comment.*` comment,
`constant.language` constant, `keyword.operator` operator, `punctuation.*` punctuation,
`meta.preprocessor` preproc, `variable.*` variable, `support.*` builtin,
`entity.other.attribute-name` attribute, `entity.name.tag` tag, `markup.heading` heading,
`markup.inserted`/`deleted` inserted/deleted, `constant.character.escape` escape,
`markup.underline.link` link. Not convertible: arbitrary `match` regexes with captures,
`include` of other grammars (embedded languages), `injectionSelector`, `while` rules, context
that depends on lookbehind. Say so as **A**; a language server's semantic tokens (through
rhun-editor/lsp) give the precise colors where one exists. Test with `open FILE` and
`print-classes LINE` headless.

## snippets

VS Code snippet files (JSON with comments: `prefix`, `body`, `description`, per language) have no
manifest equivalent. Ship them as completion items with snippet fields: `[provides] complete =
PATTERNS` and feature `snippets`. **Go** parses VS Code's snippet syntax itself
(`rhunext.PrepareSnippetTemplate`, `PrepareSnippetTemplateWithSnapshot` with variables, choices
and transforms; `CompletionItem.WithSnippetProgram`), so a snippet extension ports best to Go:
keep the JSON files as data (they ship in the release archive) and read them at start. Rust has
no template parser: build `PreparedSnippet` fields yourself or pick Go. `$CLIPBOARD` has no
source (the protocol never gives the clipboard): leave it empty and say so.

## themes to `themes/NAME.theme`

A color theme becomes an INI file (`name`, `kind = dark|light`, keys, `[syntax]`,
`[terminal]`); colors not given are derived from `bg`, `fg` and `accent`. The theme's id is the
file name without `.theme`, its title the `name` line. Test it headless through the picker:
`key ctrl+k`, `key ctrl+t`, `type` the name, `print-palette`, `key Return`, then
`print-appearance` (`shown=ID`). Keys rhun reads:
`bg fg accent accent_fg panel panel_fg border selection line_highlight line_number
line_number_active cursor error warning success info hint muted guide active popup input
scrollbar match hover link titlebar tab tab_active status git_added git_modified git_deleted
inlay inlay_bg ghost codelens highlight highlight_write`, `icon_red` … `icon_white`; `[syntax]`
`text keyword type function string number comment constant operator punctuation preproc
variable builtin attribute tag heading inserted deleted escape link`; `[terminal]` `black` …
`bright_white`. See rhun's `runtime/themes/` for complete examples.

| VS Code `colors` | rhun |
| --- | --- |
| `editor.background`, `editor.foreground` | `bg`, `fg` |
| `focusBorder` (else `button.background`, `textLink.foreground`) | `accent`; `button.foreground` is `accent_fg` |
| `sideBar.background`, `panel.background`; `sideBar.foreground` | `panel`; `panel_fg` |
| `panel.border`, `editorGroup.border`, `contrastBorder` | `border` |
| `editor.selectionBackground`, `editor.lineHighlightBackground` | `selection`, `line_highlight` |
| `editorLineNumber.foreground`, `editorLineNumber.activeForeground`, `editorCursor.foreground` | `line_number`, `line_number_active`, `cursor` |
| `editorError.foreground`, `editorWarning.foreground`, `editorInfo.foreground`, `editorHint.foreground` | `error`, `warning`, `info`, `hint` |
| `titleBar.activeBackground`, `tab.inactiveBackground`, `tab.activeBackground`, `statusBar.background` | `titlebar`, `tab`, `tab_active`, `status` |
| `editorInlayHint.foreground`/`.background`, `editorGhostText.foreground`, `editorCodeLens.foreground` | `inlay`, `inlay_bg`, `ghost`, `codelens` |
| `editor.wordHighlightBackground`, `editor.wordHighlightStrongBackground`, `editor.findMatchBackground` | `highlight`, `highlight_write`, `match` |
| `gitDecoration.added/modified/deletedResourceForeground` | `git_added`, `git_modified`, `git_deleted` |
| `terminal.ansiBlack` … `terminal.ansiBrightWhite` | `[terminal] black` … `bright_white` |
| `type` (`dark`, `light`, `hc-black`, `hc-light`) | `kind = dark` or `light` |
| `tokenColors` by scope | `[syntax]` classes, by the scope table above (the most general rule per class) |

Not convertible: `fontStyle` (italic, bold), per-scope colors finer than rhun's classes,
`semanticTokenColors` beyond their class, alpha in most interface colors (rhun takes
`#rrggbb`; blend an alpha color onto `bg` yourself), the hundreds of workbench colors without a
rhun key.

## iconThemes to `icon-themes/NAME.icons`

| VS Code icon theme JSON | `.icons` |
| --- | --- |
| `iconDefinitions.ID.iconPath` (SVG or PNG) | `[icons] ID = png/ID.png` (SVG rendered to a 32 or 64 px PNG with `rsvg-convert -w 64 -h 64`; at most 64 KB, 256 px) |
| `iconDefinitions.ID.fontCharacter` + `fontColor` (Seti-style fonts) | a PNG rendered from the glyph, or `rhun:ICON COLOR` from rhun's set; with a COLOR word the PNG's alpha is a mask in the theme's color |
| `file`, `folder`, `folderExpanded` | `file`, `folder`, `folder_open` |
| `fileExtensions` | `[extensions]` (no dot; `d.ts` works) |
| `fileNames` | `[files]` |
| `languageIds` | `[languages]` (rhun's grammar names) |
| `folderNames`, `folderNamesExpanded` | `[folders]`, `[folders_open]` |
| `light`, `highContrast`, `rootFolder*`, `hidesExplorerArrows` | none: one set (color masks follow the theme) |

Limits: 512 icons, 4096 mapping lines, 256 KB per file: a large theme (Material Icon Theme has
over a thousand) keeps the most common ones and says so.

## debuggers to `[debugger.TYPE]`

| VS Code `debuggers[]` | rhun |
| --- | --- |
| `type` | the section name `[debugger.TYPE]` (`[a-z0-9-]{1,32}`) |
| `label` | `title` |
| `languages` | `languages` (grammar names) |
| `program` + `runtime` (`node`), `args`; a `DebugAdapterExecutable` from code | `program` (searched on PATH and `paths`, or a path inside the extension's folder), `args`, `transport = stdio`; a Node adapter needs `program = node` and the user's Node |
| a `DebugAdapterServer` (port) | `transport = tcp` with `${port}` in `args`, or a launch configuration's `server = HOST:PORT` |
| `DebugAdapterInlineImplementation` | none: the adapter must be a program; the extension's own binary can be it (a second mode) |
| `configurationAttributes.launch/attach.properties` | `attr.NAME` = `string`, `path`, `int`, `bool`, `list`, `map` or `choice A B` |
| `initialConfigurations`, `configurationSnippets` | `templates = debug/NAME.ini` (launch.ini fragments) |
| `variables` (command variables) | `${config:KEY}`, rhun's `${file}`, `${ask:…}`, `${pick:A\|B}`; others through `resolve = yes` |
| a `DebugConfigurationProvider` | `resolve = yes` and the `debug-resolve` handler (`api.md`) |
| `installation`/adapter download | `install = COMMAND` shown when `program` is missing; never download it yourself without `network = yes` and the user's say |

Users' `launch.json` becomes `.rhun/launch.ini` (a section per configuration); offer the
conversion in the README. Add `[needs] debug = yes`.

## problemMatchers and tasks

A VS Code matcher is a regular expression with numbered groups; a rhun pattern is a line with
fields: `{file}:{line}:{col}: {severity}: {message}` (a run of spaces matches blanks, `{_}` any
text; `columns bytes` when the tool counts bytes). Multi-line `pattern` arrays become steps
(`1 …`, `2 …`, `loop`). `$tsc`, `$gcc`, `$go`, `$rustc`, `$msCompile` are the built-ins `tsc`,
`gcc`, `go`, `rustc`, `msbuild`; `$eslint-stylish` and others need a pattern. `background.
beginsPattern`/`endsPattern` are a task's `busy` and `ready` patterns. Check each pattern with
`check_pattern` / `CheckPattern` in a unit test against the tool's real output.
