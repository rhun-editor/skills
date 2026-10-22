# The vscode API in rhun-ext and rhunext

Rust is `rhun-ext` 1.4.1 (`ext: &Ext` in a `Handler` method), Go is `rhunext` v1.4.1 (`ext
*rhunext.Ext`); both speak protocol 1.1. Every name below exists in those versions. A call of a
1.1 feature needs the feature in the run call (Rust `run(&[Feature::X, …], h)`, Go
`rhunext.RunWith([]string{rhunext.FeatureX, …}, h)`) and fails locally with `unknown` when rhun
did not agree it: check `ext.has(Feature::X)` / `ext.Has(rhunext.FeatureX)` and fall back. The
"Page" column names the rhun-extension skill's reference page with the details. Status: **C**
converts, **A** approximates, **N** has no equivalent (what to do instead is in the row).

VS Code calls return promises; SDK calls block until rhun answers and return a `Result` (Rust) or
`(…, error)` (Go). Handlers run one at a time, so a call that waits for the user (a pick list,
a prompt, the approval of a terminal) holds every other event until it returns: keep those to
commands, and put background work on a thread (Rust, `ext.clone()`) or a goroutine.

## Extension lifecycle and ExtensionContext

| VS Code | Rust | Go | |
| --- | --- | --- | --- |
| `activate(context)` | `Handler::ready(ext, project)` | `Ready(ext, project)` | C |
| `deactivate()` | `Handler::bye(ext)` (leave within 1 s; stop the programs you started) | `RunWith` returns on `bye`: clean up after it in `main` | C |
| `context.subscriptions`, `Disposable` | nothing: everything goes when the process stops | | C |
| `context.globalState` (Memento) | a file in `$RHUN_STATE` (the state folder, writable, also `HOME`) | same | A |
| `context.workspaceState` | a file in `$RHUN_STATE` keyed by the project path (`ready`'s `project`) | same | A |
| `context.secrets` | `ext.secret_get`, `secret_set`, `secret_delete`, `secret_keys` (feature `Secrets`, `[needs] secrets = yes`) | `SecretGetContext`, `SecretSetContext`, `SecretDeleteContext`, `SecretKeysContext` | C |
| `context.extensionPath`, `asAbsolutePath`, `extensionUri` | the working directory (rhun starts the program in its folder): `std::env::current_dir()` | `os.Getwd()` | C |
| `globalStorageUri`, `storageUri`, `logUri` | `$RHUN_STATE`; logs go to standard error (rhun keeps 1 MB) | `os.Getenv("RHUN_STATE")`, `ext.Logf` | C |
| `context.extensionMode`, `environmentVariableCollection` | none | | N |
| `extensions.getExtension(id).exports` | none; another extension's `callable = yes` command: `ext.run_command("OWNER/EXT.NAME", args)`, or its `[tools]`: `ext.tool_call` | `RunCommand`, `CallTool` | A |
| `vscode.l10n.t` | the English string | | A |

## commands

| VS Code | Rust | Go | |
| --- | --- | --- | --- |
| `registerCommand(id, fn)` | `[commands] NAME = Title` + `Handler::invoke(ext, &Invocation)` (`inv.name`, `inv.doc`, `inv.from`, `inv.body`, `inv.target()`; feature `CommandArgs`) | `Invoke(ext, inv)` (`inv.Name()`, `inv.Doc`, `inv.From`, `inv.Body`, `inv.Target()`) | C |
| `registerTextEditorCommand` | the same; `inv.doc` is the active document (0: none); the selection from your tracked `selection` events (`porting.md`) | the same | C |
| command arguments (`executeCommand(id, arg)`, keybinding `args`, menu args) | `inv.body` (bytes: parse JSON yourself), `from` `key`/`menu`/`ref` | `inv.Body` | C |
| register at run time | `ext.register(name, keys, title)`, `ext.unregister(name)` (feature `Commands`) | `Register`, `Unregister` | C |
| `executeCommand('setContext', key, value)` | `ext.set_context(key, value)`, `remove_context` (feature `ContextKeys`) | `SetContext`, `RemoveContext` | C |
| `executeCommand` of a VS Code command | rhun's own: `ext.cmd(name)` (v1 section 8a: `undo`, `toggle_comment`, `find`, `goto_line`, ...) or `ext.run_command("goto_line", b"12 1")` | `Cmd`, `RunCommand` | A |
| `executeCommand('vscode.open', uri)`, `'revealInExplorer'` | `ext.open(path, Pos::new(l, c))`; `ext.cmd("reveal_file")` | `Open`, `Cmd` | C |
| `executeCommand('vscode.diff', a, b, title)` | `ext.diff(left, right, title, Show::Front)` (feature `Diff`; documents or virtual URIs) | `Diff(l, r, title, rhunext.DiffShow)` | C |
| `executeCommand('vscode.execute*Provider')`, `getCommands` | none: an extension cannot ask other providers or list commands | | N |
| a command another extension calls | `[command-options] NAME.callable = yes` + `Handler::call(ext, &CallRequest)` | `Callee` (`Call`) | C |

## window

| VS Code | Rust | Go | |
| --- | --- | --- | --- |
| `showInformationMessage(msg)` (no items) | `ext.toast(msg)` (2 s) | `Toast` | C |
| `showInformation/Warning/ErrorMessage(msg, ...items)` | `ext.notify(notify::Severity::Warning, &Note::new(msg).button("Retry"))` → NOTE; the click arrives in `Handler::note_event(ext, note, NoteEvent::Pressed(n))`; `.button_command(title, cmd, args)` runs a command instead (feature `Notify`) | `Notify(rhunext.SeverityWarning, rhunext.NewNote(msg).Button("Retry"))`, `NoteListener` | C (the promise becomes an event) |
| `{ modal: true }` | `ext.choose(prompt, &["Yes", "No"])` (blocks; `None`: cancelled) | `Choose` (0-based; `Cancelled(err)`) | A |
| `showQuickPick(items)` | `ext.choose(prompt, &items)` → `Option<usize>` | `Choose(prompt, items)` | C |
| `showQuickPick` with `canPickMany`, `detail`, `description`, buttons, `matchOnDescription` | `ext.quick(&quick::Step::pick(title).flags(&[Flag::Multi]).items(&[quick::Item::new(id, label).desc(d)]))`, `Handler::quick_event` (feature `QuickInput`) | `Quick(rhunext.QuickPick(title).Flags("multi").Items(rhunext.NewQuickItem(id, label).Desc(d)))`, `QuickListener` | C |
| `showInputBox({ prompt, value })` | `ext.ask(prompt)` → `Option<String>` | `Ask` | C |
| `showInputBox` with `validateInput`, `password`, `placeHolder` | `quick::Step::input(title).placeholder(p).flags(&[Flag::Password])`; validate on `QuickEvent::Value` with `quick_set(q, &Step::new().msg(seq, "error", text))` | `QuickInput(title)`, `QuickSet`, `.Msg(seq, "error", text)` | C |
| `createQuickPick`, `createInputBox` (multi-step) | the same session: `quick_set` per step, `Flag::Back` | `QuickSet`, `Flags("back")` | C |
| `withProgress({ location }, task)` | `ext.progress(ProgressWhere::Status \| Note \| View(v), cancellable, title)` → P, `progress_set(p, Some(40), msg)`, `progress_end(p)`; `Handler::progress_cancel` (feature `Progress`) | `Progress(rhunext.ProgressStatus, cancel, title)`, `ProgressSet`, `ProgressEnd`, `ProgressCanceller` | C |
| `setStatusBarMessage(text)` | `ext.status(text)` | `Status` | C |
| `createStatusBarItem` (`text` with `$(icon)`, `tooltip`, `command`, `color`, `backgroundColor`) | `[status]` items, or `ext.set_status_item(&StatusItem{ name, align, priority, text, tooltip, command, args, color, background, when })`, `remove_status_item` (feature `StatusItems`); 8 items, 120 bytes; icons from rhun's list or `ext:NAME` | `SetStatusItem(StatusItem{…})`, `RemoveStatusItem` | C/A |
| `createOutputChannel(name)`, `LogOutputChannel` | a `[views]` view `kind = log`, `place = panel`; `ext.patch(view, &Patch::new().log(&[(LogLevel::Info, line)]))` (features `ViewTree`, `ViewLog`); or standard error for debug logs | `PatchView(v, rhunext.NewPatch().Log(rhunext.LogLine{…}))` | C |
| `activeTextEditor` | none: the command's `inv.doc`, the `active DOC` event, tracked `cursor`/`selection` events | `inv.Doc`, `ParseCursor`, `ParseSelection` | A |
| `activeTextEditor.selection(s)` | `[subscribe] selection = *` and `cursor = *`, remember per document; all carets: feature `Selections`, `Handler::selections(ext, &Selections)` | `ParseSelection`, `ParseSelections` | A |
| `editor.selection = …`, `editor.selections = …` | `ext.select(doc, range)`; `ext.select_many(doc, &[Selection::new(anchor, cursor)])` | `Select`, `SelectMany` | C |
| `editor.edit(b => b.replace/insert/delete)` | one range: `ext.replace(doc, version, range, text)`; several: `ext.apply(&WorkspaceEdit::new().doc(doc, version, edits))` (one undo step; `[needs] documents = write`; feature `WorkspaceEdits` for one transaction) | `Replace`; `var e rhunext.WorkspaceEdit; e.AddDoc(doc, v, edits...); ext.Apply(e)` | C |
| `editor.insertSnippet(SnippetString)` | `ext.insert_snippet(doc, v, span, &PreparedSnippet{…})` (feature `Snippets`) | `p, err := rhunext.PrepareSnippetTemplate(src, rhunext.SnippetContext{})`, then `InsertSnippetProgram(doc, v, span, p)` | C |
| `editor.revealRange` | `ext.reveal(doc, pos, Reveal::Center)` (feature `Reveal`) | `Reveal(doc, pos, rhunext.RevealCenter)` | C |
| `editor.options` (tabSize, insertSpaces) | `ext.option(doc, "tab_width", "4")` (sets; nothing reads them back except a format request's `tab_width`) | `Option` | A |
| `createTextEditorDecorationType`, `setDecorations` | `ext.decoration_type(name, "background warning/15\n")`, `ext.decorate(doc, version, name, 0, 0, &[DecorationRange{…}])` (feature `Decorations`, `documents = read`) | `DecorationType`, `Decorate` | C/A |
| `visibleRanges`, `onDidChangeTextEditorVisibleRanges` | `parse_visible(ev)` (with `Decorations`) | `ParseVisible` | C |
| `showTextDocument`, `openTextDocument(path)` | `ext.open(path, Pos::new(line, col))` → Doc | `Open` | C |
| `openTextDocument({ content })` (untitled) | a virtual document: `[provides] schemes`, `Handler::content`, `ext.open_uri(uri, pos)` (feature `Uris`; read-only) | `ContentProvider`, `OpenURIContext` | A |
| `createTreeView`, `registerTreeDataProvider` | `[views] KEY.kind = tree`; fill on `Handler::view_open`, `ext.patch(view, &Patch::new().kids(…))`, events in `view_event`, lazy children in `view_children` (feature `ViewTree`) | `RetainedViewer`, `PatchView`, `ChildrenProvider` | C |
| `TreeItem.command`, `contextValue`, `iconPath`, `description`, `tooltip` | `Node::new(id, label).command(name, args).context(v).icon(Icon::named("file"), …).desc(d).tip(t)` | `NewNode(id, label).Command(…).Context(…).Icon(…).Desc(…).Tip(…)` | C/A (icons from a fixed list) |
| `createWebviewPanel`, `registerWebviewViewProvider` | none. Instead: a `doc` view (Markdown, `Patch::doc`, feature `ViewDoc`), a `form` or `table` view, a picture view drawn with `Picture` (charts: `rhun_ext::chart`), or `ext.open_url` to a page | `Doc`, `Form`, `Columns` patches; `Picture` | N |
| `registerCustomEditorProvider` | feature `CustomEditors`, a picture-drawn tab: `Handler::editor_open`, `editor_save` | `CustomEditor` | A |
| `createTerminal({ name, shellPath, shellArgs, cwd, env })`, `terminal.sendText` | `ext.open_terminal(&TerminalSpec{ label, program: Program::argv(&[…]), … })`, `ext.terminal_input(term, bytes)` (features `Terminals`; `[needs] terminal = run`/`input`) | `OpenTerminal(TerminalSpec{…})`, `TerminalInput` | C |
| a suggested command line | `ext.term(cmd)` (`terminal = suggest`; the user presses Enter) | `Term` | C |
| `onDidChangeActiveTextEditor` | `Event` `active DOC` | `ev.Name == "active"` | C |
| `onDidChangeTextEditorSelection` | `selection` / `cursor` events, or `Handler::selections` | `ParseSelection`, `ParseCursor`, `ParseSelections` | C |
| `registerUriHandler` | `ext.link()` → a `rhun://ext/OWNER/NAME/TOKEN` link, `parse_link(ev)` (feature `Urls`) | `LinkContext`, `ParseLink` | A |
| `registerFileDecorationProvider`, `tabGroups`, `activeColorTheme`, `onDidChangeWindowState`, `showOpenDialog`, `showSaveDialog`, `showWorkspaceFolderPick` | none; for a file: `ask` a path or a `quick` pick filled from `find_files` | | N |

## workspace

| VS Code | Rust | Go | |
| --- | --- | --- | --- |
| `getConfiguration(section).get(key)` | `[settings]` + `Handler::setting(ext, key, value)` (every value after `ready`, and on change); `Settings::update`, `.string`, `.bool`, `.int` | `rhunext.Settings`: `Update(ev)`, `String`, `Bool`, `Int` in `Event` | C |
| per folder or language (`get` with a scope) | `ext.setting_get(key, folder, lang)` (feature `ScopedSettings`, `KEY.scope`, `KEY.languages`) | `SettingGetContext`, `ParseSettingChanged` | C |
| `onDidChangeConfiguration` | `Handler::setting` | `setting` events | C |
| `getConfiguration().update(…)` | none: settings belong to the user | | N |
| `getConfiguration('editor')` (other sections) | none to read; `config.SECTION.KEY` in when clauses only | | N |
| `workspaceFolders`, `rootPath` | `ready`'s `project`; several folders: `parse_folders(ev)`, `ext.folder_of(path)` (feature `WorkspaceFolders`) | `ParseFolders`, `FolderOfContext` | C |
| `findFiles(include, exclude, max)` | `ext.find_files(&files::FileQuery{…}, \|batch\| true)` (feature `Files`, `project = read`); `.gitignore` applies | `FindFilesContext(ctx, rhunext.FileQuery{…}, fn)` | C |
| `findTextInFiles` | `ext.search_text(&Pattern::Literal(s), &q, \|batch\| true)` | `SearchTextContext` | C |
| `fs.readFile`, `fs.writeFile`, `fs.stat`, `fs.readDirectory` | `std::fs` (`[needs] project = read`/`write`, `home = read`) | `os` | C |
| `fs.rename`, `fs.delete`, `fs.createDirectory` with undo | `WorkspaceEdit::new().create(…).rename(…).delete(…)` and `ext.apply` (feature `FileOperations`) | `Create`, `Rename`, `Delete`, `Apply` | C |
| `applyEdit(WorkspaceEdit)` | `ext.apply(&WorkspaceEdit::new().doc(…).file(path, None, edits))` (feature `WorkspaceEdits`) | `Apply(e)` with `AddDoc`, `AddFile` | C |
| `textDocuments`, `onDidOpenTextDocument`, `onDidCloseTextDocument` | `[subscribe] opened = PATTERNS`; `Opened::parse(ev)`; `closed` | `ParseOpened`; `closed` | C |
| `onDidChangeTextDocument` (contentChanges) | `[subscribe] edits = PATTERNS`: byte deltas, `parse_edits`, `apply_edits`; or `changed` (no content) | `ParseEdits`, `ApplyEdits` | C |
| `onDidSaveTextDocument` | `[subscribe] saved = PATTERNS` | `saved` | C |
| `onWillSaveTextDocument` (`waitUntil(edits)`) | `[provides] save_actions`, `Handler::save_actions` (feature `SaveActions`; the user's `code_actions_on_save` kinds) | `SaveActionProvider` | A |
| `TextDocument.getText(range)`, `lineAt`, `lineCount`, `offsetAt`, `positionAt` | `ext.text(doc)` / `ext.range_text(doc, range)` → (bytes, version); compute lines yourself (`porting.md`) | `Text`, `RangeText` | C |
| `createFileSystemWatcher(glob)` | `ext.watch_workspace()`, `parse_workspace_watch(ev)`: no paths, reconcile (feature `WorkspaceWatch`) | `WatchWorkspace`, `ParseWorkspaceWatch` | A |
| `registerTextDocumentContentProvider(scheme)` | `[provides] schemes = NAME`, `Handler::content(ext, uri)`, `ext.content_changed(uri)` (feature `Uris`) | `ContentProvider`, `ContentChangedContext` | C |
| `registerFileSystemProvider` | none (virtual documents are read-only) | | N |
| `registerTaskProvider`, `tasks.executeTask`, `fetchTasks` | `ext.set_tasks(&[tasks::Task::new(id, label, Program::argv(&[…]))])`, `start_task(id)`, `run_task(&task)` (features `Terminals`, `Tasks`) | `SetTasks`, `StartTask`, `RunTask` | C |
| `onDidEndTaskProcess` | `tasks::parse_terminal_event(ev)` (`Ended { code, … }`) | `ParseTerminalEvent` | C |
| `registerNotebookSerializer` | `[provides] notebook`, `Handler::nb_open`, `nb_save` (feature `Notebooks`) | `NotebookProvider` | C |
| `isTrusted`, `onDidGrantWorkspaceTrust` | the folder's trusted flag in `parse_folders`; `gitTrusted` in when clauses | `ParseFolders` | A |

## languages

Each provider needs its feature in the run call and its `[provides]` key with file patterns
(`rhun-extension/reference/language.md`, `presentation.md`). VS Code's `DocumentSelector`
(`{ language: 'go' }`) becomes patterns (`*.go`).

| VS Code `register…Provider` | `[provides]` | Rust `Handler::` | Go interface / method | |
| --- | --- | --- | --- | --- |
| `CompletionItemProvider` (+ `resolveCompletionItem`) | `complete`, `complete.triggers` | `complete`, `complete_resolve` | `Complete`, `CompletionResolver` | C |
| `HoverProvider` | `hover` | `hover` (`HoverQuery`) | `HoverProvider` | C |
| `SignatureHelpProvider` | `signature`, `signature.triggers` | `signatures` (`SignatureList`) | `SignatureLister` | C |
| `DefinitionProvider`, `ReferenceProvider` | `definition`, `references` | `definition`, `references` | `Definition`, `References` | C |
| `DeclarationProvider`, `TypeDefinitionProvider`, `ImplementationProvider` | `declaration`, `type_definition`, `implementation` | `declaration`, `type_definition`, `implementation` (`Navigation`) | `Declarer`, `TypeDefiner`, `Implementer` | C |
| `DocumentSymbolProvider`, `WorkspaceSymbolProvider` | `symbols`, `workspace_symbols` | `symbols`, `workspace_symbols` | `Symbols`, `WorkspaceSymbolProvider` | C |
| `CodeActionsProvider` | `actions` | `code_actions`, `code_action_resolve` (`CodeActions`) | `CodeActionProvider2`, `CodeActionResolver` | C |
| `RenameProvider` (+ `prepareRename`) | `rename` | `rename`, `prepare_rename` | `Rename`, `RenamePreparer` | C |
| `DocumentFormattingEditProvider`, `…RangeFormatting…`, `OnTypeFormatting…` | `format`, `range_format`, `type_format` | `format`, `range_format`, `type_format` (`PartialFormat`) | `Format`, `RangeFormatter`, `TypeFormatter` | C |
| `CodeLensProvider` | `code_lens` | `code_lenses` | `CodeLensProvider` | C |
| `InlayHintsProvider` | `inlay_hints` | `inlay_hints` | `InlayHintProvider` | C |
| `DocumentSemanticTokensProvider` (+ range) | `semantic_tokens` | `semantic_tokens`, `ext.semantic_legend` | `SemanticTokensProvider`, `SemanticLegend` | C |
| `FoldingRangeProvider` | `folding` | `folding_ranges` | `FoldingRangeProvider` | C |
| `DocumentHighlightProvider` | `highlights` | `highlights` | `HighlightProvider` | C |
| `DocumentLinkProvider` | `document_links` | `document_links` | `DocumentLinker` | C |
| `ColorProvider` | `colors` | `colors`, `color_edit` | `ColorProvider` | C |
| `InlineCompletionItemProvider` | `inline_complete` | `inline_complete` | `InlineCompleter` | C |
| `SelectionRangeProvider` | `selection_ranges` | `selection_ranges` | `SelectionRanger` | C |
| `CallHierarchyProvider`, `TypeHierarchyProvider` | `call_hierarchy`, `type_hierarchy` | `call_hierarchy`, `calls_in`, `calls_out`, `type_hierarchy`, `supertypes`, `subtypes` | `CallHierarchyProvider`, `TypeHierarchyProvider` | C |
| `LinkedEditingRangeProvider` | `linked` | `linked` (`LinkedEditing`) | `LinkedEditor` | C |
| `createDiagnosticCollection().set(uri, diags)` | (`documents = read`) | open documents: `ext.diagnostics(doc, version, &[Diagnostic{…}])`; any file: `ext.file_diagnostics(&Target::Path(p), None, &ds)` (`FileDiagnostics`) | `Diagnostics`; `FileDiagnostics(rhunext.Target{Path: p}, …)` | C |
| `Diagnostic.tags` (Unnecessary, Deprecated), `relatedInformation`, `code`, `codeDescription` | | `Diagnostic { tags, related, code, href, … }` | the same fields | C |
| `setLanguageConfiguration` | `[language.NAME]` (static) | | | A |
| `setTextDocumentLanguage`, `getLanguages`, `match` | none: rhun picks the grammar by file name | | | N |
| `EvaluatableExpressionProvider`, `InlineValuesProvider`, `DocumentDropEditProvider`, `DocumentPasteEditProvider`, `createLanguageStatusItem` | none (a status item for the last one) | | | N |

### Language clients (`vscode-languageclient`)

An extension that starts a language server and wires it to VS Code converts best by not porting
the client: rhun-editor/lsp already is one, with every feature above. Its servers come from a
built-in table (gopls, rust-analyzer, typescript-language-server, pyright, clangd, zls,
lua-language-server, ...: its README) and `$RHUN_STATE/lsp.ini` `[servers] LANGUAGE = command`;
server settings and initialization options go in its `settings.json`. So:

- the server is in lsp's table (or replaces one for a language it covers): the conversion is a
  pack (`pack = rhun-editor/lsp`) plus the grammar and language keys, and a README that gives the
  `lsp.ini` and `settings.json` lines the VS Code extension's settings used to send
  (`initializationOptions`, `workspace/configuration` sections);
- the server is for a language or file type lsp does not list: lsp's `[provides]` patterns and
  `[needs] run` are fixed in its manifest, and no configuration widens them. Fork rhun-editor/lsp
  (Go): add the patterns, the program in `run`, the server's table entry and its middleware
  (custom requests, commands the VS Code client handled), or write a client on the SDK
  (`rhun-extension/reference/language.md`, `editing.md` for `workspace/applyEdit`);
- the VS Code extension downloads its server: rhun runs only what is installed; give the user
  the install command (`install` in a README, or a toast), never download without
  `network = yes` and saying so.

## debug

| VS Code | rhun | |
| --- | --- | --- |
| a `debuggers` contribution + `DebugAdapterExecutable`/`Server` | `[debugger.TYPE]` (`contributes.md`); rhun speaks DAP to the adapter itself | C |
| `registerDebugConfigurationProvider` (`resolveDebugConfiguration`) | `resolve = yes` + Rust `Handler::resolve_debug(ext, &DebugResolve) -> Resolved`, Go `DebugResolver.ResolveDebug` (feature `Debug`) | C |
| `provideDebugConfigurations` | `templates = debug/TYPE.ini` | C |
| `startDebugging(folder, config)` | Rust `ext.debug_start(name, &DebugConfig)`, Go `DebugStart(ctx, name, cfg)`: within 10 s of one of its commands | C |
| `stopDebugging` | `ext.debug_stop(session)`, `DebugStop` | C |
| `onDidTerminateDebugSession` | `DebugEnded::parse(ev)`, `ParseDebugEnded` | C |
| `onDidStartDebugSession`, `activeDebugSession`, `customRequest`, `breakpoints`, `addBreakpoints`, `registerDebugAdapterTrackerFactory`, `activeStackItem` | none: no DAP crosses the extension wire | N |
| `DebugAdapterInlineImplementation` | the adapter as a program (the extension's binary in an adapter mode) | A |

## scm and Git

| VS Code | Rust | Go | |
| --- | --- | --- | --- |
| `scm.createSourceControl(id, label, root)` | `ext.scm_register(root, label)` → SCM (feature `Scm`) | `ScmRegister(ctx, root, label)` | C |
| `createResourceGroup`, `resourceStates`, `SourceControlResourceState.command`, `decorations` | `ext.scm_update(scm, &ScmPanel{ groups: vec![ScmGroup{ resources: vec![ScmResource{…}], … }], … })` (the whole panel each time) | `ScmUpdate(ctx, scm, rhunext.ScmPanel{…})` | C |
| `inputBox.value`, `acceptInputCommand` | `ext.scm_input(scm)`, `scm_input_set`; the panel's first action is Ctrl+Enter | `ScmInput`, `ScmInputSet` | C |
| `quickDiffProvider` | `[provides] quickdiff`, `Handler::quickdiff`, `ext.quickdiff_refresh` (feature `QuickDiff`) | `QuickDiffProvider`, `QuickDiffRefresh` | C |
| `count`, `statusBarCommands` | the panel's status text; a status item | | A |
| the Git extension's API (`getExtension('vscode.git').exports.getAPI(1)`: repositories, state, blame, log, show) | `ext.git_repo()`, `git_status()`, `git_blame(doc)`, `git_log(LogOf::File(p), max)`, `git_show(rev, path)`, `git_diff_open`, `scm::is_git_changed(ev)` (feature `GitData`, `[needs] git = read`) | `GitRepo`, `GitStatus`, `GitBlame`, `GitLog`, `GitShow`, `GitDiffOpen`, `IsGitChanged` | C |
| Git operations (commit, push, checkout) through that API | run `git` yourself (`[needs] run = git`, `project = write`) | `os/exec` | A |
| a commit message generator (SCM input action) | `[provides] commit_message = yes`, `Handler::commit_message` (feature `CommitMessage`) | `CommitMessageProvider` | C |

## tests

| VS Code | Rust | Go | |
| --- | --- | --- | --- |
| `tests.createTestController(id, label)` | `[provides] tests = PATTERNS` (feature `Tests`) | the same | C |
| `controller.resolveHandler`, `items.add`, `createTestItem(id, label, uri)` | `Handler::resolve_tests(ext, parent)`, `ext.set_test_items(parent, &items, replace)`, `TestItem::new(TestKind::Test, id, label).at(path, Some(range))` | `TestController.ResolveTests`, `SetTestItems`, `rhunext.TestItem{…}` | C |
| `createRunProfile(kind Run/Debug/Coverage, handler)` | `tests.profiles = run debug coverage`; `Handler::run_tests(ext, &TestRun)` (`run.profile`) | `RunTests` | C |
| `createTestRun`, `run.started/passed/failed(message)/skipped`, `appendOutput`, `end` | `TestReporter::new(&ext, run)`, `TestResult::new(id, TestState::Failed).message(TestMessage::new(t).values(exp, act))`, `TestResult::output`, `ext.end_test_run(run)` | `NewTestReporter`, `TestResult{…}`, `EndTestRun` | C |
| spawning the test runner | `ext.test_exec(run, &TestCommand::new("go").args([…]))` (`[needs] tests = go`) | `TestExec` | C |
| `run.addCoverage(FileCoverage)` | `ext.report_coverage(run, &[FileCoverage{…}])` (line counts only) | `ReportCoverage` | C/A |
| `token.onCancellationRequested` | `Handler::cancel_tests` | `CancelTests` | C |

## notebooks

| VS Code | rhun | |
| --- | --- | --- |
| `registerNotebookSerializer` | `[provides] notebook`, Rust `nb_open`/`nb_save`, Go `NotebookProvider` | C |
| `createNotebookController` (`executeHandler`) | `notebook.kernel = yes`, Rust `nb_run`, `ext.nb_state`, `nb_output`, `nb_stream`; Go `NbRun`, `NbState`, `NbOutput`, `NbStream` | C |
| a kernel process | Rust `ext.nb_exec(nb, &Command{…})`, Go `NbExec` (`[needs] kernels = python3`) | C |
| `NotebookCellOutputItem` mime types | text, ANSI, PNG, JPEG, GIF items; HTML and JSON become text | A |
| `notebookRenderer`, `createRendererMessaging`, notebook toolbar menus | none | N |

## env

| VS Code | Rust | Go | |
| --- | --- | --- | --- |
| `env.clipboard.readText`, `writeText` | none (only a picture view's copy: `Handler::view_copy`). Insert the text into a document, show it in a card or a doc view for the user to copy | | N |
| `env.openExternal(uri)` | `ext.open_url(url)` (http and https; the user sees it) (feature `Urls`) | `OpenURLContext` | C |
| `env.asExternalUri` | `ext.external_uri(uri)` (remote windows forward it) | `ExternalURIContext` | C |
| `env.appName`, `appRoot`, `uriScheme` | `RHUN_VERSION`; the scheme is `rhun` | `os.Getenv` | A |
| `env.remoteName` | `RHUN_REMOTE`; `remoteName` in clauses | | C |
| `env.language`, `machineId`, `sessionId`, `uiKind`, `shell`, `logLevel` | none | | N |
| `env.isTelemetryEnabled`, `createTelemetryLogger` | none: drop telemetry (and say so) | | N |
| `process.platform`, `os.homedir()` | `RHUN_TARGET` (`linux-x86_64`, `macos-arm64`, `windows-x64`); `HOME` is the state folder, not the user's home | | A |

## authentication

| VS Code | rhun | |
| --- | --- | --- |
| `authentication.getSession('github' \| 'microsoft', scopes)` | no built-in providers. Sign in yourself: OAuth with PKCE (`rhun_ext::auth::Pkce`, `Loopback`; Go `NewPKCE`, `ListenLoopback`), `ext.external_uri` + `ext.open_url` for the browser, the token kept with `secret_set` (features `Urls`, `Secrets`; `network = yes`). A personal access token typed by the user (`ask`, `Flag::Password`) is the simple path | A |
| `registerAuthenticationProvider`, `onDidChangeSessions` | none | N |

## Others

| VS Code | rhun | |
| --- | --- | --- |
| `vscode.Uri` (`file:`, `untitled:`) | absolute paths; `#DOC` for an untitled document; your own scheme with `uris` | C |
| `Position`, `Range`, `Selection` | `Pos::new(line, col)`, `Range::new(a, b)`, `Selection::new(anchor, cursor)` (Go `rhunext.Pos{Line, Col}`, `Range{Start, End}`, `Selection{Anchor, Cursor}`): 1-based, byte columns (`porting.md`) | C |
| `CancellationToken` | Rust: `0 cancel ID` ends a request, the SDK drops the late answer; Go `req.Context` | C |
| `EventEmitter` | your own state; rhun's events arrive in `Handler::event` | C |
| `lm.*`, `chat.*` (Copilot) | `agents`, `tools`, `edit-review` features (`ai.md`): rows and tools, no language model of rhun's | A |
| `comments.createCommentController` | none; diagnostics, decorations with tips, or a tree view | N |
| `window.createWebviewPanel` charts and previews | a picture view (`rhun_ext::chart` bar, line and pie charts), a doc view | A |
