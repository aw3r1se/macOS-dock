# macos-dock

A macOS-like Dock for Vue 3: auto-hiding bar with app icons, launch bounce,
running indicators, minimized window tiles and right-click menus that behave
like the real ones.

## 🔧 Installation

```sh
npm i @aw3r1se/macos-dock
```

The package ships Vue SFC sources, so it needs a bundler with Vue support
(Vite + `@vitejs/plugin-vue`, Vue CLI, Nuxt, …).

## ✏️ Usage

```vue
<script setup>
  import { ref, computed } from 'vue';
  import { AppDock } from '@aw3r1se/macos-dock';
  import TerminalIcon from './TerminalIcon.vue';

  const running = ref(false);
  const autohide = ref(true);

  const apps = computed(() => [{
    id: 'terminal',
    label: 'Terminal',
    icon: TerminalIcon,
    running: running.value,
    menu: running.value
      ? [{ label: 'Quit', action: () => (running.value = false), alt: { label: 'Force Quit' } }]
      : [{ label: 'Open', action: () => (running.value = true) }],
  }]);
</script>

<template>
  <AppDock
    v-model:autohide="autohide"
    :apps="apps"
    :pinned="!running"
    @activate="(id) => (running = true)"
  />
</template>
```

## 📚 API

### `<AppDock>`

| Prop | Type | Default | |
|---|---|---|---|
| `apps` | `App[]` | — | Icons left of the divider |
| `windows` | `DockWindow[]` | `[]` | Minimized window tiles, right of the divider |
| `pinned` | `boolean` | `false` | Keep the Dock on screen (e.g. no window is open) |
| `autohide` | `boolean` | `true` | Hide until the pointer reaches the bottom edge. `v-model:autohide`, toggled from the divider's menu |
| `size` | `number` | `56` | Tile size, px |

```ts
type App = {
  id: string
  label: string
  icon: Component    // rendered with class="icon"; the same component badges window tiles
  running?: boolean  // dot under the icon; false → true bounces the icon
  menu?: MenuItem[]  // right-click / click-and-hold menu
}

type DockWindow = {
  id: string
  app: string        // owning app's id
  label?: string     // defaults to the app's label
  thumb?: Thumb | null
}

type Thumb = {
  node: HTMLElement  // cloned and scaled into the tile
  width: number      // the window's size, px
  height: number
  place?: (copy: HTMLElement) => void // runs once the copy is in the document
}

type MenuItem =
  | { separator: true }
  | {
      label: string
      action?: () => void
      disabled?: boolean
      mark?: 'check' | 'diamond'          // ✓ current window, ◆ minimized one
      alt?: { label: string, action?: () => void } // shown while Option is held
    }
```

| Event | Payload | |
|---|---|---|
| `activate` | app `id` | App icon clicked |
| `restore` | window `id` | Window tile clicked |
| `update:autohide` | `boolean` | Hiding toggled from the divider's menu |

Exposed methods (via a template ref), for minimize/restore animations:

- `iconCenter(appId)` → `{ x, y }` — viewport center of the app's icon, as it will be once the Dock is shown;
- `thumbRect(windowId, width, height)` → `{ left, top, width, height }` — where a window of that size lands in its tile. The window must already be in `windows`.

### Menus

Right-click, Ctrl-click or click-and-hold on an icon opens its `menu` above it.
While open: the Dock stays shown, tooltips hide, the icon dims; Option swaps in
`alt` items; ↑/↓, Enter/Space and Esc work and keys don't reach the page; press,
drag to an item and release selects it; the chosen item blinks before the menu
closes. A press outside closes the menu (a press on a Dock icon doesn't also
activate it). The divider's menu toggles hiding.

### Styling

| CSS variable | Default |
|---|---|
| `--dock-font` | system UI font |
| `--dock-menu-highlight` | `#0a68d9` |

`<DockMenu>` and `<DockThumb>` are exported too, for use outside the Dock.

## 📄 License

MIT
