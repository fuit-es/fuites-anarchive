<template>
  <v-image
    v-if="raster"
    ref="imgNode"
    :config="textConfig"
    :key="key" />
</template>

<script setup>
// ======================================================================= Setup
const props = defineProps({
  parentConfig: {
    type: Object,
    required: true
  },
  options: {
    type: Object,
    required: false,
    default: () => ({})
  },
  text: {
    type: String,
    required: true
  },
  hidden: {
    type: Boolean,
    required: false,
    default: false
  }
})

const emit = defineEmits(['loaded'])

// ======================================================================== Data
const key = ref(0)
const raster = ref(false)
const imgNode = ref(null)
// Scene pixels added around each glyph. The cached picture stays sharp;
// only the hit mask is blurred out into this margin.
const HIT_MARGIN = 4

// ==================================================================== Computed
const textConfig = computed(() => ({
  thingie_id: props.parentConfig.thingie_id,
  width: props.parentConfig.width,
  height: props.parentConfig.height,
  image: raster.value,
  visible: !props.hidden,
  ...props.options
}))

// ==================================================================== Watchers
watch(() => props.options, () => { key.value++ }, { deep: true })
watch([
  () => props.text,
  () => props.parentConfig.width,
  () => props.parentConfig.height
], () => { rasterizeText() })
watch(key, () => { nextTick(drawTextHitArea) })

// ======================================================================= Hooks
onMounted(() => { rasterizeText() })

// ===================================================================== Methods
/**
 * @method drawTextHitArea
 * @desc Cache the sharp text, then blur only the hit mask so clicks near a glyph still land
 */

const drawTextHitArea = () => {
  const node = imgNode.value?.getNode()
  if (!node) { return }
  node.cache({ offset: HIT_MARGIN })
  if (!node.isCached()) { return }
  node.drawHitFromCache()
  widenCachedHitArea(node, HIT_MARGIN)
  node.getLayer()?.batchDraw()
}

/**
 * @method widenCachedHitArea
 * @desc Box-blur the hit mask. Any pixel the blur reaches becomes a solid hit
 */

const widenCachedHitArea = (node, radius) => {
  const hitCanvas = node._getCanvasCache()?.hit
  const color = hexToRgb(node.colorKey)
  if (!hitCanvas || !color || radius < 1) { return }
  const context = hitCanvas.getContext()
  const width = hitCanvas.getWidth()
  const height = hitCanvas.getHeight()
  let imageData
  try {
    imageData = context.getImageData(0, 0, width, height)
  } catch {
    return
  }
  const data = imageData.data
  const alpha = boxBlurAlpha(data, width, height, radius)
  for (let i = 0, pixel = 0; i < alpha.length; i++, pixel += 4) {
    if (alpha[i] > 0) {
      data[pixel] = color.r
      data[pixel + 1] = color.g
      data[pixel + 2] = color.b
      data[pixel + 3] = 255
    } else {
      data[pixel + 3] = 0
    }
  }
  context.putImageData(imageData, 0, 0)
}

/**
 * @method boxBlurAlpha
 * @desc Separable box blur of the alpha channel. Radius is in hit-canvas pixels
 */

const boxBlurAlpha = (data, width, height, radius) => {
  const count = width * height
  const alpha = new Float32Array(count)
  for (let i = 0; i < count; i++) { alpha[i] = data[i * 4 + 3] }
  const tmp = new Float32Array(count)
  const windowSize = radius * 2 + 1
  for (let y = 0; y < height; y++) {
    const row = y * width
    let sum = 0
    for (let i = -radius; i <= radius; i++) {
      sum += alpha[row + Math.min(width - 1, Math.max(0, i))]
    }
    for (let x = 0; x < width; x++) {
      tmp[row + x] = sum / windowSize
      sum -= alpha[row + Math.min(width - 1, Math.max(0, x - radius))]
      sum += alpha[row + Math.min(width - 1, Math.max(0, x + radius + 1))]
    }
  }
  for (let x = 0; x < width; x++) {
    let sum = 0
    for (let i = -radius; i <= radius; i++) {
      sum += tmp[Math.min(height - 1, Math.max(0, i)) * width + x]
    }
    for (let y = 0; y < height; y++) {
      alpha[y * width + x] = sum / windowSize
      sum -= tmp[Math.min(height - 1, Math.max(0, y - radius)) * width + x]
      sum += tmp[Math.min(height - 1, Math.max(0, y + radius + 1)) * width + x]
    }
  }
  return alpha
}

/**
 * @method hexToRgb
 */

const hexToRgb = hex => {
  const value = parseInt(String(hex).replace('#', ''), 16)
  if (Number.isNaN(value)) { return null }
  return {
    r: (value >> 16) & 255,
    g: (value >> 8) & 255,
    b: value & 255
  }
}

const rasterizeText = () => {
  if (process.client) {
    const div = document.createElement('div')
    div.innerHTML = props.text
    const canvas = useGetHiPPICanvas({ width: textConfig.value.width, height: textConfig.value.height })
    const render = useRenderTextElementToCanvas(canvas, div, textConfig.value)
    raster.value = render
    key.value++
    div.remove()
    emit('loaded', true)
    // document.body.appendChild(render)
    // render.style.position = 'absolute'
    // render.style.left = '100px'
    // render.style.top = '100px'
  }
}
</script>
