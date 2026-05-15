# Toolbox Missing Detection Frontend

工具箱缺失检测中控台前端。项目用于展示工业工具箱检测流程，包括摄像头采集、本地图片导入、4 die / 5 die 规格切换、检测结果图展示、缺失明细统计和 OCR 编号信息。

> 当前仓库是前端项目。实际图像检测、OCR、缺失规则判断和相机采集逻辑由后端接口提供。

## Features

- Vue 3 + Vite 单页前端
- 4 die / 5 die 工具箱规格切换
- 摄像头拍照检测入口
- 本地图片上传检测入口
- 输入图像与推理结果图并排展示
- 缺失数量、检测目标数、检测耗时等状态指标
- 缺失明细、零件统计、OCR 明细表格
- Vite dev proxy 转发 `/api` 和 `/runtime` 到后端

## Tech Stack

- Vue 3
- Vite
- CSS

## Project Structure

```text
.
├── index.html
├── package.json
├── vite.config.js
├── .env.example
└── src
    ├── App.vue
    ├── main.js
    └── styles.css
```

## Getting Started

Install dependencies:

```bash
npm install
```

Start the development server:

```bash
npm run dev
```

Default frontend URL:

```text
http://127.0.0.1:5173/
```

## Backend API

In development, Vite proxies these paths to `http://127.0.0.1:8000`:

- `/api`
- `/runtime`

Proxy configuration is in `vite.config.js`.

You can also set a backend base URL through `.env`:

```bash
VITE_API_BASE_URL=http://127.0.0.1:8000
```

If `VITE_API_BASE_URL` is empty, frontend requests use relative paths and are handled by the Vite proxy in development.

## Expected API Endpoints

### Live Stream

```http
GET /api/live-stream?camera_index=0
```

Used as the live input image source.

### Capture and Detect

```http
POST /api/capture-and-detect
Content-Type: application/json
```

Request body:

```json
{
  "camera_index": 0,
  "toolbox_type": "die4",
  "expected_die_count": 4
}
```

`toolbox_type` can be:

- `die4`
- `die5`

### Upload and Detect

```http
POST /api/detect-upload
Content-Type: multipart/form-data
```

Form fields:

```text
image=<file>
toolbox_type=die4
expected_die_count=4
```

## Expected Detection Response

The frontend expects a response with fields like:

```json
{
  "capture_image_url": "/runtime/capture.jpg",
  "annotated_image_url": "/runtime/annotated.jpg",
  "missing_status": "missing_detected",
  "missing_total_items": 1,
  "detection_count": 4,
  "processing_time_ms": 320,
  "processing_time_sec": 0.32,
  "missing_parts": [
    {
      "class_name": "die",
      "missing_count": 1,
      "missing_codes": ["D04"]
    }
  ],
  "part_summaries": [
    {
      "class_name": "die",
      "required_count": 5,
      "found_count": 4,
      "missing_count": 1,
      "ocr_texts": ["D01", "D02", "D03", "D05"],
      "ocr_codes": ["D01", "D02", "D03", "D05"]
    }
  ],
  "detections": [
    {
      "class_name": "die",
      "confidence": 0.982,
      "ocr_texts": ["D01"],
      "ocr_codes": ["D01"]
    }
  ]
}
```

## Build

Create a production build:

```bash
npm run build
```

Preview the production build locally:

```bash
npm run preview
```

## Notes

- The frontend does not include model inference code.
- The selected toolbox specification is sent to the backend as `toolbox_type` and `expected_die_count`.
- The backend should use these fields to apply different missing-detection rules for 4 die and 5 die toolboxes.
