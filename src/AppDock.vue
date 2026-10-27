<script setup>
import { ref, reactive, computed, watch } from 'vue'
import DockThumb from './DockThumb.vue'
import DockMenu from './DockMenu.vue'

const props = defineProps({
  // Apps on the left: [{ id, label, icon: Component, running?, menu? }].
  // `menu` is the right-click menu (items as in DockMenu).
  apps: { type: Array, required: true },
  // Minimized windows, right of the divider (macOS default: windows minimize
  // into their own Dock tile, not into the app icon): [{ id, app, label?, thumb? }].
  // `app` is the owning app's id (its icon badges the tile); `thumb` is a
  // snapshot (see DockThumb), null while the window flies in or out.
  windows: { type: Array, default: () => [] },
  // Keep the dock on screen (e.g. when no window is open)
  pinned: { type: Boolean, default: false },
  // Hide the dock until the pointer reaches the bottom edge (v-model:autohide;
  // toggled from the divider's menu, like macOS)
  autohide: { type: Boolean, default: true },
  // Tile size, px
  size: { type: Number, default: 56 },
})

const emit = defineEmits(['activate', 'restore', 'update:autohide'])

const BOUNCE_MS = 600
// Click-and-hold on an icon opens its menu, like macOS
const HOLD_MS = 700
// Owner of the dock's own menu (the divider's)
const DOCK = Symbol('dock')

const dockEl = ref()
const appEls = new Map()
const windowEls = new Map()
const hovering = ref(false)
const bouncing = reactive(new Set())
// { id, owner: app id | DOCK, anchor, altKey } while a menu is open
const menu = ref(null)
let menuSeq = 0
let holdTimer
// the press that closed a menu (or opened one by holding) doesn't also click
let swallowClick = false

function swallowNextClick() {
  swallowClick = true
  // click follows its pointerup in the same task; whatever the release, stop after it
  document.addEventListener('pointerup', () => setTimeout(() => (swallowClick = false)), { once: true, capture: true })
}

const visible = computed(() => props.pinned || !props.autohide || hovering.value || !!menu.value)
const appById = computed(() => new Map(props.apps.map((app) => [app.id, app])))
const windowLabel = (win) => win.label ?? appById.value.get(win.app)?.label

const setEl = (map, id) => (el) => (el ? map.set(id, el) : map.delete(id))

const dockMenu = computed(() => [
  { label: props.autohide ? 'Turn Hiding Off' : 'Turn Hiding On', action: () => emit('update:autohide', !props.autohide) },
])

// Items stay live while the menu is open (e.g. an app quits meanwhile)
const menuItems = computed(() => {
  const owner = menu.value?.owner
  return owner === DOCK ? dockMenu.value : (appById.value.get(owner)?.menu ?? [])
})

function openMenu(owner, el, e) {
  clearTimeout(holdTimer)
  const items = owner === DOCK ? dockMenu.value : appById.value.get(owner)?.menu
  if (!items?.length) return
  const rect = el.getBoundingClientRect()
  const shift = hiddenShift()
  const anchor = { left: rect.left, top: rect.top - shift, width: rect.width, height: rect.height }
  menu.value = { id: ++menuSeq, owner, anchor, altKey: !!e?.altKey }
}

function closeMenu(e) {
  if (e?.button === 0 && dockEl.value?.contains(e.target)) swallowNextClick()
  menu.value = null
}

// An app bounces while it launches, however it was launched
watch(
  () => props.apps.map((app) => [app.id, !!app.running]),
  (now, before) => {
    const was = new Map(before)
    for (const [id, running] of now) {
      if (!running || was.get(id) !== false || bouncing.has(id)) continue
      bouncing.add(id)
      setTimeout(() => bouncing.delete(id), BOUNCE_MS)
    }
  },
)

function onIconPointerdown(app, e) {
  if (e.button !== 0 || menu.value) return
  clearTimeout(holdTimer)
  const el = e.currentTarget
  holdTimer = setTimeout(() => {
    swallowNextClick()
    openMenu(app.id, el, e)
  }, HOLD_MS)
}

const cancelHold = () => clearTimeout(holdTimer)

function activate(app) {
  if (swallowClick) return (swallowClick = false)
  emit('activate', app.id)
}

function restore(win) {
  if (swallowClick) return (swallowClick = false)
  emit('restore', win.id)
}

function onZoneLeave(e) {
  if (!dockEl.value?.contains(e.relatedTarget)) hovering.value = false
}

// Vertical offset of the dock from its resting (shown) place while auto-hidden
function hiddenShift() {
  const restingTop = window.innerHeight - 6 - dockEl.value.offsetHeight
  return dockEl.value.getBoundingClientRect().top - restingTop
}

// Viewport center of an app's icon, as it will be once the dock is shown
function iconCenter(appId) {
  const el = appEls.get(appId)
  if (!el) return null
  const rect = el.getBoundingClientRect()
  return { x: rect.left + rect.width / 2, y: rect.top - hiddenShift() + rect.height / 2 }
}

// Where a w×h window lands in its tile (viewport coords, as the dock will be
// once shown): scaled to fit the tile, centred in it. The window must already
// be in `windows`.
function thumbRect(windowId, w, h) {
  const el = windowEls.get(windowId)
  if (!el) return null
  const slot = el.getBoundingClientRect()
  const scale = Math.min(props.size / w, props.size / h)
  return {
    left: slot.left + (props.size - w * scale) / 2,
    top: slot.top - hiddenShift() + (props.size - h * scale) / 2,
    width: w * scale,
    height: h * scale,
  }
}

defineExpose({ iconCenter, thumbRect })
</script>

