<script setup>
import { ref } from 'vue'
import { createWorker, PSM } from 'tesseract.js'
import { InferenceClient } from '@huggingface/inference'
import ResultsPanel from './components/ResultsPanel.vue'
import HelpModal from './components/HelpModal.vue'

const fileInput = ref(null)
const imageData = ref(null)
const processedPreview = ref(null)
const processing = ref(false)
const progressStatus = ref('')
const progressPct = ref(0)
const ocrResult = ref(null)
const showHelp = ref(false)
const dragOver = ref(false)
const errorMsg = ref(null)
const showPreprocessed = ref(false)

const ocrMode = ref('local')
const apiKey = ref('')
const gvisionKey = ref('')
const hfToken = ref('')
const preprocessEnabled = ref(true)

const OCR_SPACE_URL = 'https://api.ocr.space/parse/image'
const DEFAULT_KEY = 'K85526706188957'
const DEFAULT_HF_TOKEN = ''
const MAX_SIZE = 3000

// ─── Image Utilities ───

function loadImage(src) {
  return new Promise((resolve, reject) => {
    const img = new Image()
    img.onload = () => resolve(img)
    img.onerror = reject
    img.src = src
  })
}

function createCanvas(img) {
  const maxDim = Math.max(img.width, img.height)
  let scale
  if (maxDim < 500) scale = Math.min(4, MAX_SIZE / maxDim)
  else if (maxDim < 1000) scale = Math.min(2.5, MAX_SIZE / maxDim)
  else scale = Math.min(1, MAX_SIZE / maxDim)
  const c = document.createElement('canvas')
  c.width = Math.round(img.width * scale)
  c.height = Math.round(img.height * scale)
  const ctx = c.getContext('2d')
  ctx.imageSmoothingEnabled = true
  ctx.imageSmoothingQuality = 'high'
  ctx.drawImage(img, 0, 0, c.width, c.height)
  return { canvas: c, ctx, w: c.width, h: c.height, originalMax: maxDim }
}

// ─── Advanced Preprocessing Pipeline ───

function grayscale(d) {
  const p = d.data
  for (let i = 0; i < p.length; i += 4) {
    const g = 0.299 * p[i] + 0.587 * p[i + 1] + 0.114 * p[i + 2]
    p[i] = p[i + 1] = p[i + 2] = g
  }
}

// Median filter — removes speckle/salt-pepper noise while preserving edges
function medianFilter(d, w, h) {
  const p = d.data, out = new Uint8ClampedArray(p)
  const buf = new Uint8Array(9)
  for (let y = 1; y < h - 1; y++)
    for (let x = 1; x < w - 1; x++) {
      let idx = 0
      for (let ky = -1; ky <= 1; ky++)
        for (let kx = -1; kx <= 1; kx++)
          buf[idx++] = p[((y + ky) * w + (x + kx)) * 4]
      buf.sort()
      const i = (y * w + x) * 4
      out[i] = out[i + 1] = out[i + 2] = buf[4]
      out[i + 3] = 255
    }
  d.data.set(out)
}

// CLAHE — Contrast Limited Adaptive Histogram Equalization
// Normalizes uneven lighting from phone photos
function clahe(d, w, h, tiles = 8, clipLimit = 2.5) {
  const p = d.data
  const tileMaps = []
  for (let ty = 0; ty < tiles; ty++) {
    tileMaps[ty] = []
    for (let tx = 0; tx < tiles; tx++) {
      const x0 = Math.round(tx * w / tiles), x1 = Math.round((tx + 1) * w / tiles)
      const y0 = Math.round(ty * h / tiles), y1 = Math.round((ty + 1) * h / tiles)
      const hist = new Float64Array(256)
      let count = 0
      for (let y = y0; y < y1; y++)
        for (let x = x0; x < x1; x++) { hist[p[(y * w + x) * 4]]++; count++ }
      const clip = clipLimit * count / 256
      let excess = 0
      for (let i = 0; i < 256; i++) {
        if (hist[i] > clip) { excess += hist[i] - clip; hist[i] = clip }
      }
      const bonus = excess / 256
      for (let i = 0; i < 256; i++) hist[i] += bonus
      const map = new Uint8Array(256)
      let sum = 0
      for (let i = 0; i < 256; i++) { sum += hist[i]; map[i] = Math.round((sum / count) * 255) }
      tileMaps[ty][tx] = map
    }
  }
  for (let y = 0; y < h; y++)
    for (let x = 0; x < w; x++) {
      const idx = (y * w + x) * 4, val = p[idx]
      const fy = (y * tiles / h) - 0.5, fx = (x * tiles / w) - 0.5
      const ty1 = Math.max(0, Math.floor(fy)), ty2 = Math.min(tiles - 1, ty1 + 1)
      const tx1 = Math.max(0, Math.floor(fx)), tx2 = Math.min(tiles - 1, tx1 + 1)
      const dy = Math.max(0, fy - ty1), dx = Math.max(0, fx - tx1)
      const v = Math.round(
        (1 - dy) * ((1 - dx) * tileMaps[ty1][tx1][val] + dx * tileMaps[ty1][tx2][val]) +
        dy * ((1 - dx) * tileMaps[ty2][tx1][val] + dx * tileMaps[ty2][tx2][val])
      )
      p[idx] = p[idx + 1] = p[idx + 2] = Math.min(255, Math.max(0, v))
    }
}

