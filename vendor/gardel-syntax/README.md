# Gardel Language Support

Official syntax highlighting and language definition for **Gardel** (`.gdl` files).

## Features

- **Language ID**: `gardel`
- **Grammar Scope**: `source.gardel`
- **File Extensions**: `.gdl`
- Accurate syntax highlighting for:
  - Modules, concepts, programs, and reactive programs
  - Contract keywords (`require`, `ensure`, `invariant`, `variant`, `trigger`, etc.)
  - Special identifiers (`Current`, `Result`)
  - Eiffel-style comments (`-- ...`)
  - Full Gardel escape sequences in character and string literals (`\u{...}`, `\n`, etc.)
  - Auto-closing pairs and indentation rules

## Packaging

To package into a `.vsix` file:

```bash
npx @vscode/vsce package
```

## Installing

Install the extension directly via CLI:

```bash
# In Antigravity IDE:
antigravity-ide --install-extension gardel-syntax-0.1.0.vsix

# In VS Code:
code --install-extension gardel-syntax-0.1.0.vsix
```

## GitHub Linguist Submission

This repository provides the canonical TextMate grammar for Gardel (`syntaxes/gardel.tmLanguage.json`).

To submit Gardel to GitHub Linguist:
1. Add an entry to `lib/linguist/languages.yml`:
   ```yaml
   Gardel:
     type: programming
     color: "#a44136"
     extensions:
       - ".gdl"
     tm_scope: source.gardel
     ace_mode: text
     language_id: <assigned-id>
   ```
2. Add this repository as a submodule under `vendor/grammars/`.

## License

[MIT](LICENSE)
