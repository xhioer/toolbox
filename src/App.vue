<template>
  <div class="page-bg">
    <div class="grid-overlay"></div>
    <main class="shell">
      <header class="hero-panel">
        <div class="brand">
          <p class="eyebrow">INDUSTRIAL TOOLBOX AI</p>
          <h1>工具箱缺失检测中控台</h1>
          <p class="sub">拍照检测 / 本地导入 / OCR 编号识别 / 缺失报警</p>
        </div>

        <div class="status-stack">
          <div class="clock">{{ nowText }}</div>
          <div class="signal-row">
            <span :class="['signal', busy ? 'warn' : 'ok']"></span>
            <span>{{ busy ? '检测中' : '待机' }}</span>
            <span :class="['signal', missingTotalItems > 0 ? 'warn' : 'idle']"></span>
            <span>{{ missingTotalItems > 0 ? '发现缺失' : '无报警' }}</span>
          </div>
        </div>
      </header>

      <section class="command-grid">
        <article class="panel command-panel">
          <div class="section-head">
            <div>
              <p class="section-kicker">PROFILE</p>
              <h2>规格选择</h2>
            </div>
            <span class="pill">{{ activeToolboxSpec.dieCount }} die</span>
          </div>

          <div class="segmented" role="group" aria-label="toolbox spec">
            <button
              v-for="spec in toolboxSpecs"
              :key="spec.type"
              type="button"
              :class="['seg-btn', toolboxSpec === spec.type ? 'active' : '']"
              :disabled="busy"
              @click="toolboxSpec = spec.type"
            >
              <i class="spec-mark" aria-hidden="true"></i>
              <strong>{{ spec.dieCount }} die</strong>
              <span>{{ spec.label }}</span>
            </button>
          </div>

        </article>

        <article class="panel command-panel">
          <div class="section-head">
            <div>
              <p class="section-kicker">CONTROL</p>
              <h2>检测操作</h2>
            </div>
            <span class="pill">CAM {{ cameraIndex }}</span>
          </div>

          <div class="control-line">
            <label for="camera-index">摄像头索引</label>
            <input id="camera-index" v-model.number="cameraIndex" type="number" min="0" />
          </div>

          <div class="action-row">
            <button class="btn primary" :disabled="busy" @click="runCamera">拍照并检测</button>
            <label class="file-pick">
              <input type="file" accept="image/*" @change="onChooseFile" />
              <span>{{ selectedFile ? selectedFile.name : '选择图片文件' }}</span>
            </label>
            <button class="btn" :disabled="busy" @click="runUpload">导入图片检测</button>
          </div>
        </article>
      </section>

      <section class="metrics">
        <article class="metric panel">
          <p>系统状态</p>
          <strong :class="busy ? 't-warn' : 't-ok'">{{ busy ? '检测中' : '待机' }}</strong>
          <span>{{ busy ? '模型推理进行中' : '等待操作' }}</span>
        </article>
        <article class="metric panel">
          <p>当前规格</p>
          <strong>{{ activeToolboxSpec.label }}</strong>
          <span>{{ activeToolboxSpec.dieCount }} 个 die 标准</span>
        </article>
        <article class="metric panel">
          <p>检测目标数</p>
          <strong>{{ detectionCount }}</strong>
          <span>当前推理框数量</span>
        </article>
        <article class="metric panel">
          <p>缺失总数</p>
          <strong :class="missingTotalItems > 0 ? 't-warn' : 't-ok'">{{ missingTotalItems }}</strong>
          <span>按所选规格统计</span>
        </article>
        <article class="metric panel wide">
          <p>检测耗时</p>
          <strong>{{ processingCost }}</strong>
          <span>单次完整流程</span>
        </article>
      </section>

      <section v-if="alarmText" :class="['alarm', alarmWarn ? 'warn' : 'ok']">
        <strong>{{ alarmWarn ? '报警' : '提示' }}</strong>
        <span>{{ alarmText }}</span>
      </section>

      <section class="vision-grid">
        <article class="panel image-card">
          <div class="section-head">
            <div>
              <p class="section-kicker">SOURCE</p>
              <h2>输入图像</h2>
            </div>
            <span class="pill">Live</span>
          </div>
          <div class="image-box">
            <img v-if="liveStreamUrl" :src="liveStreamUrl" alt="capture-live" />
            <p v-else>尚未输入图像</p>
            <div class="scan-line"></div>
          </div>
        </article>

        <article class="panel image-card">
          <div class="section-head">
            <div>
              <p class="section-kicker">INFERENCE</p>
              <h2>推理结果图</h2>
            </div>
            <span class="pill">Boxes</span>
          </div>
          <div class="image-box">
            <img v-if="annotatedImageUrl" :src="annotatedImageUrl" alt="annotated" />
            <p v-else>暂无检测结果</p>
            <div class="scan-line"></div>
          </div>
        </article>
      </section>

      <section class="data-grid">
        <article class="panel table-card">
          <div class="section-head compact">
            <div>
              <p class="section-kicker">MISSING</p>
              <h2>缺失明细</h2>
            </div>
          </div>
          <div class="table-wrap">
            <table class="table">
              <thead>
                <tr>
                  <th>零件</th>
                  <th>缺失数量</th>
                  <th>缺失编号</th>
                </tr>
              </thead>
              <tbody>
                <tr v-if="missingParts.length === 0">
                  <td colspan="3" class="t-ok">未检测到缺失</td>
                </tr>
                <tr v-for="item in missingParts" :key="`${item.class_name}-${item.missing_count}`">
                  <td>{{ item.class_name || '-' }}</td>
                  <td class="t-warn">{{ item.missing_count }}</td>
                  <td>{{ item.missing_codes?.length ? item.missing_codes.join(', ') : item.missing_code_hint || '-' }}</td>
                </tr>
              </tbody>
            </table>
          </div>
        </article>

        <article class="panel table-card">
          <div class="section-head compact">
            <div>
              <p class="section-kicker">SUMMARY</p>
              <h2>零件统计与 OCR</h2>
            </div>
          </div>
          <div class="table-wrap">
            <table class="table">
              <thead>
                <tr>
                  <th>零件</th>
                  <th>应有</th>
                  <th>检测到</th>
                  <th>缺失</th>
                  <th>OCR 文本</th>
                  <th>OCR 编号</th>
                </tr>
              </thead>
              <tbody>
                <tr v-if="partSummaries.length === 0">
                  <td colspan="6" class="t-muted">暂无数据</td>
                </tr>
                <tr v-for="row in partSummaries" :key="row.class_name">
                  <td>{{ row.class_name }}</td>
                  <td>{{ row.required_count }}</td>
                  <td>{{ row.found_count }}</td>
                  <td :class="row.missing_count > 0 ? 't-warn' : 't-ok'">{{ row.missing_count }}</td>
                  <td>{{ row.ocr_texts?.length ? row.ocr_texts.join(' | ') : '-' }}</td>
                  <td>{{ row.ocr_codes?.length ? row.ocr_codes.join(', ') : '-' }}</td>
                </tr>
              </tbody>
            </table>
          </div>
        </article>

        <article class="panel table-card full">
          <div class="section-head compact">
            <div>
              <p class="section-kicker">OCR DETAIL</p>
              <h2>检测项 OCR 明细</h2>
            </div>
          </div>
          <div class="table-wrap">
            <table class="table">
              <thead>
                <tr>
                  <th>#</th>
                  <th>零件</th>
                  <th>置信度</th>
                  <th>OCR 文本</th>
                  <th>OCR 编号</th>
                </tr>
              </thead>
              <tbody>
                <tr v-if="detections.length === 0">
                  <td colspan="5" class="t-muted">暂无数据</td>
                </tr>
                <tr v-for="(d, idx) in detections" :key="`${d.class_name}-${idx}`">
                  <td>{{ idx + 1 }}</td>
                  <td>{{ d.class_name || '-' }}</td>
                  <td>{{ typeof d.confidence === 'number' ? d.confidence.toFixed(3) : '-' }}</td>
                  <td>{{ d.ocr_texts?.length ? d.ocr_texts.join(' | ') : '-' }}</td>
                  <td>{{ d.ocr_codes?.length ? d.ocr_codes.join(', ') : '-' }}</td>
                </tr>
              </tbody>
            </table>
          </div>
        </article>
      </section>
    </main>
  </div>