// Sauvola adaptive thresholding — handles uneven lighting far better than Otsu
// Uses integral images for O(n) performance
function sauvola(d, w, h, winSize = 15, k = 0.2, R = 128) {
  const p = d.data
  const half = Math.floor(winSize / 2)
  const out = new Uint8ClampedArray(p)
  const integral = new Float64Array(w * h)
  const integralSq = new Float64Array(w * h)
  for (let y = 0; y < h; y++) {
    let rowSum = 0, rowSumSq = 0
    for (let x = 0; x < w; x++) {
      const val = p[(y * w + x) * 4]
      rowSum += val; rowSumSq += val * val
      integral[y * w + x] = rowSum + (y > 0 ? integral[(y - 1) * w + x] : 0)
      integralSq[y * w + x] = rowSumSq + (y > 0 ? integralSq[(y - 1) * w + x] : 0)
    }
  }
  for (let y = 0; y < h; y++)
    for (let x = 0; x < w; x++) {
      const x1 = Math.max(0, x - half), y1 = Math.max(0, y - half)
      const x2 = Math.min(w - 1, x + half), y2 = Math.min(h - 1, y + half)
      const count = (x2 - x1 + 1) * (y2 - y1 + 1)
      let sum = integral[y2 * w + x2], sumSq = integralSq[y2 * w + x2]
      if (x1 > 0) { sum -= integral[y2 * w + (x1 - 1)]; sumSq -= integralSq[y2 * w + (x1 - 1)] }
      if (y1 > 0) { sum -= integral[(y1 - 1) * w + x2]; sumSq -= integralSq[(y1 - 1) * w + x2] }
      if (x1 > 0 && y1 > 0) { sum += integral[(y1 - 1) * w + (x1 - 1)]; sumSq += integralSq[(y1 - 1) * w + (x1 - 1)] }
      const mean = sum / count
      const stdev = Math.sqrt(Math.max(0, (sumSq / count) - (mean * mean)))
      const threshold = mean * (1 + k * (stdev / R - 1))
      const idx = (y * w + x) * 4
      const v = p[idx] < threshold ? 0 : 255
      out[idx] = out[idx + 1] = out[idx + 2] = v; out[idx + 3] = 255
    }
  d.data.set(out)
}

// Deskew — detect and correct rotation via projection profile analysis
function detectSkewAngle(d, w, h) {
  const p = d.data
  let bestAngle = 0, bestScore = 0
  for (let angle = -10; angle <= 10; angle += 0.5) {
    const rad = (angle * Math.PI) / 180
    const cos = Math.cos(rad), sin = Math.sin(rad)
    const cx = w / 2, cy = h / 2
    const profile = new Int32Array(h)
    // Sample every 3rd pixel for speed
    for (let y = 0; y < h; y += 1)
      for (let x = 0; x < w; x += 3) {
        if (p[(y * w + x) * 4] === 0) {
          const ny = Math.round((x - cx) * sin + (y - cy) * cos + cy)
          if (ny >= 0 && ny < h) profile[ny]++
        }
      }
    let score = 0
    for (let i = 0; i < h; i++) score += profile[i] * profile[i]
    if (score > bestScore) { bestScore = score; bestAngle = angle }
  }
  return Math.abs(bestAngle) < 0.5 ? 0 : bestAngle
}

function rotateCanvas(canvas, ctx, angle) {
  if (angle === 0) return
  const rad = (angle * Math.PI) / 180
  const temp = document.createElement('canvas')
  temp.width = canvas.width; temp.height = canvas.height
  const tCtx = temp.getContext('2d')
  tCtx.fillStyle = 'white'
  tCtx.fillRect(0, 0, temp.width, temp.height)
  tCtx.translate(temp.width / 2, temp.height / 2)
  tCtx.rotate(-rad)
  tCtx.drawImage(canvas, -canvas.width / 2, -canvas.height / 2)
  ctx.clearRect(0, 0, canvas.width, canvas.height)
  ctx.drawImage(temp, 0, 0)
}

function sharpen(ctx, w, h) {
  const k = [0, -1, 0, -1, 5, -1, 0, -1, 0]
  const s = ctx.getImageData(0, 0, w, h), d = ctx.createImageData(w, h)
  const sd = s.data, dd = d.data
  for (let y = 1; y < h - 1; y++)
    for (let x = 1; x < w - 1; x++) {
      for (let c = 0; c < 3; c++) {
        let v = 0
        for (let ky = -1; ky <= 1; ky++)
          for (let kx = -1; kx <= 1; kx++)
            v += sd[((y + ky) * w + (x + kx)) * 4 + c] * k[(ky + 1) * 3 + (kx + 1)]
        dd[(y * w + x) * 4 + c] = Math.min(255, Math.max(0, v))
      }
      dd[(y * w + x) * 4 + 3] = 255
    }
  return d
}

// Remove table gridlines
function removeLines(d, w, h) {
  const p = d.data
  for (let y = 0; y < h; y++) {
    let black = 0
    for (let x = 0; x < w; x++) if (p[(y * w + x) * 4] === 0) black++
    if (black / w > 0.5) {
      for (let x = 0; x < w; x++) { const i = (y * w + x) * 4; p[i] = p[i + 1] = p[i + 2] = 255 }
    }
  }
  for (let x = 0; x < w; x++) {
    let black = 0
    for (let y = 0; y < h; y++) if (p[(y * w + x) * 4] === 0) black++
    if (black / h > 0.5) {
      for (let y = 0; y < h; y++) { const i = (y * w + x) * 4; p[i] = p[i + 1] = p[i + 2] = 255 }
    }
  }
}

// Morphological operations
function dilate(d, w, h) {
  const src = new Uint8ClampedArray(d.data), p = d.data
  for (let y = 0; y < h; y++)
    for (let x = 0; x < w; x++) {
      let min = 255
      for (let ky = -1; ky <= 1; ky++)
        for (let kx = -1; kx <= 1; kx++) {
          const ny = y + ky, nx = x + kx
          if (ny >= 0 && ny < h && nx >= 0 && nx < w)
            min = Math.min(min, src[(ny * w + nx) * 4])
        }
      const i = (y * w + x) * 4; p[i] = p[i + 1] = p[i + 2] = min
    }
}

