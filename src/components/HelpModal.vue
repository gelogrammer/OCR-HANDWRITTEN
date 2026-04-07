<script setup>
defineEmits(['close'])
</script>

<template>
  <div class="overlay" @click.self="$emit('close')">
    <div class="modal">
      <div class="modal-header">
        <h2>OCR Assistant Guide</h2>
        <button class="btn-close" @click="$emit('close')">
          <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5">
            <line x1="18" y1="6" x2="6" y2="18"/><line x1="6" y1="6" x2="18" y2="18"/>
          </svg>
        </button>
      </div>

      <div class="modal-body">
        <p class="intro">
          This app uses <strong>Tesseract.js</strong> (local, free, unlimited) and
          <strong>OCR.space</strong> (cloud API) to extract text from images.
          It includes automatic image preprocessing to handle colored backgrounds and handwriting.
        </p>

        <div class="steps">

          <div class="step">
            <div class="step-number">1</div>
            <div class="step-content">
              <h3>Choose your engine</h3>
              <p>
                <strong>Local (Tesseract.js)</strong> &mdash; Runs in your browser. Free, no API limits.
                Returns confidence scores and word-level analysis. Best for general use.<br>
                <strong>Cloud (OCR.space)</strong> &mdash; Uses a cloud API. Pre-configured API key included.
                Good for comparison or when local results are poor.
              </p>
            </div>
          </div>

          <div class="step">
            <div class="step-number">2</div>
            <div class="step-content">
              <h3>Set page segmentation (Local mode)</h3>
              <p>Choose the mode that matches your image content:</p>
              <table class="psm-table">
                <tr><td><code>Auto</code></td><td>General documents with mixed content</td></tr>
                <tr><td><code>Variable</code></td><td>Pages with multiple text blocks</td></tr>
                <tr><td><code>Single block</code></td><td>Tables, lists, crew rosters</td></tr>
                <tr><td><code>Single line</code></td><td>One line of text</td></tr>
                <tr><td><code>Single word</code></td><td>Single number or short text (e.g., "800000", "T60")</td></tr>
              </table>
            </div>
          </div>

          <div class="step">
            <div class="step-number">3</div>
            <div class="step-content">
              <h3>Image preprocessing</h3>
              <p>
                When enabled, the app automatically converts images to grayscale, sharpens edges,
                and applies <strong>Otsu adaptive thresholding</strong> to produce clean black-on-white text.
                This is critical for images with colored backgrounds (green, yellow, etc.).
                You can preview the preprocessed image before running OCR.
              </p>
            </div>
          </div>

          <div class="step">
            <div class="step-number">4</div>
            <div class="step-content">
              <h3>Interpret results</h3>
              <div class="conf-levels">
                <div class="conf-row high">
                  <span class="icon">HIGH</span>
                  <div>
                    <strong>70%+ confidence</strong>
                    <p>Text extracted reliably. No review needed.</p>
                  </div>
                </div>
                <div class="conf-row medium">
                  <span class="icon">MED</span>
                  <div>
                    <strong>40%&ndash;69% confidence</strong>
                    <p>Review recommended. Check flagged words in Word Analysis.</p>
                  </div>
                </div>
                <div class="conf-row low">
                  <span class="icon">LOW</span>
                  <div>
                    <strong>0%&ndash;39% confidence</strong>
                    <p>Try a different PSM mode, clearer image, or toggle preprocessing.</p>
                  </div>
                </div>
              </div>
            </div>
          </div>

        </div>

        <div class="tip">
          <strong>Pro tips:</strong> For handwritten crew lists, use <code>Single block</code> mode with preprocessing ON.
          For single numbers on colored backgrounds, use <code>Single word</code> mode.
          Compare Local vs Cloud results — sometimes one engine reads handwriting better than the other.
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.overlay {
  position: fixed;
  inset: 0;
  background: rgba(15, 23, 42, 0.45);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 100;
  padding: 24px;
}

.modal {
  background: white;
  border-radius: 14px;
  width: 100%;
  max-width: 620px;
  max-height: 88vh;
  display: flex;
  flex-direction: column;
  box-shadow: 0 20px 60px rgba(0,0,0,0.2);
}

.modal-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 20px 24px 16px;
  border-bottom: 1px solid var(--border);
  flex-shrink: 0;
}
.modal-header h2 {
  font-size: 1.1rem;
  font-weight: 700;
  color: var(--text);
}
.btn-close {
  background: none;
  border: none;
  color: var(--text-muted);
  padding: 4px;
  border-radius: 6px;
  display: flex;
  transition: color 0.15s;
}
.btn-close:hover { color: var(--text); }

.modal-body {
  overflow-y: auto;
  padding: 20px 24px 24px;
  display: flex;
  flex-direction: column;
  gap: 20px;
}

.intro {
  font-size: 0.9rem;
  color: var(--text-muted);
  line-height: 1.6;
}

/* Steps */
.steps { display: flex; flex-direction: column; gap: 20px; }

.step {
  display: flex;
  gap: 14px;
}
.step-number {
  flex-shrink: 0;
  width: 28px;
  height: 28px;
  background: var(--primary);
  color: white;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 0.8rem;
  font-weight: 700;
  margin-top: 2px;
}
.step-content h3 {
  font-size: 0.925rem;
  font-weight: 700;
  color: var(--text);
  margin-bottom: 6px;
}
.step-content p {
  font-size: 0.875rem;
  color: var(--text-muted);
  line-height: 1.6;
}

.psm-table {
  margin-top: 8px;
  border-collapse: collapse;
  font-size: 0.825rem;
  width: 100%;
}
.psm-table td {
  padding: 4px 10px 4px 0;
  color: var(--text-muted);
  vertical-align: top;
}
.psm-table td:first-child { width: 110px; }
code {
  background: #f1f5f9;
  border: 1px solid var(--border);
  border-radius: 4px;
  padding: 1px 6px;
  font-size: 0.82rem;
  color: var(--primary-dark);
}

/* Confidence levels */
.conf-levels { display: flex; flex-direction: column; gap: 8px; margin-top: 8px; }
.conf-row {
  display: flex;
  gap: 12px;
  align-items: flex-start;
  padding: 10px 12px;
  border-radius: 8px;
  border: 1px solid;
}
.conf-row.high   { background: #f0fdf4; border-color: #86efac; }
.conf-row.medium { background: #fffbeb; border-color: #fcd34d; }
.conf-row.low    { background: #fef2f2; border-color: #fca5a5; }
.conf-row .icon {
  font-size: 0.7rem;
  font-weight: 800;
  line-height: 1.4;
  padding: 2px 6px;
  border-radius: 4px;
  flex-shrink: 0;
}
.conf-row.high .icon { background: #86efac; color: #166534; }
.conf-row.medium .icon { background: #fcd34d; color: #92400e; }
.conf-row.low .icon { background: #fca5a5; color: #991b1b; }
.conf-row strong { font-size: 0.875rem; display: block; margin-bottom: 2px; }
.conf-row p { font-size: 0.82rem; color: var(--text-muted); margin: 0; }

/* Tip */
.tip {
  background: #eef2ff;
  border: 1px solid #c7d2fe;
  border-radius: 8px;
  padding: 12px 14px;
  font-size: 0.85rem;
  color: #3730a3;
  line-height: 1.5;
}
</style>