</template>

<script setup>
import { computed, onMounted, onUnmounted, ref, watch } from 'vue';

const busy = ref(false);
const cameraIndex = ref(0);
const selectedFile = ref(null);
const toolboxSpec = ref('die4');

const toolboxSpecs = [
  { type: 'die4', label: '4 die 工具箱', dieCount: 4 },
  { type: 'die5', label: '5 die 工具箱', dieCount: 5 }
];

const activeToolboxSpec = computed(() => {
  return toolboxSpecs.find((spec) => spec.type === toolboxSpec.value) || toolboxSpecs[0];
});

const nowText = ref('');
const processingCost = ref('-');
const detectionCount = ref(0);
const missingTotalItems = ref(0);

const alarmText = ref('');
const alarmWarn = ref(false);

const annotatedImageUrl = ref('');
const missingParts = ref([]);
const partSummaries = ref([]);
const detections = ref([]);

let timer = null;
const liveStreamUrl = ref('');
const apiBase = (import.meta.env.VITE_API_BASE_URL || '').replace(/\/+$/, '');

function apiUrl(path) {
  if (!apiBase) return path;
  return `${apiBase}${path}`;
}

function withTs(url) {
  const sep = url.includes('?') ? '&' : '?';
  return `${url}${sep}t=${Date.now()}`;
}

