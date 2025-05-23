<!-- eslint-disable ts/ban-ts-comment -->
<!-- eslint-disable ts/no-unused-expressions -->
<script setup lang='ts'>
import type { CanvasOptions, Point, Props, Signature, WaterMarkObj } from '../types'
import { nanoid } from 'nanoid'
import SignaturePad from 'signature_pad'
import { onBeforeUnmount, onMounted, ref, watch, watchEffect } from 'vue'

defineOptions({
  name: 'VueSignaturePad',
  inheritAttrs: false,
})

const props = withDefaults(defineProps<Props>(), {
  throttle: 16,
  minWidth: 5,
  maxWidth: 5,
  height: '100%',
  width: '100%',
  options: {
    backgroundColor: 'rgb(255,255,255)',
    penColor: 'rgb(0, 0, 0)',
  },
  disabled: false,
  clearOnResize: false,
  defaultUrl: '',
})

const emits = defineEmits<{
  (e: 'beginStroke', val: string): void
  (e: 'endStroke', val: string): void
  (e: 'beforeUpdateStroke', val: string): void
  (e: 'afterUpdateStroke', val: string): void
}>()

const canvasOptions = ref<CanvasOptions>({
  signaturePad: {} as Signature,
  dotSize: 0.5,
  minWidth: 2,
  maxWidth: 2,
  throttle: 16,
  backgroundColor: props.options.backgroundColor,
  penColor: props.options.penColor,
  canvasUuid: `canvas${nanoid()}`,
})

function isCanvasEmpty(): boolean {
  return canvasOptions.value.signaturePad.isEmpty()
}

function saveSignature(format?: string): string {
  return format ? canvasOptions.value.signaturePad.toDataURL(format) : canvasOptions.value.signaturePad.toDataURL()
}

function clearCanvas() {
  return canvasOptions.value.signaturePad.clear()
}

const canvas = document.getElementById(canvasOptions.value.canvasUuid) as HTMLCanvasElement

function fromDataURL(url: string) {
  return canvasOptions.value.signaturePad.fromDataURL(url)
}

function undo() {
  const canvasData: Point[][] = canvasOptions.value.signaturePad.toData()
  if (canvasData.length) {
    canvasData.pop()
    canvasOptions.value.signaturePad.fromData(canvasData)
  }
};

function addWaterMark(data: WaterMarkObj) {
  if (!(Object.prototype.toString.call(data) === '[object Object]')) {
    throw new Error(`Expected Object, got ${typeof data}.`)
  }
  else {
    const textData = {
      text: data.text || '',
      x: data.x || 20,
      y: data.y || 20,
      sx: data.sx || 40,
      sy: data.sy || 40,
    }
    const VCanvas = document.getElementById(canvasOptions.value.canvasUuid) as HTMLCanvasElement
    const ctx = VCanvas.getContext('2d')
    if (ctx) {
      ctx.font = data.font || '20px sans-serif'
      ctx.fillStyle = data.fillStyle || '#333'
      ctx.strokeStyle = data.strokeStyle || '#333'
      if (data.style === 'all') {
        ctx.fillText(textData.text, textData.x, textData.y)

        ctx.strokeText(textData.text, textData.sx, textData.sy)
      }
      else if (data.style === 'stroke') {
        ctx.strokeText(textData.text, textData.sx, textData.sy)
      }
      else {
        ctx.fillText(textData.text, textData.x, textData.y)
      }
    }
    canvasOptions.value.signaturePad._isEmpty = false
  }
}

function resizeCanvas(c: HTMLCanvasElement) {
  let url
  if (!isCanvasEmpty()) {
    url = saveSignature()
  }
  const ratio = Math.max(window.devicePixelRatio || 1, 1)
  const reg = /px/
  c.width = props.width && reg.test(props?.width) ? Number.parseInt(props?.width.replace(/px/g, '')) * ratio : c.offsetWidth * ratio
  c.height = props.height && reg.test(props?.height) ? Number.parseInt(props?.height.replace(/px/g, '')) * ratio : c.offsetHeight * ratio
  c.getContext('2d')?.scale(ratio, ratio)
  clearCanvas()
  !props.clearOnResize && url !== undefined && fromDataURL(url)
  props.waterMark && Object.keys(props.waterMark).length && addWaterMark(props.waterMark)
}

function handleBeginStroke() {
  return canvasOptions.value.signaturePad.addEventListener('beginStroke', () => {
    emits('beginStroke', 'Signature started')
  })
}

function handleEndStroke() {
  return canvasOptions.value.signaturePad.addEventListener('endStroke', () => {
    emits('endStroke', 'Signature ended')
  })
}

function handleBeforeUpdateStroke() {
  return canvasOptions.value.signaturePad.addEventListener('beforeUpdateStroke', () => {
    emits('beforeUpdateStroke', 'Signature before update')
  })
}

function handleAfterUpdateStroke() {
  return canvasOptions.value.signaturePad.addEventListener('afterUpdateStroke', () => {
    emits('afterUpdateStroke', 'Signature after update')
  })
}
function draw() {
  const canvas = document.getElementById(canvasOptions.value.canvasUuid) as HTMLCanvasElement
  if (!canvas) {
    throw new Error(`Canvas element with ID ${canvasOptions.value.canvasUuid} not found.`)
  };
  // @ts-expect-error
  canvasOptions.value.signaturePad = new SignaturePad(canvas, canvasOptions.value)
  handleBeginStroke()
  handleEndStroke()
  handleBeforeUpdateStroke()
  handleAfterUpdateStroke()
  const resizeHandler = () => {
    resizeCanvas(canvas)
  }
  window.addEventListener('resize', resizeHandler)
  resizeCanvas(canvas)
  if (props.defaultUrl !== '') {
    fromDataURL(props.defaultUrl)
  }
  if (props.disabled) {
    canvasOptions.value.signaturePad.off()
  }
  else {
    canvasOptions.value.signaturePad.on()
  }
}

watchEffect(() => {
  // Update penColor
  canvasOptions.value.penColor = props.options.penColor
  canvasOptions.value.signaturePad.penColor = props.options.penColor
  // Update backgroundColor
  canvasOptions.value.backgroundColor = props.options.backgroundColor
  canvasOptions.value.signaturePad.backgroundColor = props.options.backgroundColor
})

watch(() => props.minWidth, (newVal) => {
  if (newVal) {
    canvasOptions.value.minWidth = newVal
    canvasOptions.value.signaturePad.minWidth = newVal
  }
}, { immediate: true })

watch(() => props.maxWidth, (newVal) => {
  if (newVal) {
    canvasOptions.value.maxWidth = newVal
    canvasOptions.value.signaturePad.maxWidth = newVal
  }
}, { immediate: true })

watch(
  () => props.disabled,
  (val) => {
    if (val) {
      return canvasOptions.value.signaturePad.off()
    }
    else {
      return canvasOptions.value.signaturePad.on()
    }
  },
)
defineExpose({
  saveSignature,
  clearCanvas,
  addWaterMark,
  isCanvasEmpty,
  fromDataURL,
  undo,
})

onMounted(() => {
  draw()
})

onBeforeUnmount(() => {
  const resizeHandler = () => {
    resizeCanvas(canvas)
  }
  window.removeEventListener('resize', resizeHandler)
})
</script>

<template>
  <div :style="{ width: props.width, height: props.height }" @touchmove.prevent>
    <canvas
      :id="canvasOptions.canvasUuid" style="width: 100%; height: 100%;"
      :data-uid="canvasOptions.canvasUuid"
    />
  </div>
</template>