function erode(d, w, h) {
  const src = new Uint8ClampedArray(d.data), p = d.data
  for (let y = 0; y < h; y++)
    for (let x = 0; x < w; x++) {
      let max = 0
      for (let ky = -1; ky <= 1; ky++)
        for (let kx = -1; kx <= 1; kx++) {
          const ny = y + ky, nx = x + kx
          if (ny >= 0 && ny < h && nx >= 0 && nx < w)
            max = Math.max(max, src[(ny * w + nx) * 4])
        }
      const i = (y * w + x) * 4; p[i] = p[i + 1] = p[i + 2] = max
    }
}

// Morphological close (dilate then erode) — reconnects broken handwriting strokes
function morphClose(d, w, h) { dilate(d, w, h); erode(d, w, h) }

// Add white border — Tesseract performs better when text doesn't touch edges
function addBorder(canvas, ctx, padding = 20) {
  const temp = document.createElement('canvas')
  temp.width = canvas.width + padding * 2
  temp.height = canvas.height + padding * 2
  const tCtx = temp.getContext('2d')
  tCtx.fillStyle = 'white'
  tCtx.fillRect(0, 0, temp.width, temp.height)
  tCtx.drawImage(canvas, padding, padding)
  canvas.width = temp.width; canvas.height = temp.height
  ctx.drawImage(temp, 0, 0)
}

// ─── Preprocessing Pipelines ───

// Light preprocessing for Cloud APIs — CLAHE + deskew (no binarization)
async function preprocessLight(dataUrl) {
  const img = await loadImage(dataUrl)
  const { canvas, ctx, w, h } = createCanvas(img)

  if (preprocessEnabled.value) {
    let d = ctx.getImageData(0, 0, w, h)
    grayscale(d)
    medianFilter(d, w, h)
    clahe(d, w, h, 8, 2.5)
    ctx.putImageData(d, 0, 0)

    // Deskew after CLAHE (need binary for angle detection)
    d = ctx.getImageData(0, 0, w, h)
    const tempD = new ImageData(new Uint8ClampedArray(d.data), w, h)
    sauvola(tempD, w, h, 25, 0.3)
    const angle = detectSkewAngle(tempD, w, h)
    if (angle !== 0) rotateCanvas(canvas, ctx, angle)
  }

  // JPEG to keep under OCR.space 1MB limit
  let out = canvas.toDataURL('image/jpeg', 0.85)
  if (out.length > 1_300_000) out = canvas.toDataURL('image/jpeg', 0.6)
  if (out.length > 1_300_000) out = canvas.toDataURL('image/jpeg', 0.4)
  return out
}

// Heavy preprocessing for Tesseract — full pipeline to binary
async function preprocessHeavy(dataUrl) {
  const img = await loadImage(dataUrl)
  const { canvas, ctx, w, h, originalMax } = createCanvas(img)
  if (!preprocessEnabled.value) return canvas.toDataURL('image/png')

  // 1. Grayscale
  let d = ctx.getImageData(0, 0, w, h)
  grayscale(d)
  ctx.putImageData(d, 0, 0)

  // 2. Median filter — remove noise
  d = ctx.getImageData(0, 0, w, h)
  medianFilter(d, w, h)
  ctx.putImageData(d, 0, 0)

  // 3. CLAHE — normalize lighting
  d = ctx.getImageData(0, 0, w, h)
  clahe(d, w, h, 8, 2.5)
  ctx.putImageData(d, 0, 0)

  // 4. Sharpen
  ctx.putImageData(sharpen(ctx, w, h), 0, 0)

  // 5. Sauvola adaptive threshold (replaces global Otsu)
  d = ctx.getImageData(0, 0, w, h)
  const winSize = Math.max(15, Math.round(Math.min(w, h) / 40) | 1)
  sauvola(d, w, h, winSize, 0.2)

  // 6. Remove table gridlines
  removeLines(d, w, h)

  // 7. Morphological close — reconnect broken strokes
  morphClose(d, w, h)

  ctx.putImageData(d, 0, 0)

  // 8. Deskew
  d = ctx.getImageData(0, 0, w, h)
  const angle = detectSkewAngle(d, w, h)
  if (angle !== 0) rotateCanvas(canvas, ctx, angle)

  // 9. Add white border
  addBorder(canvas, ctx, 20)

  return canvas.toDataURL('image/png')
}

// ─── Post-OCR: Clean + Classify ───

function cleanLine(line) {
  return line.trim()
    .replace(/^\|[-\s]*\|$/g, '')
    .replace(/^\|+\s*/, '').replace(/\s*\|+$/g, '')
    .replace(/^[•·▪►]\s*/g, '')
    .replace(/^["\u201C\u201D]+\s*/, '')
    .replace(/^[.\-:;,=]*\s*/, '')
    .replace(/^\d{1,2}\s*[.|):\s\t|=-]+\s*/g, '')
    .replace(/^[IlJ|]\s*[.|):\s\t|-]+\s*/g, '')
    .replace(/\t+/g, ' ').replace(/\s{2,}/g, ' ')
    .trim()
}

