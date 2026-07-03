# packages/tui — `@earendil-works/pi-tui`

Terminal UI framework with differential rendering. Used by the interactive mode of the coding agent. Provides a component-based architecture for building flicker-free terminal applications.

## Core concepts

### Differential rendering

The `TUI` class (`src/tui.ts`, ~58KB) manages rendering using three strategies:

1. **Full re-render** — only when terminal dimensions change
2. **Line-level diff** — only re-renders changed lines
3. **CSI 2026 synchronized output** — atomic screen updates (supported terminals)

Combined with bracketed paste mode and Kitty/iTerm2 graphics protocols, this produces a smooth, flicker-free terminal UI.

### Component model

All UI elements implement the `Component` interface:

```typescript
interface Component {
  render(width: number): string[];
  handleInput?(data: string): boolean;
  // focus management, invalidation, hardware cursor
}
```

Components are composed into a tree. The TUI manages focus, input routing, and re-rendering.

### Components

| Component | File | Description |
|-----------|------|-------------|
| `Text` | `text.ts` | Simple text display |
| `TruncatedText` | `truncated-text.ts` | Text truncated to fit width |
| `Input` | `input.ts` | Single-line text input |
| `Editor` | `editor.ts` (77KB) | Full multi-line text editor — the largest component |
| `Markdown` | `markdown.ts` | Markdown renderer with theme support |
| `Loader` | `loader.ts` | Animated loading indicator |
| `CancellableLoader` | `cancellable-loader.ts` | Loader with cancel support |
| `SelectList` | `select-list.ts` | Selectable list with keyboard navigation |
| `SettingsList` | `settings-list.ts` | Settings UI list |
| `Box` | `box.ts` | Layout container with padding/borders |
| `Spacer` | `spacer.ts` | Flexible spacer for layout |
| `Image` | `image.ts` | Inline image rendering (Kitty/iTerm2) |

## Key systems

### Keyboard handling

- **`keys.ts`** (45KB) — keyboard input parsing, Kitty protocol support, key matching
- **`keybindings.ts`** — configurable keybinding system with conflict detection
- **`stdin-buffer.ts`** — input buffering for batch splitting

### Terminal integration

- **`terminal.ts`** — `Terminal` interface and `ProcessTerminal` implementation
- **`terminal-image.ts`** — Kitty/iTerm2 image protocol rendering, image dimension detection
- **`terminal-colors.ts`** — terminal color scheme detection via OSC 11
- **`native-modifiers.ts`** — native modifier key handling

### Utilities

- **`utils.ts`** (32KB) — text utilities: visible width, ANSI wrapping, column slicing, truncation
- **`autocomplete.ts`** (23KB) — autocomplete providers for file paths and slash commands
- **`fuzzy.ts`** — fuzzy matching/filtering
- **`kill-ring.ts`** — Emacs-style kill ring
- **`undo-stack.ts`** — undo/redo stack
- **`word-navigation.ts`** — word boundary navigation

## Native modules

Prebuilt native addons for `darwin` and `win32` platforms in `native/`.

## Package structure

```
packages/tui/
  src/
    index.ts              Public API surface
    tui.ts                Core TUI class, Component interface, rendering engine (~58KB)
    terminal.ts           Terminal abstraction
    keys.ts               Keyboard input parsing (~45KB)
    keybindings.ts        Configurable keybindings
    stdin-buffer.ts       Input buffering
    autocomplete.ts       Autocomplete providers (~23KB)
    fuzzy.ts              Fuzzy matching
    kill-ring.ts          Emacs-style kill ring
    undo-stack.ts         Undo/redo stack
    word-navigation.ts    Word boundary navigation
    terminal-image.ts     Kitty/iTerm2 image protocols
    terminal-colors.ts    Terminal color detection
    utils.ts              Text utilities (~32KB)
    native-modifiers.ts   Native modifier keys
    editor-component.ts   Editor component interface
    components/           UI components
      text.ts, truncated-text.ts, input.ts, editor.ts, markdown.ts,
      loader.ts, cancellable-loader.ts, select-list.ts, settings-list.ts,
      box.ts, spacer.ts, image.ts
  native/                 Prebuilt native addons
  test/                   ~33 test files
```

## Testing

The test suite (~33 files) covers all components, rendering, overlays, autocomplete, editor, markdown, terminal images, key parsing, fuzzy matching, and CJK/regional-indicator edge cases.

## Related docs

- [packages/coding-agent/docs/tui.md](../../packages/coding-agent/docs/tui.md) — end-user TUI documentation