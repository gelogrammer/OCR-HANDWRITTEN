<script setup>
import { ref } from 'vue'
import ResultsPanel from './components/ResultsPanel.vue'
import HelpModal from './components/HelpModal.vue'

const fileInput = ref(null)
const imageData = ref(null)
const imageBlob = ref(null)
const processing = ref(false)
const progressStatus = ref('')
const ocrResult = ref(null)
const showHelp = ref(false)
const dragOver = ref(false)
const errorMsg = ref(null)
const hfToken = ref('')

const OCR_SPACE_URL = 'https://api.ocr.space/parse/image'

function handleFile(file) {
  if (!file || !file.type.startsWith('image/')) return
  imageBlob.value = file
  const reader = new FileReader()
  reader.onload = e => {
    imageData.value = e.target.result
    ocrResult.value = null
    errorMsg.value = null
  }
  reader.readAsDataURL(file)
}

function onDrop(e) {
  dragOver.value = false
  handleFile(e.dataTransfer.files[0])
}

function onFileInput(e) {
  handleFile(e.target.files[0])
}

function clearImage() {
  imageData.value = null
  imageBlob.value = null
  ocrResult.value = null
  errorMsg.value = null
  if (fileInput.value) fileInput.value.value = ''
}

async function runOCR() {
  if (!imageData.value) return
  processing.value = true
  progressStatus.value = 'Sending to OCR.space...'
  ocrResult.value = null
  errorMsg.value = null

  try {
    const body = new FormData()
    body.append('base64image', imageData.value)
    body.append('language', 'eng')
    body.append('ocrengine', '2')
    body.append('scale', 'true')
    body.append('apikey', hfToken.value.trim() || 'hEllO')

    const response = await fetch(OCR_SPACE_URL, { method: 'POST', body })
    const data = await response.json()

    if (data.IsErroredOnProcessing) {
      throw new Error(data.ErrorMessage?.[0] || 'OCR processing failed.')
    }

    const text = data.ParsedResults?.[0]?.ParsedText || ''
    if (!text.trim()) throw new Error('No text detected in the image.')

    ocrResult.value = { text: text.trim() }
  } catch (err) {
    errorMsg.value = err.message
  } finally {
    processing.value = false
  }
}
</script>

