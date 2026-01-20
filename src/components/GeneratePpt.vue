<template>
  <div class="ppt-view">
    <header class="ppt-bar reveal">
      <div class="ppt-status">
        <div class="ppt-status__label">生成状态</div>
        <div class="ppt-status__text">
          <span>{{ gening ? descMsg : '生成完成，可预览与下载' }}</span>
          <span v-if="gening" class="ppt-status__time">{{ descTime }}秒</span>
        </div>
      </div>
      <div class="ppt-actions">
        <button v-if="!gening && pptId" class="button button--primary" @click="downloadPptx">下载PPTX</button>
      </div>
    </header>

    <div class="ppt-content reveal delay-1">
      <aside class="ppt-thumbs">
        <div class="ppt-thumbs__title">页面</div>
        <div class="ppt-thumbs__list">
          <button
            v-for="(page, index) in pages"
            :key="index"
            type="button"
            :class="['thumb', { 'is-active': currentIdx === index }]"
            @click="drawPptx(index)"
          >
            <div class="thumb__index">{{ index + 1 }}</div>
            <canvas
              :ref="(el) => canvasList[index] = el"
              width="288"
              height="162"
              class="thumb__canvas"
            />
          </button>
          <div v-if="gening && currentIdx > 0" class="thumb thumb--ghost">
            <div class="thumb__index">{{ currentIdx + 2 }}</div>
            <div class="thumb__loading">生成中...</div>
          </div>
        </div>
      </aside>

      <main class="ppt-preview" ref="preview">
        <svg ref="svg" class="ppt-canvas"></svg>
      </main>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, nextTick } from 'vue'
import pako from 'pako'
import base64js from 'base64-js'
import { SSE } from '../utils/sse.js'
import { Ppt2Svg } from '../utils/ppt2svg.js'
import { Ppt2Canvas } from '../utils/ppt2canvas.js'

const props = defineProps<{
  token: string,
  templateId: string,
  outline: string,
  dataUrl: string
}>()

const gening = ref(false)
const descTime = ref(0)
const descMsg = ref('正在生成中，请稍后...')
const svg = ref()
const preview = ref<HTMLElement | null>(null)
const pptId = ref('')
const pages = ref([])
const canvasList = ref([] as any)
const currentIdx = ref(0)

var pptxObj = null as any
var painter = null as any

function generatePptx(templateId: string, outline: string, dataUrl: string) {
  var count = 0
  var timer = setInterval(() => {
    count = count + 1
    descTime.value = count
  }, 1000)
  gening.value = true
  const url = 'https://ppt-master.yfw.me/api/public/ppt/generateContent?apiKey=4u2Fo50Alk1ym2Os'
  var source = new SSE(url, {
    method: 'POST',
    headers: {
      'token': props.token,
      'Cache-Control': 'no-cache',
      'Content-Type': 'application/json',
    },
    payload: JSON.stringify({ asyncGenPptx: true, templateId, outlineMarkdown: outline, dataUrl }),
  }) as any
  source.onmessage = function (data: any) {
    let json = JSON.parse(data.data)
    if (json.status == -1) {
      alert('生成PPT异常：' + json.error)
      return
    }
    if (json.pptId) {
      descMsg.value = `正在生成中，进度 ${json.current}/${json.total}，请稍后...`
      asyncGenPptxInfo(json.pptId)
    }
  }
  source.onend = function (data: any) {
    if (data.data.startsWith('{') && data.data.endsWith('}')) {
      let json = JSON.parse(data.data)
      if (json.code != 0) {
        alert('生成PPT异常：' + json.message)
        return
      }
    }
    clearInterval(timer)
    gening.value = false
    descMsg.value = '正在生成中，请稍后...'
    setTimeout(() => {
      drawPptxList(0, false)
    }, 200)
  }
  source.onerror = function (err: any) {
    clearInterval(timer)
    console.error('生成内容异常', err)
    alert('生成内容异常')
  }
  source.stream()
}

