<script setup>
import { ref } from 'vue'

const props = defineProps({
  result: { type: Object, required: true }
})

const copied = ref(false)

async function copyText() {
  await navigator.clipboard.writeText(props.result.text)
  copied.value = true
  setTimeout(() => { copied.value = false }, 2000)
}
</script>

<template>
  <div class="results">

    <!-- Extracted text -->
    <div class="section">
      <div class="section-header">
        <h3>Extracted Text</h3>
        <button class="btn-copy" @click="copyText">
          <svg v-if="!copied" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <rect x="9" y="9" width="13" height="13" rx="2"/><path d="M5 15H4a2 2 0 0 1-2-2V4a2 2 0 0 1 2-2h9a2 2 0 0 1 2 2v1"/>
          </svg>
          <svg v-else width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5">
            <polyline points="20 6 9 17 4 12"/>
          </svg>
          {{ copied ? 'Copied!' : 'Copy' }}
        </button>
      </div>
      <pre class="raw-text">{{ result.text || '(no text detected)' }}</pre>
    </div>

    <!-- TrOCR tip -->
    <div class="tip">
      <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
        <circle cx="12" cy="12" r="10"/><line x1="12" y1="8" x2="12" y2="12"/><line x1="12" y1="16" x2="12.01" y2="16"/>
      </svg>
      <span>TrOCR is optimised for <strong>single lines</strong> of handwriting. For best results, crop your image to one line at a time.</span>
    </div>

  </div>
</template>

<style scoped>
.results {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.section {
  background: white;
  border: 1px solid var(--border);
  border-radius: var(--radius);
  padding: 16px 20px;
}
.section-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 12px;
}
.section-header h3 {
  font-size: 0.875rem;
  font-weight: 700;
  color: var(--text);
  text-transform: uppercase;
  letter-spacing: 0.04em;
}

.btn-copy {
  display: flex;
  align-items: center;
  gap: 5px;
  background: none;
  border: 1.5px solid var(--border);
  border-radius: 6px;
  padding: 4px 10px;
  font-size: 0.8rem;
  font-weight: 500;
  color: var(--text-muted);
  transition: all 0.15s;
}
.btn-copy:hover { border-color: var(--primary); color: var(--primary); }

.raw-text {
  background: #f8fafc;
  border: 1px solid var(--border);
  border-radius: 7px;
  padding: 14px;
  font-family: 'Courier New', monospace;
  font-size: 0.875rem;
  line-height: 1.7;
  white-space: pre-wrap;
  word-break: break-word;
  min-height: 80px;
  max-height: 400px;
  overflow-y: auto;
  color: var(--text);
}

.tip {
  display: flex;
  align-items: flex-start;
  gap: 8px;
  background: #eef2ff;
  border: 1px solid #c7d2fe;
  border-radius: 8px;
  padding: 12px 14px;
  font-size: 0.85rem;
  color: #3730a3;
  line-height: 1.5;
}
.tip svg { flex-shrink: 0; margin-top: 2px; }
</style>
