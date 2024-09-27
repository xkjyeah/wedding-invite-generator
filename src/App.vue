<script setup lang="ts">
import { VNodeRef, ref, watch } from 'vue'
import Error from './components/Error.vue'
import { sortBy, uniq, debounce } from 'lodash'

const fileInput = ref<(HTMLInputElement & VNodeRef) | null>(null)
const canvasInput = ref<(HTMLCanvasElement & VNodeRef) | null>(null)


const currentUrl = new URL(window.location.href)
function loadStringParam(url: URL, key: string): string | null {
  const value = url.searchParams.get(key)
  if (!value) { return null }

  if (typeof value !== 'string') {
    return null
  } else {
    return value
  }
}
function loadObjectParam(url: URL, key: string): any {
  const value = url.searchParams.get(key)
  if (!value) { return null }

  const parsedValue = JSON.parse(value)

  return parsedValue
}

const name = ref<string>(loadStringParam(currentUrl, 'name') || "Tan Ah Kow")
const fontLoadingCode = ref<string>(loadStringParam(currentUrl, 'fontLoadingCode') || "")
const fontFamily = ref<string>(loadStringParam(currentUrl, 'fontFamily') || "sans-serif")
const isItalic = ref<string>(loadStringParam(currentUrl, 'isItalic') || "")
const isBold = ref<string>(loadStringParam(currentUrl, 'isBold') || "")
const fontColor = ref<string>(loadStringParam(currentUrl, 'fontColor') || "#000000")
const fontSize = ref<string>(loadStringParam(currentUrl, 'fontSize') || '14')
const coordinates = ref<[number, number]>(loadObjectParam(currentUrl, 'coordinates') || [0.5, 0.5])
const imageSize = ref<{ width: number, height: number }>({ width: 500, height: 500 })

const rawImageDataURL = ref<string>("")
const imageDataURL = ref<string>("")

const availableFontFaces = ref<string[]>([])

const errors = ref<Record<string, string>>({})

async function updatePreview(input: Event) {
  const file = (input.currentTarget as HTMLInputElement).files?.[0]

  if (!file) {
    return
  }

  const dataUrlPromise = new Promise<string>((resolve) => {
    const reader = new FileReader();

    reader.addEventListener('load', (e) => {
      const result = e.target?.result
      if (result) {
        resolve(result as string)
      }
    });

    reader.readAsDataURL(file);
  })

  rawImageDataURL.value = await dataUrlPromise
  generateInvitationCard()
}

const generateInvitationCard = debounce(async function (): Promise<void> {
  const canvasElement = canvasInput.value as HTMLCanvasElement

  // Generate the image we want to draw
  const imageEle = new Image()
  const sizePromise = new Promise<{ width: number, height: number }>((resolve) => {
    imageEle.addEventListener('load', () => {
      resolve({ width: imageEle.width, height: imageEle.height })
    })
  })
  imageEle.src = rawImageDataURL.value

  imageSize.value = await sizePromise

  // Allow an update!
  await new Promise(resolve => setTimeout(resolve, 0))

  const context = canvasElement.getContext("2d")!

  context.drawImage(imageEle, 0, 0)

  const lx = imageSize.value.width * coordinates.value[0]
  const ly = imageSize.value.height * coordinates.value[1]

  const effectiveSize = (parseFloat(fontSize.value) || 14) / 600 * imageSize.value.height

  context.font = `${isItalic.value ? 'italic' : ''} ${isBold.value ? 'bold' : ''} ${effectiveSize}px "${fontFamily.value}"`

  context.textAlign = 'center'
  context.fillStyle = fontColor.value
  context.fillText(name.value, lx, ly)

  // Update the URL!
  imageDataURL.value = canvasElement.toDataURL()
}, 1000)

const saveSettings = debounce(function () {
  const currentUrl = new URL(window.location.href)
  const settingsToSave = {
    fontFamily: fontFamily.value,
    isBold: isBold.value,
    isItalic: isItalic.value,
    fontLoadingCode: fontLoadingCode.value,
    fontSize: fontSize.value,
    fontColor: fontColor.value,
    name: name.value,
    coordinates: coordinates.value,
  }

  for (let [k, v] of Object.entries(settingsToSave)) {
    if (typeof v === 'string') {
      currentUrl.searchParams.set(k, v)
    } else {
      currentUrl.searchParams.set(k, JSON.stringify(v))
    }
  }

  window.history.replaceState(null, '', currentUrl.toString())
}, 1000)

