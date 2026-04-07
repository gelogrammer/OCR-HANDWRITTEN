<script setup>
import { ref, computed } from 'vue'

const props = defineProps({
  result: { type: Object, required: true }
})

const copied = ref(false)
const showWords = ref(false)
const showEngines = ref(false)

const analysis = computed(() => props.result.analysis)

const confidenceClass = computed(() => {
  const c = props.result.confidence
  if (c === null) return ''
  if (c >= 70) return 'conf-high'
  if (c >= 40) return 'conf-medium'
  return 'conf-low'
})

const flaggedWords = computed(() => {
  if (!props.result.words?.length) return []
  return props.result.words
    .filter(w => w.confidence < 60 && w.text.trim())
    .sort((a, b) => a.confidence - b.confidence)
    .slice(0, 20)
})

async function copyText() {
  await navigator.clipboard.writeText(props.result.text)
  copied.value = true
  setTimeout(() => { copied.value = false }, 2000)
}

async function copyFormatted() {
  const text = analysis.value.lines.map(l => l.formatted).join('\n')
  await navigator.clipboard.writeText(text)
}

const typeIcon = { number: '#', code: '{}', text: 'Aa' }
const typeColor = { number: 'type-num', code: 'type-code', text: 'type-text' }
</script>

<template>
  <div class="results">

    <!-- Meta badges -->
    <div class="meta-row">
      <span class="badge">{{ result.engine }}</span>
      <span v-if="result.confidence != null" :class="['badge', confidenceClass]">
        Confidence: {{ result.confidence }}%
      </span>
      <span v-if="analysis" class="badge badge-gray">{{ analysis.lines.length }} items detected</span>
    </div>

    <!-- Confidence bar -->
    <div v-if="result.confidence != null" class="conf-bar-wrap">
      <div class="conf-bar">
        <div class="conf-bar-fill" :class="confidenceClass" :style="{ width: result.confidence + '%' }" />
      </div>
      <span class="conf-value">{{ result.confidence }}%</span>
    </div>

    <!-- Auto-Analysis: classified lines -->
    <div v-if="analysis && analysis.lines.length" class="section">
      <div class="section-header">
        <h3>Detected Content</h3>
        <div class="type-summary">
          <span v-if="analysis.counts.number" class="type-count type-num">{{ analysis.counts.number }} numbers</span>
          <span v-if="analysis.counts.code" class="type-count type-code">{{ analysis.counts.code }} codes</span>
          <span v-if="analysis.counts.text" class="type-count type-text">{{ analysis.counts.text }} text</span>
        </div>
      </div>
      <div class="analyzed-lines">
        <div v-for="(line, i) in analysis.lines" :key="i" class="analyzed-line">
          <span :class="['type-badge', typeColor[line.type]]">{{ typeIcon[line.type] }}</span>
          <span class="line-raw">{{ line.raw }}</span>
          <span class="line-arrow">→</span>
          <span class="line-formatted">{{ line.formatted }}</span>
          <span :class="['line-type', typeColor[line.type]]">{{ line.type }}</span>
        </div>
      </div>
    </div>

    <!-- Raw text -->
    <div class="section">
      <div class="section-header">
        <h3>Raw OCR Output</h3>
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

    <!-- Per-engine results (Cloud multi-engine mode) -->
    <div v-if="result.engineResults?.length" class="section">
      <div class="section-header">
        <h3>Individual Engine Results</h3>
        <button class="btn-copy" @click="showEngines = !showEngines">
          {{ showEngines ? 'Hide' : 'Show' }} ({{ result.engineResults.length }})
        </button>
      </div>
      <div v-if="showEngines" class="engine-list">
        <div v-for="(er, i) in result.engineResults" :key="i" class="engine-result">
          <span class="engine-label">{{ er.engine }}</span>
          <pre class="raw-text alt-text">{{ er.text }}</pre>
        </div>
      </div>
    </div>

    <!-- Flagged words -->
    <div v-if="flaggedWords.length > 0" class="section">
      <div class="section-header">
        <h3>Flagged Words</h3>
        <span class="flagged-count">{{ flaggedWords.length }}</span>
      </div>
      <div class="flagged-list">
        <div v-for="(w, i) in flaggedWords" :key="i" class="flagged-word">
          <span class="fw-text">{{ w.text }}</span>
          <span class="fw-conf">{{ w.confidence }}%</span>
        </div>
      </div>
    </div>

    <!-- Word analysis -->
    <div v-if="result.words?.length > 0" class="section">
      <div class="section-header">
        <h3>Word Analysis</h3>
        <button class="btn-copy" @click="showWords = !showWords">
          {{ showWords ? 'Hide' : 'Show' }} ({{ result.words.length }})
        </button>
      </div>
      <div v-if="showWords" class="word-grid">
        <span v-for="(w, i) in result.words" :key="i"
          :class="['word-chip', w.confidence >= 70 ? 'wc-high' : w.confidence >= 40 ? 'wc-med' : 'wc-low']"
          :title="w.confidence + '%'"
        >{{ w.text }}</span>
      </div>
    </div>

    <div class="tip">
      <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
        <circle cx="12" cy="12" r="10"/><line x1="12" y1="8" x2="12" y2="12"/><line x1="12" y1="16" x2="12.01" y2="16"/>
      </svg>
      <span>Each line is auto-classified as <strong>number</strong>, <strong>code</strong>, or <strong>text</strong>. Try both Local and Cloud engines to compare accuracy.</span>
    </div>
  </div>
</template>

<style scoped>
.results { display: flex; flex-direction: column; gap: 16px; }

