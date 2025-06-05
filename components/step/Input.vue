<script setup lang="ts">
import JSZip from 'jszip'
import { ref, onBeforeMount, onBeforeUnmount } from 'vue'

const emit = defineEmits(['update:files', 'update:outputName', 'next'])

const input = ref<HTMLInputElement | null>(null)
const outputName = ref('output')
const selectedText = ref('No files selected')
const error = ref<string | undefined>()

function isPNG(file: File | { name: string }) {
  return file.name.toLowerCase().endsWith('.png')
}
function isZIP(file: File) {
  return file.name.toLowerCase().endsWith('.zip')
}

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
    error.value = 'No PNG images found!'
    return
  }

  error.value = undefined
  selectedText.value = `${images.length} image(s)`
  emit('update:files', files)
}

const handleFileInput = (e: Event) => {
  const target = e.target as HTMLInputElement
  if (target.files) {
    handleFiles([...target.files])
  }
}
</script>

<template>
  <div class="col gap2 centerChildren">
    <!-- Вибір файлів -->
    <button @click="input?.click()">Select Files</button>
    <input
      ref="input"
      type="file"
      accept=".zip,.png"
      multiple
      @change="handleFileInput"
      style="display: none"
    />

    <!-- Вивід кількості файлів -->
    <span>{{ selectedText }}</span>

    <!-- Поле для шляху (output file name) -->
    <input
      type="text"
      v-model="outputName"
      placeholder="Output name"
      @input="emit('update:outputName', outputName)"
    />

    <!-- Помилка, якщо є -->
    <span v-if="error" class="error">{{ error }}</span>

    <!-- Кнопка Next завжди активна -->
    <button @click="emit('next')">Next</button>
  </div>
</template>

<style scoped>
.error {
  color: red;
}
.col {
  display: flex;
  flex-direction: column;
  gap: 1rem;
  align-items: center;
}
</style>
