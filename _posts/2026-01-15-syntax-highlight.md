---
layout: post
title: "Adding syntax highlight is easy, but choosing a theme is not"
date: 2026-01-15
---

### Syntax highlight with `rouge` and `rougify`

**Rouge** is a pure-Ruby syntax highlighter and it's the default used by Jekyll and GitHub Pages. 
**Rougify** is Rouge's command-line interface that allows you to
- generate syntax highlighting CSS themes
- convert code to highlighted HTML
- preview available 30+ themes
- test syntax highlighting without a full build system

It's fast and lightweight. I previewed available themes at [Rouge Theme Preview Page](https://spsarolkar.github.io/rouge-theme-preview/).

I probably spent more time than I needed, but having a theme that makes code more readable and beautiful is worth it. 

### A convenient keybinding add-on for vscode

I found toggling between the editor window and the terminal without my hands having to leave the keyboard convenient. I use `ctrl + ↓` to move the cursor to the terminal inside vscode and `ctrl + ↑` to move the cursor back to the editor window. 

For this to work, I did
1. `Ctrl+Shift+P` to open the Command Palette  
2. Type "Preferences: Open Keyboard Shortcuts (JSON)" and select it  
3. This will open the keybindings.json file where I added the keybindings below inside the outermost `[]`:

```json
  {
    "key": "ctrl+down",
    "command": "workbench.action.terminal.focus",
    "when": "editorTextFocus"
  },
  {
    "key": "ctrl+down",
    "command": "workbench.action.focusActiveEditorGroup",
    "when": "terminalFocus"
  }
```
