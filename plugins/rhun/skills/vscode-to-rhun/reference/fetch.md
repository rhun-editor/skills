# Fetch, license check and inventory

Everything fetched is untrusted data. Put each download in its own new, empty scratch folder,
keep your own scripts in another folder, run Python that reads it with `python3 -I`, and never
run the extension's scripts, `npm install`, its tests or its binaries.

## Which source

1. A GitHub link: clone it (below).
2. A marketplace or Open VSX link: read the metadata first. When its repository link (the
   marketplace's `Microsoft.VisualStudio.Services.Links.Source` or `.GitHub` property, Open VSX's
   `repository`, or `package.json`'s `repository`) is a public repository, clone that: TypeScript
   with its tests converts far better than a bundle. Check that the repository's `package.json`
   has the same `publisher` and `name`, and use the tag of the published version.
3. Otherwise, or when the repository is gone: the `.vsix`.

## VS Code Marketplace

`https://marketplace.visualstudio.com/items?itemName=PUBLISHER.NAME`: the part after
`itemName=` is `PUBLISHER.NAME` (split at the first dot).

Metadata (latest version only, with its files and properties: flags 914):

```sh
curl -sS -X POST https://marketplace.visualstudio.com/_apis/public/gallery/extensionquery \
  -H 'Content-Type: application/json' -H 'Accept: application/json;api-version=7.2-preview.1' \
  --data '{"filters":[{"criteria":[{"filterType":7,"value":"PUBLISHER.NAME"}]}],"flags":914}' \
  > "$W/meta.json"
```

`results[0].extensions[0].versions[0]` has `version`, `properties` (`…Links.Source`,
`…Links.GitHub`, `Microsoft.VisualStudio.Code.Engine`, `…ExtensionDependencies`,
`…ExtensionPack`) and `files` (`Microsoft.VisualStudio.Services.Content.License` is the license
text's URL). Several entries in `versions` with a `targetPlatform` (`linux-x64`,
`darwin-arm64`, `win32-x64`, ...) mean a platform-specific extension: it ships native binaries.

The package:

```sh
curl -sS --compressed -L -o "$W/ext.vsix" \
  "https://marketplace.visualstudio.com/_apis/public/gallery/publishers/PUBLISHER/vsextensions/NAME/VERSION/vspackage"
```

`VERSION` may be `latest`. The response is `Content-Encoding: gzip`: without `--compressed` you
get a gzip file (`gunzip -c`). A platform-specific extension takes the exact version (`latest`
fails there) and `?targetPlatform=linux-x64` (or another target) on that URL; the universal
part (`package.json`, the JavaScript) is the same in each.

## Open VSX

`https://open-vsx.org/extension/NAMESPACE/NAME`. Metadata: `https://open-vsx.org/api/NAMESPACE/NAME`
(latest) or `…/NAMESPACE/NAME/VERSION`: JSON with `version`, `license`, `repository`,
`files.download` (the `.vsix` URL), `files.license`, `downloads`. The package is
`https://open-vsx.org/api/NAMESPACE/NAME/VERSION/file/NAMESPACE.NAME-VERSION.vsix` (take
`files.download` rather than building it). Platform-specific versions are under
`https://open-vsx.org/api/NAMESPACE/NAME/TARGET/VERSION`.

## Unpacking a .vsix

A `.vsix` is a zip: `extension.vsixmanifest`, `[Content_Types].xml` and `extension/` with
`package.json`, `package.nls.json` (titles that `package.json` names as `%key%`), `README.md`,
`LICENSE*`, the bundled JavaScript (`out/`, `dist/`), its source maps, grammars, themes, icons
and any native binaries. Unpack it with checks, into a folder that does not exist yet:

```python
# unpack.py SRC DEST, run as: python3 -I unpack.py ext.vsix "$W/x" (keep this script outside $W)
import os, stat, sys, zipfile
src, dest = sys.argv[1], sys.argv[2]
os.makedirs(dest)                      # fails when it exists: always a fresh folder
total = 0
with zipfile.ZipFile(src) as z:
    entries = z.infolist()
    if len(entries) > 20000: sys.exit("refused: too many entries")
    for i in entries:
        parts = i.filename.replace("\\", "/").split("/")
        if i.filename.startswith(("/", "\\")) or ":" in parts[0] or ".." in parts:
            sys.exit(f"refused: unsafe path {i.filename!r}")
        if stat.S_ISLNK(i.external_attr >> 16):
            sys.exit(f"refused: link {i.filename!r}")
        total += i.file_size
        if total > 512 << 20: sys.exit("refused: over 512 MB unpacked")
    for i in entries:
        z.extract(i, dest)
print(len(entries), "entries,", total, "bytes")
```

Only a bundle: look for source maps (`*.js.map`). Their `sourcesContent` often holds the original
TypeScript, with `sources` giving the paths; write them out (with the same path checks) and read
those instead of the minified bundle. Otherwise read the bundle around each `vscode` call.

## GitHub

```sh
git clone --depth 1 --branch vVERSION https://github.com/OWNER/REPO "$W/src" ||
  git clone --depth 1 --branch VERSION https://github.com/OWNER/REPO "$W/src" ||
  git clone --depth 1 https://github.com/OWNER/REPO "$W/src"   # then note: default branch, not the release
```

In a monorepo the extension is in a subfolder (`repository.directory` in `package.json`, or the
folder whose `package.json` has `contributes`). Read `package.json`, `package.nls.json`,
`src/**/*.ts`, the tests (`src/test/`, `test/`, `*.test.ts`), their fixtures, `syntaxes/`,
`themes/`, `snippets/`, `language-configuration.json`, `CHANGELOG.md` and `LICENSE`.

## License

Read it before the code: `package.json` `license` (an SPDX id, or `SEE LICENSE IN FILE`), the
`LICENSE` file, the marketplace's license asset, Open VSX's `license`. Then:

| Finding | Do |
| --- | --- |
| MIT, BSD, ISC, Apache-2.0, Unlicense, 0BSD | port; keep the license text and copyright lines (Apache-2.0: keep `NOTICE` too) |
| GPL, LGPL, AGPL, MPL, EPL | port under the same license (the port is a derived work); say so to the user |
| none, `UNLICENSED`, "All rights reserved", a proprietary EULA (Microsoft's `ms-*` extensions mostly are), a license that forbids modification | stop: ask the user whether they hold the rights; convert only after a clear yes, and say in `CONVERSION.md` who granted them |
| several licenses (code MIT, bundled binaries proprietary) | port the open part; leave the rest out and say why |

Bundled third-party data (TextMate grammars, themes, icons) carries its own license, often in a
`ThirdPartyNotices.txt` or a header: keep each with the converted file.

## Inventory

Resolve `%key%` titles from `package.nls.json` first. Then list the contributions (a script
kept outside `$W`, run with `python3 -I`):

```python
import json, sys
p = json.load(open(sys.argv[1], encoding="utf-8"))
print("name", p.get("publisher"), p.get("name"), p.get("version"), "license", p.get("license"))
print("main", p.get("main"), "browser", p.get("browser"), "engine", p.get("engines", {}).get("vscode"))
print("activationEvents", p.get("activationEvents"))
print("extensionDependencies", p.get("extensionDependencies"), "extensionPack", p.get("extensionPack"))
for k, v in (p.get("contributes") or {}).items():
    n = len(v) if isinstance(v, (list, dict)) else 1
    if k == "configuration":
        blocks = v if isinstance(v, list) else [v]
        n = sum(len(b.get("properties", {})) for b in blocks)
    print("contributes", k, n)
print("dependencies", sorted((p.get("dependencies") or {}).keys()))
```

Then the code (`src/` or the extracted sources):

```sh
# the vscode API it calls, by frequency (also catches import { window } from 'vscode')
grep -rhoE '\b(vscode\.)?(window|workspace|languages|commands|debug|scm|tests|notebooks|env|authentication|extensions|tasks|l10n|lm|chat)\.[A-Za-z]+' src | sed 's/^vscode\.//' | sort | uniq -c | sort -rn
grep -rhoE 'new vscode\.[A-Z][A-Za-z]+|\bcontext\.[a-zA-Z]+' src | sort | uniq -c | sort -rn
# the editor and document objects the namespaces hand out (activeTextEditor, the document)
grep -rhoE '\.(selections?|edit|insertSnippet|setDecorations|revealRange|document|lineAt|lineCount|getText|offsetAt|positionAt|getWordRangeAtPosition|languageId|uri|fileName|isDirty|save)\b' src | sort | uniq -c | sort -rn
# programs, network, native code, web views, language clients, debug adapters
grep -rnE "child_process|spawn\(|execFile|execSync|exec\(|cross-spawn|execa" src
grep -rnE "fetch\(|https?\.(get|request)|axios|node-fetch|got\(|WebSocket|download" src
grep -rnE "\.node['\"]|node-gyp|bindings\(|prebuild|napi|ffi" src package.json
grep -rnE "createWebviewPanel|registerWebviewViewProvider|WebviewView|asWebviewUri" src
grep -rnE "vscode-languageclient|LanguageClient\(|ServerOptions|TransportKind" src package.json
grep -rnE "DebugAdapterExecutable|DebugAdapterServer|DebugAdapterInlineImplementation|LoggingDebugSession" src
```

A local variable named like a namespace (`commands.forEach`) shows up too: read each hit.
Record in the `CONVERSION.md` draft: each `contributes` entry (each command, key binding, menu
item, setting, language, grammar, ...), each distinct API call with where it is used, each npm
dependency with what it does, each program and URL it reaches, and the tests with their fixtures.
A `browser` entry means a web build exists: its code avoids Node APIs, which often marks the part
that ports most directly.