<template>
  <div class="app">
    <!-- Header -->
    <header class="header">
      <div class="header-inner">
        <div class="logo">
          <svg width="28" height="28" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"/>
            <polyline points="14 2 14 8 20 8"/>
            <line x1="16" y1="13" x2="8" y2="13"/>
            <line x1="16" y1="17" x2="8" y2="17"/>
            <polyline points="10 9 9 9 8 9"/>
          </svg>
          <span>Handwriting OCR</span>
        </div>
        <div class="header-right">
          <span class="model-badge">OCR.space · Engine 2</span>
          <button class="btn-help" @click="showHelp = true">
            <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5">
              <circle cx="12" cy="12" r="10"/>
              <path d="M9.09 9a3 3 0 0 1 5.83 1c0 2-3 3-3 3"/>
              <line x1="12" y1="17" x2="12.01" y2="17"/>
            </svg>
            Guide
          </button>
        </div>
      </div>
    </header>

    <!-- Main layout -->
    <main class="main">
      <!-- Left: Upload + Controls -->
      <div class="left-panel">

        <!-- Upload zone -->
        <div
          class="upload-zone"
          :class="{ 'drag-over': dragOver, 'has-image': imageData }"
          @dragover.prevent="dragOver = true"
          @dragleave.prevent="dragOver = false"
          @drop.prevent="onDrop"
          @click="!imageData && fileInput.click()"
        >
          <input ref="fileInput" type="file" accept="image/*" @change="onFileInput" hidden />

          <div v-if="!imageData" class="upload-placeholder">
            <svg width="48" height="48" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5" opacity="0.4">
              <rect x="3" y="3" width="18" height="18" rx="2"/>
              <circle cx="8.5" cy="8.5" r="1.5"/>
              <polyline points="21 15 16 10 5 21"/>
            </svg>
            <p class="upload-title">Drop image here or click to browse</p>
            <p class="upload-hint">PNG, JPG, WEBP &mdash; single line of handwriting works best</p>
          </div>

          <template v-else>
            <img :src="imageData" class="preview-img" alt="Uploaded image" />
            <button class="btn-clear" @click.stop="clearImage" title="Remove image">
              <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5">
                <line x1="18" y1="6" x2="6" y2="18"/><line x1="6" y1="6" x2="18" y2="18"/>
              </svg>
            </button>
          </template>
        </div>

        <!-- Controls -->
        <div class="controls">
          <div class="token-group">
            <label>OCR.space API Key <span class="optional">(free — leave blank to use demo key)</span></label>
            <input
              v-model="hfToken"
              type="password"
              placeholder="K89xxxxx..."
              :disabled="processing"
              class="token-input"
            />
            <a href="https://ocr.space/ocrapi/freekey" target="_blank" class="token-link">Get a free API key →</a>
          </div>

          <button class="btn-run" :disabled="!imageData || processing" @click="runOCR">
            <svg v-if="!processing" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5">
              <polygon points="5 3 19 12 5 21 5 3"/>
            </svg>
            <span v-if="processing" class="spinner" />
            {{ processing ? 'Processing...' : 'Run OCR' }}
          </button>

          <div v-if="processing" class="progress-wrap">
            <div class="progress-bar">
              <div class="progress-fill progress-animate" />
            </div>
            <p class="progress-status">{{ progressStatus }}</p>
          </div>
        </div>

        <div v-if="errorMsg" class="error-msg">
          <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <circle cx="12" cy="12" r="10"/><line x1="12" y1="8" x2="12" y2="12"/><line x1="12" y1="16" x2="12.01" y2="16"/>
          </svg>
          {{ errorMsg }}
        </div>
      </div>

      <!-- Right: Results -->
      <div class="right-panel">
        <div v-if="!ocrResult && !processing" class="empty-state">
          <svg width="64" height="64" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.2" opacity="0.25">
            <path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"/>
            <polyline points="14 2 14 8 20 8"/>
            <line x1="16" y1="13" x2="8" y2="13"/>
            <line x1="16" y1="17" x2="8" y2="17"/>
          </svg>
          <p>Upload an image and run OCR<br>to see results here</p>
        </div>
        <ResultsPanel v-else-if="ocrResult" :result="ocrResult" />
      </div>
    </main>

    <HelpModal v-if="showHelp" @close="showHelp = false" />
  </div>
</template>

<style scoped>
.app {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
}

/* Header */
.header {
  background: white;
  border-bottom: 1px solid var(--border);
  padding: 0 24px;
  position: sticky;
  top: 0;
  z-index: 10;
}
.header-inner {
  max-width: 1200px;
  margin: 0 auto;
  height: 60px;
  display: flex;
  align-items: center;
  justify-content: space-between;
}
.logo {
  display: flex;
  align-items: center;
  gap: 10px;
  font-size: 1.1rem;
  font-weight: 700;
  color: var(--primary);
}
.header-right {
  display: flex;
  align-items: center;
  gap: 12px;
}
.model-badge {
  background: #eef2ff;
  color: var(--primary);
  font-size: 0.78rem;
  font-weight: 600;
  padding: 4px 10px;
  border-radius: 99px;
  border: 1px solid #c7d2fe;
}
.btn-help {
  display: flex;
  align-items: center;
  gap: 6px;
  background: none;
  border: 1.5px solid var(--border);
  border-radius: 8px;
  padding: 6px 14px;
  font-size: 0.875rem;
  font-weight: 500;
  color: var(--text-muted);
  transition: all 0.15s;
}
.btn-help:hover {
  border-color: var(--primary);
  color: var(--primary);
}

/* Main layout */
.main {
  flex: 1;
  max-width: 1200px;
  margin: 0 auto;
  width: 100%;
  padding: 24px;
  display: grid;
  grid-template-columns: 380px 1fr;
  gap: 24px;
  align-items: start;
}

