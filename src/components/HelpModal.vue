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
          This dashboard uses <strong>Tesseract.js</strong> to extract and interpret text from
          handwritten images. Follow the steps below for best results.
        </p>

        <div class="steps">

          <div class="step">
            <div class="step-number">1</div>
            <div class="step-content">
              <h3>Assess the image</h3>
              <p>
                Check that your image is clear, well-lit, and high-contrast. For best results
                scan at <strong>300 DPI minimum</strong>. If the confidence score comes back
                below 50%, try re-uploading a higher quality scan.
              </p>
            </div>
          </div>

          <div class="step">
            <div class="step-number">2</div>
            <div class="step-content">
              <h3>Configure settings</h3>
              <p>
                Select the correct <strong>language</strong> and choose a
                <strong>page segmentation mode</strong> that matches your document layout:
              </p>
              <table class="psm-table">
                <tr><td><code>3</code></td><td>Auto — works for most documents</td></tr>
                <tr><td><code>6</code></td><td>Single block of text</td></tr>
                <tr><td><code>7</code></td><td>Single line</td></tr>
                <tr><td><code>8</code></td><td>Single word</td></tr>
                <tr><td><code>13</code></td><td>Raw line (no processing)</td></tr>
              </table>
            </div>
          </div>

          <div class="step">
            <div class="step-number">3</div>
            <div class="step-content">
              <h3>Interpret confidence scores</h3>
              <div class="conf-levels">
                <div class="conf-row high">
                  <span class="icon">✅</span>
                  <div>
                    <strong>70%–100% — High confidence</strong>
                    <p>Text extracted reliably. No review needed.</p>
                  </div>
                </div>
                <div class="conf-row medium">
                  <span class="icon">⚠️</span>
                  <div>
                    <strong>40%–69% — Medium confidence</strong>
                    <p>Review recommended. Check flagged words in the Word Analysis panel.</p>
                  </div>
                </div>
                <div class="conf-row low">
                  <span class="icon">❌</span>
                  <div>
                    <strong>0%–39% — Low confidence</strong>
                    <p>Re-upload a clearer image. Ensure good lighting and no motion blur.</p>
                  </div>
                </div>
              </div>
            </div>
          </div>

          <div class="step">
            <div class="step-number">4</div>
            <div class="step-content">
              <h3>Review flagged words</h3>
              <p>
                When confidence is below 70%, the <strong>Word Analysis</strong> section highlights
                every word by confidence level. The <strong>Flagged Words</strong> table lists the
                most uncertain words — review these manually to correct any misreads.
              </p>
            </div>
          </div>

        </div>

        <div class="tip">
          <strong>Tip:</strong> For handwritten notes, mode <code>6</code> (Single Block) or
          <code>7</code> (Single Line) often gives better results than the default Auto mode.
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
.psm-table td:first-child { width: 40px; }
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
.conf-row .icon { font-size: 1.1rem; line-height: 1.4; }
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