function updateClock() {
  const d = new Date();
  nowText.value = d.toLocaleString('zh-CN', { hour12: false });
}

function playWebAlarm() {
  try {
    const ctx = new (window.AudioContext || window.webkitAudioContext)();
    const osc = ctx.createOscillator();
    const gain = ctx.createGain();
    osc.type = 'square';
    osc.frequency.value = 1040;
    gain.gain.value = 0.15;
    osc.connect(gain);
    gain.connect(ctx.destination);
    osc.start();
    setTimeout(() => osc.stop(), 280);
  } catch (e) {
    console.log('audio failed', e);
  }
}

function resetAlarm() {
  alarmText.value = '';
  alarmWarn.value = false;
}

function applyResult(data) {
  annotatedImageUrl.value = data.annotated_image_url ? withTs(apiUrl(data.annotated_image_url)) : '';

  missingParts.value = data.missing_parts || [];
  partSummaries.value = data.part_summaries || [];
  detections.value = data.detections || [];

  detectionCount.value = data.detection_count ?? 0;
  missingTotalItems.value = data.missing_total_items ?? 0;

  const ms = data.processing_time_ms ?? 0;
  const sec = data.processing_time_sec ?? ms / 1000;
  processingCost.value = `${ms} ms / ${Number(sec).toFixed(3)} s`;

  if (data.missing_status === 'missing_detected') {
    alarmWarn.value = true;
    alarmText.value = `检测到缺失，共 ${missingTotalItems.value} 个，请立即复核。`;
    playWebAlarm();
  } else {
    alarmWarn.value = false;
    alarmText.value = '状态正常，未检测到缺失。';
  }
}

function onChooseFile(e) {
  selectedFile.value = e.target.files?.[0] || null;
}

function resetLiveStream() {
  const idx = Number(cameraIndex.value || 0);
  liveStreamUrl.value = withTs(apiUrl(`/api/live-stream?camera_index=${idx}`));
}

function detectionSpecPayload() {
  return {
    toolbox_type: activeToolboxSpec.value.type,
    expected_die_count: activeToolboxSpec.value.dieCount
  };
}

async function runCamera() {
  busy.value = true;
  resetAlarm();
  try {
    const res = await fetch(apiUrl('/api/capture-and-detect'), {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        camera_index: Number(cameraIndex.value || 0),
        ...detectionSpecPayload()
      })
    });
    const data = await res.json();
    if (!res.ok) throw new Error(data.detail || '摄像头检测失败');
    applyResult(data);
  } catch (err) {
    alarmWarn.value = true;
    alarmText.value = `错误：${err.message}`;
    processingCost.value = '-';
    playWebAlarm();
  } finally {
    busy.value = false;
  }
}

async function runUpload() {
  if (!selectedFile.value) {
    alarmWarn.value = true;
    alarmText.value = '请先选择图片文件';
    return;
  }

  busy.value = true;
  resetAlarm();
  try {
    const fd = new FormData();
    fd.append('image', selectedFile.value);
    fd.append('toolbox_type', activeToolboxSpec.value.type);
    fd.append('expected_die_count', String(activeToolboxSpec.value.dieCount));

    const res = await fetch(apiUrl('/api/detect-upload'), {
      method: 'POST',
      body: fd
    });
    const data = await res.json();
    if (!res.ok) throw new Error(data.detail || '导入图片检测失败');
    applyResult(data);
  } catch (err) {
    alarmWarn.value = true;
    alarmText.value = `错误：${err.message}`;
    processingCost.value = '-';
    playWebAlarm();
  } finally {
    busy.value = false;
  }
}

onMounted(() => {
  updateClock();
  timer = window.setInterval(updateClock, 1000);
  resetLiveStream();
});

onUnmounted(() => {
  if (timer) window.clearInterval(timer);
});

watch(cameraIndex, () => {
  resetLiveStream();
});
</script>