function classifyLine(line) {
  const raw = line.trim()
  if (!raw) return null
  const cleaned = cleanLine(raw)
  if (!cleaned) return null
  if (/^(page\s+\d|assigned\s+cre)/i.test(cleaned)) return null
  // Skip standalone small numbers (row numbers that weren't stripped)
  if (/^\d{1,2}$/.test(cleaned)) return null
  // Skip very short noise or garbage strings with backslashes/special chars
  if (cleaned.length < 3) return null
  if (/[\\{}]/.test(cleaned)) return null
  if (/^[\d,.\-\s]+$/.test(cleaned)) {
    const n = cleaned.replace(/[,\s]/g, '')
    return { raw, cleaned, type: 'number', formatted: isNaN(Number(n)) ? cleaned : Number(n).toLocaleString() }
  }
  if (/^[A-Za-z0-9.\-/]{1,15}$/.test(cleaned) && /\d/.test(cleaned) && /[A-Za-z]/.test(cleaned)) {
    return { raw, cleaned, type: 'code', formatted: cleaned.toUpperCase() }
  }
  return { raw, cleaned, type: 'text', formatted: cleaned.toUpperCase() }
}

function analyzeText(text) {
  const lines = text.split('\n').map(classifyLine).filter(Boolean)
  const counts = { number: 0, code: 0, text: 0 }
  lines.forEach(l => counts[l.type]++)
  return { lines, counts }
}

// ─── Multi-engine Consensus Merger ───

function scoreText(t) {
  if (!t) return -1
  const cleaned = cleanLine(t)
  const alpha = (cleaned.match(/[A-Za-z]/g) || []).length
  const digits = (cleaned.match(/[0-9]/g) || []).length
  const noise = (cleaned.match(/[^A-Za-z0-9\s.,\-/()]/g) || []).length
  const words = cleaned.split(/\s+/).filter(w => /^[A-Za-z]{2,}$/.test(w))
  const fragments = cleaned.split(/\s+/).filter(w => w.length === 1 && /[A-Za-z]/.test(w))
  return alpha + digits * 0.5 + words.length * 3 - noise * 3 - fragments.length * 2
}

function normalizeForCompare(s) {
  return s.toUpperCase().replace(/[^A-Z0-9]/g, '')
}

