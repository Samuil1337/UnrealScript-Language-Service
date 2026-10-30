# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A VS Code extension (`EliotVU.uc`) providing an UnrealScript language server. Two TypeScript packages, each with its own `package.json` (root `npm install` runs `postinstall` to install `client/` and `server/` too):

- `client/` — thin VS Code extension that launches the server (`client/src/extension.ts`).
- `server/` — the LSP server (`server/src/server.ts`) and all language logic.

Both are bundled by esbuild into `out/extension.js` and `out/server.js`.

## Commands

```sh
npm install                 # also installs client/ and server/
npm run compile:grammar     # generate ANTLR lexer/parser -> server/src/UC/antlr/generated (gitignored, REQUIRED before tests/compile)
npm run compile:preprocessor# generate preprocessor parser (also required)
npm run build               # full production build: clean, gulp (grammar + syntax + copy presets), esbuild, tmLanguage
npm run compile             # esbuild only (no grammar regen)
npm run watch               # esbuild watch, used by the "Launch Client (Watch)" debug config (F5)
npm test                    # server unit tests (mocha via ts-mocha)
```

Run a single test file or filter by name (from `server/`):

```sh
cd server
npx ts-mocha -p ./tsconfig.test.json src/UC/test/class/class.test.ts
npx ts-mocha -p ./tsconfig.test.json "src/**/*.test.ts" -g "ClassSymbol"
```

There is no npm lint script; CI (`.github/workflows/eslint.yml`, on `develop`) runs `npx eslint .`. Config is `.eslintrc.json`: semicolons required, 4-space indent.

`npm run vscode:test` / `.vscode-test.js` targets `out/test/**` which does not exist yet — only the server mocha tests are real.

## Architecture

### Pipeline: text → symbols → indexed references

1. **Grammar** (`grammars/*.g4`, antlr4ts). The parser has `generation` (1/2/3) and `licensee` fields used in semantic predicates (e.g. `{ this.generation < 3 }? 'expands'`), so one grammar covers UE1–UE3 and licensee dialects. After editing a `.g4`, regenerate with `compile:grammar` / `compile:preprocessor`.
2. **Build** — `UCDocument.build()` (`server/src/UC/document.ts`): lexes, runs the UC3 preprocessor (`` `define ``/macros) into the token stream, parses (SLL first, falls back to LL), then `DocumentASTWalker` (`documentASTWalker.ts`) converts the parse tree into symbols (`UC/Symbols/`), expressions (`expressions.ts`) and statements (`statements.ts`). Parse errors become diagnostic nodes on the document.
3. **Index** — `indexDocument()` in `server/src/UC/indexer.ts` runs `DocumentSymbolIndexer` (declarations/types). Dependencies discovered during indexing are queued, and once all pending documents are indexed `DocumentCodeIndexer` runs a post-index pass that resolves code bodies and references (needs all dependencies resolved first).
4. **Consumers** — LSP feature modules in `server/src/` (`completion.ts`, `documentHover.ts`, `references.ts`, `rename.ts`, `documentTokenSemantics.ts`, ...) query the indexed symbols. Diagnostics come from `DocumentAnalyzer` (`UC/diagnostics/documentAnalyzer.ts`), with message templates in `diagnosticMessages.json`.

Walkers over the symbol tree extend `DefaultSymbolWalker` (`UC/symbolWalker.ts`) — indexers, the analyzer and builders all use this visitor pattern.

### Global state

`indexer.ts` holds module-level global state: `documentsByPathMap`, `documentsMap` (keyed by `NameHash`), `IndexedReferencesMap`, the `config` object, and rxjs Subjects (`documentBuilt$`, `documentIndexed$`, `documentCodeIndexed$`, `documentsCodeIndexed$`). `server.ts` wires LSP events to these streams with debouncing (`indexDocumentDebouncePeriod`, `analyzeDocumentDebouncePeriod`) and awaits them before answering requests. Tests share this global state too.

Names are interned: use `toName()` / `NameHash` (`UC/name.ts`, common names in `UC/names.ts`) rather than comparing raw strings. The hash (`packages/crc-32`, `crc32_str`) is case-insensitive, matching UnrealScript identifier semantics.

### Presets and intrinsics

`server/src/presets/{Shared,UE1,UE2,UE3}` are bundled `.uc` stubs for Core/Engine classes; `registerPresetsWorkspace()` loads the set matching the detected/configured generation. Intrinsic (C++-only) symbols are in `UC/Symbols/CoreSymbols.ts`, `EngineSymbols.ts`, `IntrinsicSymbols.ts`, plus user-configurable `unrealscript.intrinsicSymbols`. `server/src/UPK/` reads `.u`/`.upk` package headers (package names only, not contents).

## Tests

- Test files are `*.test.ts` colocated in `server/src/`. Feature tests live in `server/src/UC/test/<feature>/` alongside `.uc` fixtures.
- Typical pattern: `usingDocuments(__dirname, [files], cb)` (`UC/test/utils/utils.ts`) registers fixtures (plus `UnrealScriptTests/Core/Classes/Object.uc`), then `queueIndexDocument(doc)` and `assertDocumentValidFieldsAnalysis(doc, /\bShould(?!BeInvalid)/i)` / `assertDocumentInvalidFieldsAnalysis(doc, /\bShouldBeInvalid/i)`.
- Fixture convention: fields/functions named `Should...` must produce no diagnostics; those named `ShouldBeInvalid...` must produce at least one.
- `server/src/UC/test/UnrealScriptTests/` contains real Core/Engine sources used as dependencies.

## Repo notes

- Git-flow branching: `develop` is the integration branch (CI target), `master` holds releases (`Hotfix-x.y.z` / `Bump x.y.z` commits). Release version is bumped in both root and `server/package.json` (`client/package.json` has its own, unrelated version).
- `unlens/` is a git submodule (`../UnrealScript-Unlens`, relative URL).
- `syntaxes/*.YAML-tmLanguage` are the TextMate grammars, converted to JSON into `out/` by `compile:syntax`.
