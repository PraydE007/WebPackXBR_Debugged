<script setup lang="ts">
import JSZip from 'jszip'
import { ref, onBeforeMount, onBeforeUnmount } from 'vue'

// Приймаємо пропси і еміти
defineProps(['files'])
const emit = defineEmits(['update:files', 'update:outputName', 'next'])

// Стан
const activeSelect = ref(false)
const selectedText = ref('No files selected')
const hasFiles = ref(false)
const error = ref<string | undefined>()

const input = ref<HTMLInputElement | null>(null)

// Перевірка чи файл — PNG
function isPNG(file: File | { name: string }): boolean {
  return file.name.toLowerCase().endsWith('.png')
}

// Перевірка чи файл — ZIP
function isZIP(file: File): boolean {
  return file.name.toLowerCase().endsWith('.zip')
}

// Обробка вибраних файлів
async function handleFiles(fileList: File[]) {
  let zipFile
  for (const file of fileList) {
    if (isZIP(file)) {
      zipFile = file
      break
    }
  }

  let files: any[]
  if (zipFile) {
    const zip = await new JSZip().loadAsync(zipFile)
    files = Object.values(zip.files).map(img => ({
      name: img.name,
      arrayBuffer: async () => await img.async('ArrayBuffer'),
    }))
  } else {
    files = fileList
  }

  const images = files.filter(isPNG)

  if (images.length < 1) {
    error.value = 'No files found in input! Only .zips, .png, and clipboard contents are supported'
    console.error('No .png files found.')
    return
  }

  error.value = undefined
  hasFiles.value = true
  selectedText.value = `${images.length} image(s)`

  emit('update:files', files)

  let outputName = 'images._'
  if (zipFile) outputName = zipFile.name
  else if (images.length === 1) outputName = images[0].name

  emit('update:outputName', outputName.split('.').slice(0, -1).join('.'))
}

// Обробники подій
const handleFileInput = (e: Event) => {
  const target = e.target as HTMLInputElement
  if (target.files) {
    handleFiles([...target.files])
  }
}
const handleFileDrop = (e: DragEvent) => {
  e.preventDefault()
  if (e.dataTransfer?.files) {
    handleFiles([...e.dataTransfer.files])
  }
}
const handleFilePaste = (e: ClipboardEvent) => {
  const files = [...e.clipboardData!.files]
  if (files.length > 0) handleFiles(files)
}
const dragStart = () => (activeSelect.value = true)
const dragLeave = (e: DragEvent) => {
  if (!e.relatedTarget || (e.relatedTarget as HTMLElement).nodeName === 'BODY') {
    activeSelect.value = false
  }
}
const dragEnd = () => (activeSelect.value = false)
const prevent = (e: Event) => e.preventDefault()

// Реєстрація глобальних подій
onBeforeMount(() => {
  document.addEventListener('drop', handleFileDrop)
  document.addEventListener('paste', handleFilePaste)
  document.addEventListener('dragover', prevent)
  document.addEventListener('dragenter', dragStart)
  document.addEventListener('dragend', dragEnd)
  document.body.addEventListener('dragleave', dragLeave)
})
onBeforeUnmount(() => {
  document.removeEventListener('drop', handleFileDrop)
  document.removeEventListener('paste', handleFilePaste)
  document.removeEventListener('dragover', prevent)
  document.removeEventListener('dragenter', dragStart)
  document.removeEventListener('dragend', dragEnd)
  document.body.removeEventListener('dragleave', dragLeave)
})
</script>

<template>
  <div class="col gap2 centerChildren">
    <!-- Кнопка (логотип) для відкриття вибору файлів -->
    <StaticLogo class="interactive" :class="{ showBox: hasFiles, activeSelect }" @click="input?.click()" />

    <!-- Інпут файлу, прихований -->
    <input
      ref="input"
      type="file"
      accept=".zip,.png"
      multiple
      @change="handleFileInput"
    />

    <!-- Повідомлення про помилки -->
    <span v-if="error" class="error">{{ error }}</span>

    <!-- Текст про вибрані файли -->
    <span>{{ selectedText }}</span>

    <!-- Кнопка Next -->
    <button :class="{ hidden: !hasFiles }" @click="$emit('next')">
      Next
    </button>
  </div>

  <StaticInfo />
</template>

<style scoped lang="scss">
input[type='file'] {
  display: none;
}
.error {
  color: red;
}
.hidden {
  display: none;
}
</style>