function asyncGenPptxInfo(id: string) {
  pptId.value = id
  let url = `https://ppt-master.yfw.me/api/public/ppt/asyncPptInfo?pptId=${id}&apiKey=4u2Fo50Alk1ym2Os`
  let xhr = new XMLHttpRequest()
  xhr.open('GET', url, true)
  xhr.setRequestHeader('token', props.token)
  xhr.send()
  xhr.onload = function () {
    if (this.status === 200) {
      let resp = JSON.parse(this.responseText)
      let gzipBase64 = resp.data.pptxProperty
      let gzip = base64js.toByteArray(gzipBase64)
      let json = pako.ungzip(gzip, { to: 'string' })
      let _pptxObj = JSON.parse(json)
      pptxObj = _pptxObj
      drawPptxList(resp.data.current - 1, true)
    }
  }
  xhr.onerror = function (e) {
    console.error(e)
  }
}
async function drawPptxList(_idx?: number, asyncGen?: boolean) {
  let idx = _idx || 0
  currentIdx.value = idx || 0
  if (_idx == null || asyncGen) {
    let _pages = [] as any
    for (let i = 0; i < pptxObj.pages.length; i++) {
      if (asyncGen && i > idx) {
        break
      }
      _pages.push(pptxObj.pages[i])
    }
    pages.value = _pages
    nextTick(async () => {
      if (asyncGen) {
        canvasList.value[idx].scrollIntoView(true)
      }
      for (let i = 0; i < _pages.length; i++) {
        let imgCanvas = canvasList.value[i]
        if (!imgCanvas) {
          continue
        }
        try {
          let _ppt2Canvas = new Ppt2Canvas(imgCanvas)
          await _ppt2Canvas.drawPptx(pptxObj, i)
        } catch (e) {
          console.log('渲染第' + (i + 1) + '页封面异常', e)
        }
      }
    })
  } else {
    pages.value = pptxObj.pages || []
    nextTick(async () => {
      try {
        let imgCanvas = canvasList.value[idx]
        let _ppt2Canvas = new Ppt2Canvas(imgCanvas)
        await _ppt2Canvas.drawPptx(pptxObj, idx)
      } catch (e) {
        console.log('渲染第' + (idx + 1) + '页封面异常', e)
      }
    })
  }

  drawPptx(idx)
}

function drawPptx(idx: number) {
  currentIdx.value = idx
  painter.drawPptx(pptxObj, idx)
  if (idx == 0) {
    nextTick(async () => {
      canvasList.value[idx].scrollIntoView(true)
    })
  }
}

function downloadPptx() {
  let url = 'https://ppt-master.yfw.me/api/public/ppt/downloadPptx?apiKey=4u2Fo50Alk1ym2Os'
  let xhr = new XMLHttpRequest()
  xhr.open('POST', url, true)
  xhr.setRequestHeader('token', props.token)
  xhr.setRequestHeader('Content-Type', 'application/json')
  xhr.send(JSON.stringify({ id: pptId.value }))
  xhr.onload = function () {
    if (this.status === 200) {
      let resp = JSON.parse(this.responseText)
      let fileUrl = resp.data.fileUrl
      let a = document.createElement('a')
      a.href = fileUrl
      a.download = (resp.data.subject || 'download') + '.pptx'
      a.click()
    }
  }
  xhr.onerror = function (e) {
    console.error(e)
  }
}

function loadById(id: string) {
  gening.value = false
  pptId.value = id
  let url = 'https://ppt-master.yfw.me/api/public/ppt/loadPptx?id=' + id + '?apiKey=4u2Fo50Alk1ym2Os'
  let xhr = new XMLHttpRequest()
  xhr.open('GET', url, true)
  xhr.setRequestHeader('token', props.token)
  xhr.send()
  xhr.onload = function () {
    if (this.status === 200) {
      let resp = JSON.parse(this.responseText)
      if (resp.code != 0) {
        alert(resp.message)
        return
      }
      let pptInfo = resp.data.pptInfo
      let gzipBase64 = pptInfo.pptxProperty
      let gzip = base64js.toByteArray(gzipBase64)
      let json = pako.ungzip(gzip, { to: 'string' })
      pptInfo.pptxProperty = JSON.parse(json)
      pptxObj = pptInfo.pptxProperty
      drawPptxList()
    }
  }
  xhr.onerror = function (e) {
    console.error(e)
  }
}

function resetSize() {
  const previewEl = preview.value
  if (!previewEl) {
    return
  }
  const available = previewEl.clientWidth || document.body.clientWidth
  let width = Math.max(Math.min(available - 40, 1400), 360)
  painter.resetSize(width, width * 0.5625)
}