watch([fontFamily, isBold, isItalic, fontSize, fontColor, name, rawImageDataURL, coordinates], () => {
  generateInvitationCard()
  saveSettings()
})

const loadedFonts: Map<string, boolean> = new Map

async function handleFontLoadingCode() {
  const code = fontLoadingCode.value
  const matches = code.match(/<link href="(.*)"/)
  const url = matches?.[1]

  if (!url) {
    errors.value.fontLoadingCode = "Could not find the stylesheet URL in font loading code"
    return
  }

  if (loadedFonts.has(url)) {
    return
  }

  loadedFonts.set(url, true)

  // Insert the stylesheet into the code...
  const linkElem = document.createElement('LINK') as HTMLLinkElement
  linkElem.href = url
  linkElem.rel = 'stylesheet'
  document.head.appendChild(linkElem)

  // There doesn't seem to be a reliable way to determine if the fonts have been fetched??
  // Never mind, poll every .5s until there hasn't been any change to size
  const LIVES = 5
  let lastSize = -1
  let livesRemaining = LIVES
  for (; livesRemaining > 0;) {
    const sz = document.fonts.size
    if (sz !== lastSize) {
      livesRemaining = LIVES
      lastSize = sz
    } else {
      livesRemaining -= 1
    }
    await new Promise((resolve) => setTimeout(resolve, 500))
  }

  // Sync document fonts
  const fonts = Array.from(document.fonts.keys()).map(s => s.family)
  console.log(sortBy(uniq(fonts)))
  availableFontFaces.value = sortBy(uniq(fonts))
}

if (fontLoadingCode.value) {
  handleFontLoadingCode()
}

document.fonts.addEventListener('loadingdone', generateInvitationCard)

function handleUpdatePosition(evt: MouseEvent) {
  const img = evt.target as HTMLImageElement

  coordinates.value = [evt.offsetX / img.clientWidth, evt.offsetY / img.clientHeight]
}

</script>

<template>
  <div class="root">
    <canvas ref="canvasInput" style="display: none" :width="imageSize.width" :height="imageSize.height" />
    <p>
      Image file:
      <input type="file" ref="fileInput" @change="updatePreview" />
    </p>
    <p>
      Font URL:
      <a href="https://fonts.google.com/" target="font_window">Find some fonts on Google</a>
      <textarea v-model="fontLoadingCode" @change="handleFontLoadingCode"
        placeholder="Paste the '<link rel=...' code from Google Fonts here"></textarea>
      <Error :message="errors.fontLoadingCode" />
    </p>
    <p>
      Font face:
      <select @input="fontFamily = ($event.target as any).value">
        <option></option>
        <option v-for="ff in availableFontFaces" :value="ff" :selected="ff == fontFamily">{{ ff }}</option>
      </select>

      <label>
        <input type="checkbox" :checked="!!isBold" @input="isBold = ($event.target as any).checked ? '1' : ''" />
        Bold
      </label>

      <label>
        <input type="checkbox" :checked="!!isItalic" @input="isItalic = ($event.target as any).checked ? '1' : ''" />
        Italic
      </label>
    </p>
    <p>
      Font size:
      <input type="text" :value="fontSize" @input="fontSize = ($event.currentTarget as any).value">
    </p>
    <p>
      Color:
      <input type="text" v-model="fontColor">
    </p>
    <hr />
    <p>
      Invitee's Name:
      <input type="text" v-model="name">
    </p>
    <hr />
    <p>
      Preview:<br />
      <img v-if="imageDataURL" :src="imageDataURL"
        style="width: auto; height: auto; max-width: 600px; max-height: 600px" @click="handleUpdatePosition" />
    </p>
  </div>
</template>

<style scoped>
.root {
  max-width: 600px;
  margin: auto;
}

input[type="text"],
textarea {
  display: block;
  width: 100%;
}
</style>