.meta-row { display: flex; flex-wrap: wrap; gap: 6px; }
.badge { background: #eef2ff; color: #3730a3; border: 1px solid #c7d2fe; border-radius: 99px; font-size: 0.75rem; font-weight: 600; padding: 3px 10px; }
.badge-gray { background: #f8fafc; color: var(--text-muted); border-color: var(--border); }
.conf-high { background: #f0fdf4; color: #166534; border-color: #bbf7d0; }
.conf-medium { background: #fffbeb; color: #92400e; border-color: #fcd34d; }
.conf-low { background: #fef2f2; color: #991b1b; border-color: #fca5a5; }

.conf-bar-wrap { display: flex; align-items: center; gap: 10px; }
.conf-bar { flex: 1; height: 8px; background: #e2e8f0; border-radius: 99px; overflow: hidden; }
.conf-bar-fill { height: 100%; border-radius: 99px; transition: width 0.5s ease; }
.conf-bar-fill.conf-high { background: #22c55e; }
.conf-bar-fill.conf-medium { background: #f59e0b; }
.conf-bar-fill.conf-low { background: #ef4444; }
.conf-value { font-size: 0.85rem; font-weight: 700; color: var(--text); min-width: 36px; }

.section { background: white; border: 1px solid var(--border); border-radius: var(--radius); padding: 16px 20px; }
.section-header { display: flex; align-items: center; justify-content: space-between; margin-bottom: 12px; flex-wrap: wrap; gap: 8px; }
.section-header h3 { font-size: 0.875rem; font-weight: 700; color: var(--text); text-transform: uppercase; letter-spacing: 0.04em; }

/* Type classification */
.type-summary { display: flex; gap: 6px; }
.type-count { font-size: 0.72rem; font-weight: 600; padding: 2px 8px; border-radius: 99px; }
.type-num { background: #dbeafe; color: #1e40af; }
.type-code { background: #fae8ff; color: #86198f; }
.type-text { background: #dcfce7; color: #166534; }

.analyzed-lines { display: flex; flex-direction: column; gap: 4px; }
.analyzed-line {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 6px 10px;
  background: #f8fafc;
  border: 1px solid var(--border);
  border-radius: 6px;
  font-size: 0.85rem;
}
.type-badge {
  flex-shrink: 0;
  width: 24px;
  height: 24px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 5px;
  font-size: 0.7rem;
  font-weight: 700;
  font-family: monospace;
}
.type-badge.type-num { background: #dbeafe; color: #1e40af; }
.type-badge.type-code { background: #fae8ff; color: #86198f; }
.type-badge.type-text { background: #dcfce7; color: #166534; }

.line-raw { color: var(--text-muted); flex: 1; min-width: 0; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.line-arrow { color: var(--border); font-size: 0.8rem; flex-shrink: 0; }
.line-formatted { font-weight: 600; color: var(--text); flex: 1; min-width: 0; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.line-type { font-size: 0.7rem; font-weight: 600; padding: 1px 6px; border-radius: 4px; flex-shrink: 0; text-transform: uppercase; }
.line-type.type-num { background: #dbeafe; color: #1e40af; }
.line-type.type-code { background: #fae8ff; color: #86198f; }
.line-type.type-text { background: #dcfce7; color: #166534; }

.btn-copy { display: flex; align-items: center; gap: 5px; background: none; border: 1.5px solid var(--border); border-radius: 6px; padding: 4px 10px; font-size: 0.8rem; font-weight: 500; color: var(--text-muted); transition: all 0.15s; }
.btn-copy:hover { border-color: var(--primary); color: var(--primary); }

.raw-text { background: #f8fafc; border: 1px solid var(--border); border-radius: 7px; padding: 14px; font-family: 'Courier New', monospace; font-size: 0.875rem; line-height: 1.7; white-space: pre-wrap; word-break: break-word; min-height: 60px; max-height: 300px; overflow-y: auto; color: var(--text); }

.flagged-count { background: #fef2f2; color: #991b1b; border: 1px solid #fca5a5; border-radius: 99px; font-size: 0.72rem; font-weight: 700; padding: 2px 8px; }
.flagged-list { display: flex; flex-wrap: wrap; gap: 6px; }
.flagged-word { display: flex; align-items: center; gap: 4px; background: #fef2f2; border: 1px solid #fecaca; border-radius: 6px; padding: 4px 8px; font-size: 0.82rem; }
.fw-text { font-weight: 600; color: #991b1b; }
.fw-conf { font-size: 0.72rem; color: #b91c1c; }

.word-grid { display: flex; flex-wrap: wrap; gap: 4px; max-height: 200px; overflow-y: auto; }
.word-chip { padding: 3px 8px; border-radius: 4px; font-size: 0.8rem; font-weight: 500; }
.wc-high { background: #f0fdf4; color: #166534; }
.wc-med { background: #fffbeb; color: #92400e; }
.wc-low { background: #fef2f2; color: #991b1b; font-weight: 700; }

.engine-list { display: flex; flex-direction: column; gap: 10px; }
.engine-result { display: flex; flex-direction: column; gap: 4px; }
.engine-label { font-size: 0.75rem; font-weight: 700; color: var(--primary); text-transform: uppercase; letter-spacing: 0.04em; }
.alt-text { background: #fffbeb; border-color: #fde68a; }

.tip { display: flex; align-items: flex-start; gap: 8px; background: #eef2ff; border: 1px solid #c7d2fe; border-radius: 8px; padding: 12px 14px; font-size: 0.85rem; color: #3730a3; line-height: 1.5; }
.tip svg { flex-shrink: 0; margin-top: 2px; }
</style>