function preCleanEngine(text) {
  return text.split('\n')
    .map(l => l.trim())
    .filter(l => l && !/^\|[-\s]+\|$/.test(l))
    .filter(l => !/^Page\s+\d/i.test(l))
    .filter(l => !/^Assigned\s+Cre/i.test(l))
    .map(l => l.replace(/^\|+\s*/, '').replace(/\s*\|+$/, ''))
    .map(l => l.replace(/^[•·▪►]\s*/, ''))
    .map(l => l.replace(/^["\u201C\u201D]+\s*/, ''))
    .map(l => l.replace(/^[=]+\s*/, ''))                    // strip leading = signs
    .map(l => l.replace(/^\d{1,2}\s*[|)\s\t.-]+\s*/g, ''))
    .map(l => l.replace(/\t+/g, ' ').replace(/\s{2,}/g, ' ').trim())
    .filter(Boolean)
    // Remove lines that are mostly non-Latin (Cyrillic OCR artifacts)
    .filter(l => {
      const latin = (l.match(/[A-Za-z]/g) || []).length
      const total = l.replace(/\s/g, '').length
      return total === 0 || latin / total > 0.5
    })
    .join('\n')
}

function lcsRatio(a, b) {
  const shorter = a.length < b.length ? a : b
  const longer = a.length < b.length ? b : a
  if (shorter.length < 4) return 0
  let matches = 0, j = 0
  for (let i = 0; i < longer.length && j < shorter.length; i++) {
    if (longer[i] === shorter[j]) { matches++; j++ }
  }
  return matches / shorter.length
}

function mergeResults(results) {
  const valid = results.filter(r => r && r.trim()).map(preCleanEngine).filter(Boolean)
  if (valid.length === 0) return ''
  if (valid.length === 1) return valid[0]

  const lineArrays = valid.map(r => r.split('\n').map(l => l.trim()).filter(Boolean))
  const lineCounts = lineArrays.map(a => a.length)
  const maxLines = Math.max(...lineCounts)
  const medianCount = [...lineCounts].sort((a, b) => a - b)[Math.floor(lineCounts.length / 2)]

  // Filter out engines that returned way too few lines (< 40% of median)
  // They'll misalign the line-by-line merge
  const usableArrays = []
  for (let j = 0; j < lineArrays.length; j++) {
    if (lineArrays[j].length >= medianCount * 0.4) {
      usableArrays.push({ lines: lineArrays[j], engineIdx: j })
    }
  }
  if (usableArrays.length === 0) usableArrays.push({ lines: lineArrays[0], engineIdx: 0 })

  // Use the longest usable engine as the reference for line count
  const refLength = Math.max(...usableArrays.map(a => a.lines.length))
  const merged = []
  for (let i = 0; i < refLength; i++) {
    const candidates = []
    for (const eng of usableArrays) {
      const line = eng.lines[i]
      if (line) candidates.push({ text: line, engineIdx: eng.engineIdx })
    }
    if (candidates.length === 0) continue

    // Check consensus first
    const normalized = candidates.map(c => normalizeForCompare(c.text))
    let consensusLine = null
    for (const c of candidates) {
      const n = normalizeForCompare(c.text)
      if (normalized.filter(x => x === n).length >= 2) { consensusLine = c.text; break }
    }
    if (consensusLine) { merged.push(consensusLine); continue }

    // No consensus — pick best by score
    let bestLine = candidates[0].text, bestScore = -Infinity
    for (const c of candidates) {
      let total = scoreText(c.text)
      // Engine 2 (idx 1) bonus for handwriting
      if (c.engineIdx === 1) total += 2
      if (total > bestScore) { bestScore = total; bestLine = c.text }
    }
    merged.push(bestLine)
  }

  // Also collect unmatched lines from filtered-out engines (they may have unique good lines)
  for (let j = 0; j < lineArrays.length; j++) {
    const wasUsed = usableArrays.some(u => u.engineIdx === j)
    if (wasUsed) continue
    for (const line of lineArrays[j]) {
      const norm = normalizeForCompare(line)
      if (norm.length < 4) continue
      // Only add if it doesn't match any existing merged line
      const exists = merged.some(m => {
        const mn = normalizeForCompare(m)
        return mn === norm || lcsRatio(mn, norm) > 0.6
      })
      if (!exists && scoreText(line) > 5) merged.push(line)
    }
  }
  // Deduplicate
  const deduped = []
  for (const line of merged) {
    const norm = normalizeForCompare(line)
    if (!norm) continue
    const dupeIdx = deduped.findIndex(ex => {
      const en = normalizeForCompare(ex)
      return en === norm || lcsRatio(en, norm) > 0.6
    })
    if (dupeIdx === -1) deduped.push(line)
    else if (scoreText(line) > scoreText(deduped[dupeIdx])) deduped[dupeIdx] = line
  }
  return deduped.join('\n')
}

// ─── File handling ───

function handleFile(file) {
  if (!file || !file.type.startsWith('image/')) return
  const reader = new FileReader()
  reader.onload = e => {
    imageData.value = e.target.result
    ocrResult.value = null; errorMsg.value = null; processedPreview.value = null
  }
  reader.readAsDataURL(file)
}
function onDrop(e) { dragOver.value = false; handleFile(e.dataTransfer.files[0]) }
function onFileInput(e) { handleFile(e.target.files[0]) }
function clearImage() {
  imageData.value = null; ocrResult.value = null; errorMsg.value = null
  processedPreview.value = null
  if (fileInput.value) fileInput.value.value = ''
}

// ─── Tesseract.js (Local) — Optimized for handwriting ───

async function runTesseract(processedDataUrl) {
  progressStatus.value = 'Loading Tesseract engine...'
  progressPct.value = 10

  const worker = await createWorker('eng', 1, {
    logger: m => {
      if (m.status === 'recognizing text') {
        progressPct.value = 10 + Math.round(m.progress * 80)
        progressStatus.value = `Recognizing... ${Math.round(m.progress * 100)}%`
      }
    }
  }, {
    // Disable dictionary forcing — prevents handwriting from being
    // forced into dictionary words (e.g. "CRUZ" → "CRUX")
    load_system_dawg: '0',
    load_freq_dawg: '0',
  })

  await worker.setParameters({
    tessedit_pageseg_mode: PSM.SINGLE_COLUMN, // Better for lists/tables than AUTO
    preserve_interword_spaces: '1',           // Handwriting has irregular spacing
    user_defined_dpi: '300',                  // Explicit DPI
  })

  const { data } = await worker.recognize(processedDataUrl, { rotateAuto: true })
  await worker.terminate()

  const text = data.text.trim().replace(/\n{3,}/g, '\n\n').replace(/^\s*[-=~_]{2,}\s*$/gm, '')
  return {
    text, confidence: Math.round(data.confidence),
    words: data.words?.map(w => ({ text: w.text, confidence: Math.round(w.confidence) })) || [],
    engine: 'Tesseract.js (Optimized)',
    analysis: analyzeText(text)
  }
}

// ─── OCR.space (Cloud) — Multi-engine with consensus ───

async function callOcrSpace(dataUrl, engine) {
  const body = new FormData()
  body.append('base64image', dataUrl)
  body.append('language', 'eng')
  body.append('OCREngine', String(engine))
  body.append('scale', 'true')
  body.append('detectOrientation', 'true')
  body.append('isTable', 'true')
  body.append('isOverlayRequired', 'true')
  body.append('apikey', apiKey.value.trim() || DEFAULT_KEY)
  const res = await fetch(OCR_SPACE_URL, { method: 'POST', body })
  const data = await res.json()
  if (data.IsErroredOnProcessing) throw new Error(data.ErrorMessage?.[0] || 'Failed')
  return data.ParsedResults?.[0]?.ParsedText?.trim() || ''
}

async function runOcrSpace(processedDataUrl) {
  const results = {}
  const engines = [1, 2, 3]
  for (let i = 0; i < engines.length; i++) {
    const eng = engines[i]
    progressStatus.value = `OCR.space Engine ${eng} (${i + 1}/${engines.length})...`
    progressPct.value = 20 + Math.round((i / engines.length) * 60)
    try { results[eng] = await callOcrSpace(processedDataUrl, eng) } catch { results[eng] = '' }
  }
  const allTexts = engines.map(e => results[e]).filter(Boolean)
  const merged = mergeResults(allTexts)
  if (!merged) throw new Error('No text detected across all engines.')
  const engineTexts = engines
    .filter(e => results[e])
    .map(e => ({ engine: `Engine ${e}`, text: results[e] }))
  return {
    text: merged, confidence: null, words: [],
    engine: 'OCR.space (3-Engine Consensus)',
    analysis: analyzeText(merged),
    engineResults: engineTexts
  }
}

// ─── Google Cloud Vision (Best handwriting OCR — 1000 free/month) ───

async function runGoogleVision(processedDataUrl) {
  const key = gvisionKey.value.trim()
  if (!key) throw new Error('Google Cloud Vision API key required.')

  progressStatus.value = 'Google Cloud Vision — analyzing...'
  progressPct.value = 30

  // Strip data URL prefix to get raw base64
  const base64 = processedDataUrl.replace(/^data:image\/\w+;base64,/, '')

  const res = await fetch(`https://vision.googleapis.com/v1/images:annotate?key=${key}`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      requests: [{
        image: { content: base64 },
        features: [
          { type: 'DOCUMENT_TEXT_DETECTION', maxResults: 1 }
        ],
        imageContext: {
          languageHints: ['en']
        }
      }]
    })
  })

  const data = await res.json()
  if (data.error) throw new Error(data.error.message || 'Google Vision API error')

  const response = data.responses?.[0]
  if (response?.error) throw new Error(response.error.message)

  const fullText = response?.fullTextAnnotation?.text?.trim()
  if (!fullText) throw new Error('No text detected by Google Cloud Vision.')

  // Extract word-level confidence from pages
  const words = []
  const pages = response?.fullTextAnnotation?.pages || []
  for (const page of pages)
    for (const block of page.blocks || [])
      for (const para of block.paragraphs || [])
        for (const word of para.words || []) {
          const text = word.symbols?.map(s => s.text).join('') || ''
          const conf = Math.round((word.confidence || 0) * 100)
          if (text.trim()) words.push({ text, confidence: conf })
        }

  const avgConf = words.length > 0
    ? Math.round(words.reduce((s, w) => s + w.confidence, 0) / words.length)
    : null

  progressPct.value = 90
  return {
    text: fullText,
    confidence: avgConf,
    words,
    engine: 'Google Cloud Vision',
    analysis: analyzeText(fullText)
  }
}

