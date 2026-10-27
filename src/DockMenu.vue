<script setup>
import { ref, computed, watch, nextTick, onMounted, onBeforeUnmount } from 'vue'

const props = defineProps({
  // [{ label, action?, disabled?, mark?: 'check' | 'diamond', alt?: { label, action } } | { separator: true }]
  // `alt` replaces the item while Option is held, as in macOS menus.
  items: { type: Array, required: true },
  // Viewport rect of the icon the menu belongs to; the menu opens centred above it
  anchor: { type: Object, required: true },
  // Option held when the menu was opened
  altKey: { type: Boolean, default: false },
})

const emit = defineEmits(['close'])

const GAP = 10 // between the icon's top and the menu
const EDGE = 6 // kept clear of the viewport's edges
const FLASH_MS = 160

const menuEl = ref()
const active = ref(-1)
const alt = ref(props.altKey)
const flashing = ref(false)
const left = ref(0)

const shown = computed(() => props.items.map((item) => (alt.value && item.alt ? { ...item, ...item.alt } : item)))
const selectable = (item) => item && !item.separator && !item.disabled

const position = computed(() => ({
  left: `${left.value}px`,
  bottom: `${window.innerHeight - props.anchor.top + GAP}px`,
}))

function place() {
  const width = menuEl.value.offsetWidth
  const center = props.anchor.left + props.anchor.width / 2
  left.value = Math.min(Math.max(center - width / 2, EDGE), window.innerWidth - width - EDGE)
}

watch(shown, () => nextTick(place))

// macOS blinks the chosen item before the menu goes away
function choose(index) {
  const item = shown.value[index]
  if (!selectable(item) || flashing.value) return
  active.value = index
  flashing.value = true
  setTimeout(() => {
    emit('close')
    item.action?.()
  }, FLASH_MS)
}

function step(dir) {
  const n = shown.value.length
  let i = active.value
  for (let k = 0; k < n; k++) {
    i = i < 0 ? (dir > 0 ? 0 : n - 1) : (i + dir + n) % n
    if (selectable(shown.value[i])) return (active.value = i)
  }
}

// While open the menu takes the keyboard, so keys don't reach the page
function onKeydown(e) {
  if (e.key === 'Alt') alt.value = true
  e.preventDefault()
  e.stopPropagation()
  if (flashing.value) return
  if (e.key === 'Escape') emit('close')
  else if (e.key === 'ArrowDown') step(1)
  else if (e.key === 'ArrowUp') step(-1)
  else if (e.key === 'Enter' || e.key === ' ') {
    if (active.value >= 0) choose(active.value)
    else emit('close')
  }
}

function onKeyup(e) {
  if (e.key === 'Alt') alt.value = false
}

// A press outside closes the menu; the dock decides what the press itself does
function onPointerdown(e) {
  if (!menuEl.value.contains(e.target)) emit('close', e)
}

const onBlur = () => emit('close')

onMounted(() => {
  place()
  document.addEventListener('keydown', onKeydown, true)
  document.addEventListener('keyup', onKeyup, true)
  document.addEventListener('pointerdown', onPointerdown, true)
  window.addEventListener('blur', onBlur)
  window.addEventListener('resize', onBlur)
})

onBeforeUnmount(() => {
  document.removeEventListener('keydown', onKeydown, true)
  document.removeEventListener('keyup', onKeyup, true)
  document.removeEventListener('pointerdown', onPointerdown, true)
  window.removeEventListener('blur', onBlur)
  window.removeEventListener('resize', onBlur)
})
</script>

<template>
  <Teleport to="body">
    <div
      ref="menuEl"
      class="dock-menu"
      role="menu"
      :style="position"
      @contextmenu.prevent
      @mouseleave="flashing || (active = -1)"
    >
      <template v-for="(item, i) in shown" :key="i">
        <div v-if="item.separator" class="separator" role="separator" />
        <div
          v-else
          class="item"
          role="menuitem"
          :aria-disabled="item.disabled || undefined"
          :class="{ active: active === i, disabled: item.disabled, flashing: flashing && active === i }"
          @mouseenter="flashing || (active = selectable(item) ? i : -1)"
          @mouseup="choose(i)"
        >
          <span class="mark">{{ item.mark === 'check' ? '✓' : item.mark === 'diamond' ? '◆' : '' }}</span>
          {{ item.label }}
        </div>
      </template>
    </div>
  </Teleport>
</template>

<style scoped>
.dock-menu {
  position: fixed;
  z-index: 1001;
  min-width: 120px;
  padding: 5px;
  border-radius: 12px;
  background: rgba(36, 36, 38, 0.72);
  backdrop-filter: blur(30px) saturate(1.8);
  -webkit-backdrop-filter: blur(30px) saturate(1.8);
  box-shadow:
    inset 0 0 0 0.5px rgba(255, 255, 255, 0.2),
    0 0 0 0.5px rgba(0, 0, 0, 0.5),
    0 10px 30px rgba(0, 0, 0, 0.35);
  color: rgba(255, 255, 255, 0.88);
  font: 13px/1 var(--dock-font, -apple-system, BlinkMacSystemFont, 'SF Pro Text', 'Helvetica Neue', sans-serif);
  white-space: nowrap;
  user-select: none;
  -webkit-user-select: none;
  cursor: default;
}

.item {
  display: flex;
  align-items: center;
  height: 24px;
  padding: 0 14px 0 0;
  border-radius: 6px;
}

.mark {
  width: 22px;
  flex: none;
  text-align: center;
  font-size: 11px;
}

.item.active {
  background: var(--dock-menu-highlight, #0a68d9);
  color: #fff;
}

.item.disabled {
  color: rgba(255, 255, 255, 0.28);
}

.item.flashing {
  animation: flash 160ms steps(1) both;
}

@keyframes flash {
  0% { background: transparent; color: rgba(255, 255, 255, 0.88); }
  50%, 100% { background: var(--dock-menu-highlight, #0a68d9); color: #fff; }
}

.separator {
  height: 1px;
  margin: 5px 10px;
  background: rgba(255, 255, 255, 0.12);
}
</style>
