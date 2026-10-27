<script setup>
import { ref, watch, onMounted } from 'vue'

const props = defineProps({
  // { node, width, height, place? } — `node` is cloned and scaled into the tile,
  // `place(copy)` runs once the copy is in the document (e.g. to restore scroll)
  thumb: { type: Object, default: null },
  size: { type: Number, required: true },
})

const host = ref()

function render(shot) {
  host.value.replaceChildren()
  if (!shot) return
  const node = shot.node.cloneNode(true)
  const { width: w, height: h } = shot
  const scale = Math.min(props.size / w, props.size / h)
  Object.assign(node.style, {
    position: 'absolute',
    left: `${(props.size - w * scale) / 2}px`,
    top: `${(props.size - h * scale) / 2}px`,
    width: `${w}px`,
    height: `${h}px`,
    transform: `scale(${scale})`,
    transformOrigin: '0 0',
    transition: 'none',
    visibility: 'visible',
    pointerEvents: 'none',
  })
  host.value.appendChild(node)
  shot.place?.(node)
}

onMounted(() => render(props.thumb))
watch(() => props.thumb, render, { flush: 'post' })
</script>

<template>
  <span ref="host" />
</template>