onMounted(() => {
  // svg
  painter = new Ppt2Svg(svg.value) as any
  painter.setMode('edit')

  var mTimer = 0
  window.addEventListener('resize', function () {
    mTimer && clearTimeout(mTimer)
    mTimer = setTimeout(() => {
      resetSize()
    }, 50)
  })

  resetSize()

  let _pptId = new URLSearchParams(window.location.search).get('pptId')
  if (_pptId) {
    loadById(_pptId)
  } else {
    generatePptx(props.templateId, props.outline, props.dataUrl)
  }
})
</script>

<style scoped>
.ppt-view {
  display: flex;
  flex-direction: column;
  gap: 20px;
  flex: 1;
  min-height: 0;
  overflow: hidden;
}

.ppt-bar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 16px;
  padding: 16px 20px;
  border-radius: var(--radius-lg);
  background: rgba(255, 255, 255, 0.85);
  border: 1px solid var(--line);
  box-shadow: var(--shadow-sm);
}

.ppt-status {
  display: grid;
  gap: 4px;
}

.ppt-status__label {
  font-size: 12px;
  text-transform: uppercase;
  letter-spacing: 0.18em;
  color: var(--ink-400);
}

.ppt-status__text {
  display: flex;
  align-items: center;
  gap: 12px;
  color: var(--ink-700);
  font-weight: 600;
}

.ppt-status__time {
  font-size: 12px;
  padding: 4px 10px;
  border-radius: var(--radius-pill);
  background: var(--accent-100);
  color: var(--accent-600);
}

.ppt-content {
  display: grid;
  grid-template-columns: 220px minmax(0, 1fr);
  gap: 20px;
  align-items: stretch;
  flex: 1;
  min-height: 0;
}

.ppt-thumbs {
  background: var(--surface);
  border-radius: var(--radius-lg);
  border: 1px solid var(--line);
  box-shadow: var(--shadow-sm);
  padding: 16px;
  display: flex;
  flex-direction: column;
  min-height: 0;
}

.ppt-thumbs__title {
  font-weight: 600;
  color: var(--ink-700);
  margin-bottom: 12px;
}

.ppt-thumbs__list {
  display: flex;
  flex-direction: column;
  gap: 12px;
  flex: 1;
  min-height: 0;
  overflow: auto;
  padding-right: 4px;
}

.thumb {
  display: grid;
  grid-template-columns: 24px 1fr;
  align-items: center;
  gap: 10px;
  background: transparent;
  border: 1px solid transparent;
  border-radius: 12px;
  padding: 8px;
  cursor: pointer;
  transition: all 0.2s var(--ease);
}

.thumb__index {
  font-size: 12px;
  color: var(--ink-400);
  text-align: right;
}

.thumb__canvas {
  width: 100%;
  height: auto;
  border-radius: 10px;
  border: 1px solid var(--line);
  background: #fff;
}

.thumb.is-active {
  background: var(--accent-100);
  border-color: var(--accent-200);
}

.thumb.is-active .thumb__canvas {
  border-color: var(--accent-400);
}

.thumb--ghost {
  border: 1px dashed var(--line);
  padding: 10px;
  align-items: center;
}

.thumb__loading {
  height: 90px;
  border-radius: 10px;
  background: var(--surface-2);
  display: grid;
  place-items: center;
  font-size: 12px;
  color: var(--ink-400);
}

.ppt-preview {
  background: var(--surface);
  border-radius: var(--radius-lg);
  border: 1px solid var(--line);
  box-shadow: var(--shadow-sm);
  padding: 20px;
  display: grid;
  place-items: center;
  min-height: 0;
  overflow: auto;
}

.ppt-canvas {
  border-radius: 12px;
  border: 1px solid var(--line);
  background: #fff;
  box-shadow: var(--shadow-md);
}

@media (max-width: 1100px) {
  .ppt-content {
    grid-template-columns: 200px minmax(0, 1fr);
  }
}

@media (max-width: 900px) {
  .ppt-content {
    grid-template-columns: 1fr;
  }

  .ppt-thumbs {
    position: static;
  }

  .ppt-thumbs__list {
    flex-direction: row;
    overflow-x: auto;
    overflow-y: hidden;
    max-height: none;
  }

  .thumb {
    grid-template-columns: 1fr;
    justify-items: center;
    min-width: 160px;
  }

  .thumb__index {
    text-align: center;
  }
}

@media (max-width: 720px) {
  .ppt-bar {
    flex-direction: column;
    align-items: flex-start;
  }

  .ppt-preview {
    padding: 12px;
  }
}
</style>