// ─── HuggingFace VLM OCR (Free vision model for handwriting) ───

async function runHuggingFace(processedDataUrl) {
  const token = hfToken.value.trim()
  if (!token) throw new Error('HuggingFace token required. Get a free one at huggingface.co/settings/tokens')
  progressStatus.value = 'HuggingFace VLM — analyzing handwriting...'
  progressPct.value = 20

  const client = new InferenceClient(token)

  // Compress image for the API (keep under 1MB base64)
  const img = await loadImage(processedDataUrl)
  const canvas = document.createElement('canvas')
  const maxDim = 1500
  const scale = Math.min(1, maxDim / Math.max(img.width, img.height))
  canvas.width = Math.round(img.width * scale)
  canvas.height = Math.round(img.height * scale)
  const ctx = canvas.getContext('2d')
  ctx.drawImage(img, 0, 0, canvas.width, canvas.height)
  let dataUrl = canvas.toDataURL('image/jpeg', 0.85)
  if (dataUrl.length > 1_300_000) dataUrl = canvas.toDataURL('image/jpeg', 0.6)

  progressStatus.value = 'Sending to Qwen VLM...'
  progressPct.value = 40

  const response = await client.chatCompletion({
    model: 'Qwen/Qwen2.5-VL-72B-Instruct',
    messages: [{
      role: 'user',
      content: [
        { type: 'image_url', image_url: { url: dataUrl } },
        { type: 'text', text: 'Extract ALL handwritten text from this image. Output each name or line of text on its own line, exactly as written. Do not add numbering, bullets, or formatting. Only output the raw text, nothing else.' }
      ]
    }],
    max_tokens: 1024,
  })

  progressPct.value = 90

  const fullText = response?.choices?.[0]?.message?.content?.trim()
  if (!fullText) throw new Error('HuggingFace VLM returned no text.')

  console.log('[HF-VLM] Raw response:', fullText)

  return {
    text: fullText,
    confidence: null,
    words: [],
    engine: 'HuggingFace VLM (Qwen2.5-VL)',
    analysis: analyzeText(fullText)
  }
}

// ─── Main runner ───

async function runOCR() {
  if (!imageData.value) return
  processing.value = true
  progressPct.value = 0
  progressStatus.value = 'Preprocessing...'
  ocrResult.value = null
  errorMsg.value = null

  try {
    const mode = ocrMode.value
    const isLocal = mode === 'local'
    const isTrOCR = mode === 'trocr'
    const isGVision = mode === 'gvision'

    // Tesseract uses heavy (binary) preprocessing; TrOCR and cloud APIs use light (grayscale)
    const processed = isLocal
      ? await preprocessHeavy(imageData.value)
      : await preprocessLight(imageData.value)
    processedPreview.value = processed
    progressPct.value = 10

    let result
    if (isLocal) result = await runTesseract(processed)
    else if (isTrOCR) result = await runHuggingFace(processed)
    else if (isGVision) result = await runGoogleVision(processed)
    else result = await runOcrSpace(processed)

    progressPct.value = 100
    progressStatus.value = 'Done!'
    ocrResult.value = result
  } catch (err) {
    errorMsg.value = err.message
  } finally {
    processing.value = false
  }
}
</script>

