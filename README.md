# Handwriting OCR Dashboard

A browser-based OCR tool built with Vue 3 and Vite. Upload an image of handwritten or printed text and extract it using the [OCR.space](https://ocr.space/) API (Engine 2).

## Features

- Drag-and-drop or click-to-upload image input (PNG, JPG, WEBP)
- OCR via OCR.space API — no server required, runs entirely in the browser
- Copy extracted text to clipboard
- Optional API key field (falls back to a demo key)
- Built-in usage guide modal

## Getting Started

### Prerequisites

- Node.js 18+

### Install & run

```bash
npm install
npm run dev
```

Open `http://localhost:5173` in your browser.

### Build for production

```bash
npm run build
```

## Usage

1. Drop an image onto the upload zone (or click to browse)
2. Optionally enter your [OCR.space free API key](https://ocr.space/ocrapi/freekey) — leave blank to use the demo key
3. Click **Run OCR**
4. Copy the extracted text from the results panel

> For best results, use a clear, high-contrast image. Single lines of handwriting tend to give the most accurate output.

## Tech Stack

| | |
|---|---|
| Framework | Vue 3 (Composition API) |
| Build tool | Vite 5 |
| OCR engine | OCR.space API — Engine 2 |