/* Upload zone */
.upload-zone {
  background: white;
  border: 2px dashed var(--border);
  border-radius: var(--radius);
  min-height: 220px;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: all 0.2s;
  position: relative;
  overflow: hidden;
}
.upload-zone:not(.has-image):hover,
.upload-zone.drag-over {
  border-color: var(--primary);
  background: #eef2ff;
}
.upload-placeholder {
  text-align: center;
  padding: 24px;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 10px;
}
.upload-title {
  font-weight: 600;
  font-size: 0.95rem;
  color: var(--text);
}
.upload-hint {
  font-size: 0.8rem;
  color: var(--text-muted);
}
.preview-img {
  width: 100%;
  height: 100%;
  object-fit: contain;
  max-height: 300px;
  padding: 8px;
}
.btn-clear {
  position: absolute;
  top: 8px;
  right: 8px;
  background: rgba(0,0,0,0.55);
  color: white;
  border: none;
  border-radius: 50%;
  width: 26px;
  height: 26px;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: background 0.15s;
}
.btn-clear:hover { background: var(--error); }

/* Controls */
.controls {
  background: white;
  border: 1px solid var(--border);
  border-radius: var(--radius);
  padding: 18px;
  margin-top: 16px;
  display: flex;
  flex-direction: column;
  gap: 14px;
}

.token-group {
  display: flex;
  flex-direction: column;
  gap: 5px;
}
.token-group label {
  font-size: 0.78rem;
  font-weight: 600;
  color: var(--text-muted);
  text-transform: uppercase;
  letter-spacing: 0.04em;
}
.optional {
  font-weight: 400;
  text-transform: none;
  letter-spacing: 0;
}
.token-input {
  border: 1.5px solid var(--border);
  border-radius: 7px;
  padding: 7px 10px;
  font-size: 0.875rem;
  color: var(--text);
  background: white;
  outline: none;
  transition: border-color 0.15s;
  width: 100%;
  box-sizing: border-box;
}
.token-input:focus { border-color: var(--primary); }
.token-input:disabled { opacity: 0.5; }
.token-link {
  font-size: 0.78rem;
  color: var(--primary);
  text-decoration: none;
}
.token-link:hover { text-decoration: underline; }

.btn-run {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  background: var(--primary);
  color: white;
  border: none;
  border-radius: 8px;
  padding: 11px;
  font-size: 0.95rem;
  font-weight: 600;
  transition: background 0.15s, opacity 0.15s;
}
.btn-run:hover:not(:disabled) { background: var(--primary-dark); }
.btn-run:disabled { opacity: 0.5; cursor: not-allowed; }

.spinner {
  width: 16px;
  height: 16px;
  border: 2px solid rgba(255,255,255,0.3);
  border-top-color: white;
  border-radius: 50%;
  animation: spin 0.7s linear infinite;
}
@keyframes spin { to { transform: rotate(360deg); } }

.progress-wrap { display: flex; flex-direction: column; gap: 5px; }
.progress-bar {
  height: 6px;
  background: var(--border);
  border-radius: 99px;
  overflow: hidden;
}
.progress-fill {
  height: 100%;
  background: var(--primary);
  border-radius: 99px;
}
.progress-animate {
  width: 40%;
  animation: indeterminate 1.2s ease-in-out infinite;
}
@keyframes indeterminate {
  0%   { transform: translateX(-100%); }
  100% { transform: translateX(350%); }
}
.progress-status {
  font-size: 0.78rem;
  color: var(--text-muted);
  text-transform: capitalize;
}

.error-msg {
  display: flex;
  align-items: center;
  gap: 8px;
  background: #fef2f2;
  border: 1px solid #fecaca;
  color: var(--error);
  border-radius: 8px;
  padding: 10px 14px;
  font-size: 0.875rem;
  margin-top: 8px;
}

/* Right panel */
.right-panel {
  min-height: 400px;
}
.empty-state {
  background: white;
  border: 1px solid var(--border);
  border-radius: var(--radius);
  min-height: 400px;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 14px;
  color: var(--text-muted);
  text-align: center;
  line-height: 1.6;
  font-size: 0.95rem;
}
</style>