<template>
  <div class="app">
    <header class="header">
      <div class="header-inner">
        <div class="logo">
          <svg width="28" height="28" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"/>
            <polyline points="14 2 14 8 20 8"/><line x1="16" y1="13" x2="8" y2="13"/>
            <line x1="16" y1="17" x2="8" y2="17"/><polyline points="10 9 9 9 8 9"/>
          </svg>
          <span>Handwriting OCR</span>
        </div>
        <div class="header-right">
          <span class="model-badge" v-if="ocrMode === 'local'">Tesseract.js · Local</span>
          <span class="model-badge" v-else-if="ocrMode === 'trocr'">Qwen VLM · HuggingFace</span>
          <span class="model-badge" v-else-if="ocrMode === 'gvision'">Google Vision · Cloud</span>
          <span class="model-badge" v-else>OCR.space · 3-Engine</span>
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

    <main class="main">
      <div class="left-panel">
        <div class="upload-zone" :class="{ 'drag-over': dragOver, 'has-image': imageData }"
          @dragover.prevent="dragOver = true" @dragleave.prevent="dragOver = false"
          @drop.prevent="onDrop" @click="!imageData && fileInput.click()">
          <input ref="fileInput" type="file" accept="image/*" @change="onFileInput" hidden />
          <div v-if="!imageData" class="upload-placeholder">
            <svg width="48" height="48" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5" opacity="0.4">
              <rect x="3" y="3" width="18" height="18" rx="2"/><circle cx="8.5" cy="8.5" r="1.5"/>
              <polyline points="21 15 16 10 5 21"/>
            </svg>
            <p class="upload-title">Drop image here or click to browse</p>
            <p class="upload-hint">PNG, JPG, WEBP &mdash; handwritten text, tables, numbers</p>
          </div>
          <template v-else>
            <img :src="imageData" class="preview-img" alt="Uploaded" />
            <button class="btn-clear" @click.stop="clearImage" title="Remove">
              <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5">
                <line x1="18" y1="6" x2="6" y2="18"/><line x1="6" y1="6" x2="18" y2="18"/>
              </svg>
            </button>
          </template>
        </div>

        <div v-if="processedPreview && showPreprocessed" class="preview-section">
          <div class="preview-header">
            <span class="preview-label">Preprocessed</span>
            <button class="btn-toggle-preview" @click="showPreprocessed = false">Hide</button>
          </div>
          <img :src="processedPreview" class="preview-img-small" alt="Preprocessed" />
        </div>
        <button v-else-if="processedPreview" class="btn-show-preview" @click="showPreprocessed = true">Show preprocessed image</button>

        <div class="controls">
          <div class="engine-toggle">
            <button :class="['engine-btn', { active: ocrMode === 'local' }]" @click="ocrMode = 'local'" :disabled="processing">
              <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                <rect x="2" y="3" width="20" height="14" rx="2"/><line x1="8" y1="21" x2="16" y2="21"/><line x1="12" y1="17" x2="12" y2="21"/>
              </svg> Local
            </button>
            <button :class="['engine-btn', { active: ocrMode === 'cloud' }]" @click="ocrMode = 'cloud'" :disabled="processing">
              <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                <path d="M18 10h-1.26A8 8 0 1 0 9 20h9a5 5 0 0 0 0-10z"/>
              </svg> OCR.space
            </button>
            <button :class="['engine-btn gv-btn', { active: ocrMode === 'gvision' }]" @click="ocrMode = 'gvision'" :disabled="processing">
              <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                <path d="M1 12s4-8 11-8 11 8 11 8-4 8-11 8-11-8-11-8z"/><circle cx="12" cy="12" r="3"/>
              </svg> G.Vision
            </button>
            <button :class="['engine-btn hf-btn', { active: ocrMode === 'trocr' }]" @click="ocrMode = 'trocr'" :disabled="processing">
              <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                <path d="M12 2a4 4 0 0 1 4 4v2h-8V6a4 4 0 0 1 4-4z"/><rect x="4" y="8" width="16" height="12" rx="2"/>
                <line x1="9" y1="13" x2="9" y2="16"/><line x1="15" y1="13" x2="15" y2="16"/>
              </svg> HF VLM
            </button>
          </div>

          <div v-if="ocrMode === 'cloud'" class="token-group">
            <label>OCR.space API Key <span class="optional">(pre-configured)</span></label>
            <input v-model="apiKey" type="password" placeholder="K85526706188957" :disabled="processing" class="token-input" />
          </div>

          <div v-if="ocrMode === 'gvision'" class="token-group">
            <label>Google Cloud Vision Key <span class="optional">(1000 free/mo)</span></label>
            <input v-model="gvisionKey" type="password" placeholder="AIza..." :disabled="processing" class="token-input" />
          </div>

          <div v-if="ocrMode === 'trocr'" class="token-group">
            <label>HuggingFace Token <span class="optional">(pre-configured)</span></label>
            <input v-model="hfToken" type="password" placeholder="hf_..." :disabled="processing" class="token-input" />
          </div>

          <label class="checkbox-label">
            <input type="checkbox" v-model="preprocessEnabled" :disabled="processing" />
            Advanced preprocessing (CLAHE + Sauvola + Deskew)
          </label>

          <button class="btn-run" :disabled="!imageData || processing" @click="runOCR">
            <svg v-if="!processing" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5">
              <polygon points="5 3 19 12 5 21 5 3"/>
            </svg>
            <span v-if="processing" class="spinner" />
            {{ processing ? 'Processing...' : 'Run OCR' }}
          </button>

          <div v-if="processing" class="progress-wrap">
            <div class="progress-bar"><div class="progress-fill" :style="{ width: progressPct + '%' }" /></div>
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

      <div class="right-panel">
        <div v-if="!ocrResult && !processing" class="empty-state">
          <svg width="64" height="64" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.2" opacity="0.25">
            <path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"/>
            <polyline points="14 2 14 8 20 8"/><line x1="16" y1="13" x2="8" y2="13"/>
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
.app { min-height: 100vh; display: flex; flex-direction: column; }
.header { background: white; border-bottom: 1px solid var(--border); padding: 0 24px; position: sticky; top: 0; z-index: 10; }
.header-inner { max-width: 1200px; margin: 0 auto; height: 60px; display: flex; align-items: center; justify-content: space-between; }
.logo { display: flex; align-items: center; gap: 10px; font-size: 1.1rem; font-weight: 700; color: var(--primary); }
.header-right { display: flex; align-items: center; gap: 12px; }
.model-badge { background: #eef2ff; color: var(--primary); font-size: 0.78rem; font-weight: 600; padding: 4px 10px; border-radius: 99px; border: 1px solid #c7d2fe; }
.btn-help { display: flex; align-items: center; gap: 6px; background: none; border: 1.5px solid var(--border); border-radius: 8px; padding: 6px 14px; font-size: 0.875rem; font-weight: 500; color: var(--text-muted); transition: all 0.15s; }
.btn-help:hover { border-color: var(--primary); color: var(--primary); }
.main { flex: 1; max-width: 1200px; margin: 0 auto; width: 100%; padding: 24px; display: grid; grid-template-columns: 400px 1fr; gap: 24px; align-items: start; }
.upload-zone { background: white; border: 2px dashed var(--border); border-radius: var(--radius); min-height: 220px; display: flex; align-items: center; justify-content: center; cursor: pointer; transition: all 0.2s; position: relative; overflow: hidden; }
.upload-zone:not(.has-image):hover, .upload-zone.drag-over { border-color: var(--primary); background: #eef2ff; }
.upload-placeholder { text-align: center; padding: 24px; display: flex; flex-direction: column; align-items: center; gap: 10px; }
.upload-title { font-weight: 600; font-size: 0.95rem; color: var(--text); }
.upload-hint { font-size: 0.8rem; color: var(--text-muted); }
.preview-img { width: 100%; height: 100%; object-fit: contain; max-height: 300px; padding: 8px; }
.btn-clear { position: absolute; top: 8px; right: 8px; background: rgba(0,0,0,0.55); color: white; border: none; border-radius: 50%; width: 26px; height: 26px; display: flex; align-items: center; justify-content: center; transition: background 0.15s; }
.btn-clear:hover { background: var(--error); }
.preview-section { background: white; border: 1px solid var(--border); border-radius: var(--radius); margin-top: 8px; overflow: hidden; }
.preview-header { display: flex; align-items: center; justify-content: space-between; padding: 8px 12px; border-bottom: 1px solid var(--border); }
.preview-label { font-size: 0.75rem; font-weight: 600; color: var(--text-muted); text-transform: uppercase; letter-spacing: 0.04em; }
.btn-toggle-preview { font-size: 0.75rem; color: var(--primary); background: none; border: none; }
.preview-img-small { width: 100%; max-height: 160px; object-fit: contain; padding: 6px; background: #f8fafc; }
.btn-show-preview { width: 100%; margin-top: 8px; background: none; border: 1px dashed var(--border); border-radius: var(--radius); padding: 8px; font-size: 0.8rem; color: var(--text-muted); transition: all 0.15s; }
.btn-show-preview:hover { border-color: var(--primary); color: var(--primary); }
.controls { background: white; border: 1px solid var(--border); border-radius: var(--radius); padding: 18px; margin-top: 16px; display: flex; flex-direction: column; gap: 14px; }
.engine-toggle { display: grid; grid-template-columns: 1fr 1fr 1fr 1fr; gap: 6px; background: #f1f5f9; border-radius: 8px; padding: 4px; }
.engine-btn { display: flex; align-items: center; justify-content: center; gap: 6px; background: none; border: none; border-radius: 6px; padding: 8px; font-size: 0.82rem; font-weight: 500; color: var(--text-muted); transition: all 0.15s; }
.engine-btn.active { background: white; color: var(--primary); box-shadow: 0 1px 3px rgba(0,0,0,0.08); font-weight: 600; }
.engine-btn:disabled { opacity: 0.5; }
.gv-btn.active { color: #ea4335; }
.hf-btn.active { color: #ff9d00; }
.token-group { display: flex; flex-direction: column; gap: 5px; }
.token-group label { font-size: 0.78rem; font-weight: 600; color: var(--text-muted); text-transform: uppercase; letter-spacing: 0.04em; }
.optional { font-weight: 400; text-transform: none; letter-spacing: 0; }
.token-input { border: 1.5px solid var(--border); border-radius: 7px; padding: 7px 10px; font-size: 0.875rem; color: var(--text); background: white; outline: none; transition: border-color 0.15s; width: 100%; box-sizing: border-box; }
.token-input:focus { border-color: var(--primary); }
.token-input:disabled { opacity: 0.5; }
.checkbox-label { display: flex; align-items: center; gap: 7px; font-size: 0.85rem; color: var(--text); cursor: pointer; }
.btn-run { display: flex; align-items: center; justify-content: center; gap: 8px; background: var(--primary); color: white; border: none; border-radius: 8px; padding: 11px; font-size: 0.95rem; font-weight: 600; transition: background 0.15s, opacity 0.15s; }
.btn-run:hover:not(:disabled) { background: var(--primary-dark); }
.btn-run:disabled { opacity: 0.5; cursor: not-allowed; }
.spinner { width: 16px; height: 16px; border: 2px solid rgba(255,255,255,0.3); border-top-color: white; border-radius: 50%; animation: spin 0.7s linear infinite; }
@keyframes spin { to { transform: rotate(360deg); } }
.progress-wrap { display: flex; flex-direction: column; gap: 5px; }
.progress-bar { height: 6px; background: var(--border); border-radius: 99px; overflow: hidden; }
.progress-fill { height: 100%; background: var(--primary); border-radius: 99px; transition: width 0.3s ease; }
.progress-status { font-size: 0.78rem; color: var(--text-muted); }
.error-msg { display: flex; align-items: center; gap: 8px; background: #fef2f2; border: 1px solid #fecaca; color: var(--error); border-radius: 8px; padding: 10px 14px; font-size: 0.875rem; margin-top: 8px; }
.right-panel { min-height: 400px; }
.empty-state { background: white; border: 1px solid var(--border); border-radius: var(--radius); min-height: 400px; display: flex; flex-direction: column; align-items: center; justify-content: center; gap: 14px; color: var(--text-muted); text-align: center; line-height: 1.6; font-size: 0.95rem; }
</style>