<template>
  <div class="dock-zone" @mouseenter="hovering = true" @mouseleave="onZoneLeave" @contextmenu.prevent />
  <nav
    ref="dockEl"
    class="dock"
    :class="{ visible, 'menu-open': menu }"
    :style="{ '--dock-size': `${size}px` }"
    @mouseenter="hovering = true"
    @mouseleave="hovering = false"
    @contextmenu.prevent
  >
    <button
      v-for="app in apps"
      :key="app.id"
      :ref="setEl(appEls, app.id)"
      type="button"
      class="dock-item"
      :class="{ bouncing: bouncing.has(app.id), 'menu-owner': menu?.owner === app.id }"
      :aria-label="app.label"
      :aria-haspopup="app.menu?.length ? 'menu' : undefined"
      @click="activate(app)"
      @pointerdown="onIconPointerdown(app, $event)"
      @pointerup="cancelHold"
      @pointerleave="cancelHold"
      @contextmenu.prevent.stop="openMenu(app.id, $event.currentTarget, $event)"
    >
      <span class="tooltip">{{ app.label }}</span>
      <component :is="app.icon" class="icon" aria-hidden="true" />
      <span class="indicator" :class="{ on: app.running }" />
    </button>
    <!-- always there, as on macOS: its menu holds the Dock's own settings -->
    <span class="divider" @contextmenu.prevent.stop="openMenu(DOCK, $event.currentTarget, $event)" />
    <button
      v-for="win in windows"
      :key="win.id"
      :ref="setEl(windowEls, win.id)"
      type="button"
      class="dock-item window-tile"
      :aria-label="`${windowLabel(win)} window`"
      @click="restore(win)"
    >
      <span class="tooltip">{{ windowLabel(win) }}</span>
      <DockThumb class="thumb" :thumb="win.thumb" :size="size" />
      <!-- the app's icon badges its minimized windows -->
      <component :is="appById.get(win.app)?.icon" v-if="win.thumb" class="badge" aria-hidden="true" />
    </button>
  </nav>
  <DockMenu
    v-if="menu"
    :key="menu.id"
    :items="menuItems"
    :anchor="menu.anchor"
    :alt-key="menu.altKey"
    @close="closeMenu"
  />
</template>

<style scoped>
.dock-zone {
  position: fixed;
  left: 0;
  right: 0;
  bottom: 0;
  height: 8px;
  z-index: 999;
}

.dock {
  position: fixed;
  left: 50%;
  bottom: 6px;
  z-index: 1000;
  display: flex;
  gap: 6px;
  padding: 5px 6px 7px;
  border-radius: 18px;
  background: rgba(60, 60, 64, 0.38);
  backdrop-filter: blur(24px) saturate(1.6);
  -webkit-backdrop-filter: blur(24px) saturate(1.6);
  box-shadow:
    inset 0 0 0 0.5px rgba(255, 255, 255, 0.25),
    0 0 0 0.5px rgba(0, 0, 0, 0.4),
    0 8px 30px rgba(0, 0, 0, 0.3);
  transform: translate(-50%, calc(100% + 12px));
  transition: transform 260ms cubic-bezier(0.3, 0.7, 0.3, 1);
}

.dock.visible {
  transform: translate(-50%, 0);
}

.dock-item {
  position: relative;
  width: var(--dock-size);
  height: var(--dock-size);
  padding: 0;
  border: 0;
  background: none;
  cursor: inherit;
  outline: none;
}

.icon {
  display: block;
  width: 100%;
  height: 100%;
  filter: drop-shadow(0 2px 3px rgba(0, 0, 0, 0.35));
  transition: filter 120ms;
}

.dock-item:active .icon,
.menu-owner .icon {
  filter: drop-shadow(0 2px 3px rgba(0, 0, 0, 0.35)) brightness(0.65);
}

.bouncing .icon {
  animation: bounce 600ms ease-out;
}

@keyframes bounce {
  0%, 100% { transform: translateY(0); }
  35% { transform: translateY(-18px); }
  70% { transform: translateY(0); }
  85% { transform: translateY(-5px); }
}

/* 1px line in a wider strip, so it's easy to right-click */
.divider {
  align-self: stretch;
  width: 7px;
  margin: 4px 0;
  background: linear-gradient(rgba(255, 255, 255, 0.22), rgba(255, 255, 255, 0.22)) center / 1px 100% no-repeat;
}

.thumb {
  position: absolute;
  inset: 0;
  overflow: hidden;
  /* the copy must not inherit the button's centred text */
  text-align: left;
}

.badge {
  position: absolute;
  right: -2px;
  bottom: -2px;
  width: 24px;
  height: 24px;
  filter: drop-shadow(0 1px 2px rgba(0, 0, 0, 0.4));
}

.indicator {
  position: absolute;
  left: 50%;
  bottom: -5px;
  width: 4px;
  height: 4px;
  border-radius: 50%;
  background: rgba(255, 255, 255, 0.85);
  transform: translateX(-50%);
  opacity: 0;
  transition: opacity 200ms;
}

.indicator.on {
  opacity: 1;
}

.tooltip {
  position: absolute;
  left: 50%;
  bottom: calc(100% + 14px);
  padding: 3px 10px;
  border-radius: 6px;
  background: rgba(40, 40, 42, 0.85);
  box-shadow: inset 0 0 0 0.5px rgba(255, 255, 255, 0.2);
  color: #eee;
  font-size: 13px;
  white-space: nowrap;
  transform: translateX(-50%);
  opacity: 0;
  pointer-events: none;
  transition: opacity 120ms;
}

.dock-item:hover .tooltip {
  opacity: 1;
}

.menu-open .tooltip {
  display: none;
}
</style>
