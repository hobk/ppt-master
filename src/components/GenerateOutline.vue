<template>
  <div class="outline-view">
    <header class="view-hero reveal">
      <span class="pill">Step 01</span>
      <h1>生成大纲</h1>
      <p class="muted">输入主题、文本或文件，自动生成结构化大纲。</p>
    </header>

    <div class="view-grid">
      <section class="card form-card reveal delay-1">
        <div class="card__title">输入内容</div>
        <div class="field">
          <div class="field__label">输入方式</div>
          <div class="segmented">
            <button type="button" :class="{ 'is-active': selectType === 'subject' }" @click="setType('subject')">
              主题
            </button>
            <button type="button" :class="{ 'is-active': selectType === 'text' }" @click="setType('text')">
              文本
            </button>
          </div>
        </div>

        <div v-if="selectType === 'subject'" class="field">
          <div class="field__label">主题</div>
          <input v-model="subject" class="input" placeholder="例如：2024 产品发布会" maxlength="20" />
        </div>

        <div v-else-if="selectType === 'text'" class="field">
          <div class="field__label">内容</div>
          <textarea v-model="text" class="textarea" placeholder="粘贴你的内容或大段描述" rows="6" maxlength="6000"></textarea>
        </div>

        <div v-else class="field">
          <div class="field__label">上传文件</div>
          <label class="upload">
            <input
              id="input_file"
              class="upload__input"
              type="file"
              accept=".doc, .docx, .xls, .xlsx, .pdf, .ppt, .pptx, .txt"
            />
            <div class="upload__body">
              <div class="upload__title">拖拽文件到此或点击上传</div>
              <div class="upload__desc">支持 doc/docx/xls/xlsx/pdf/ppt/pptx/txt</div>
            </div>
          </label>
        </div>

        <div class="form-actions">
          <button class="button button--primary" :disabled="genStatus !== 0" @click="generateOutline">生成大纲</button>
          <span class="form-tip">生成后可在右侧预览并编辑</span>
        </div>
      </section>

      <section class="card preview-card reveal delay-2">
        <div class="card__title">大纲预览</div>
        <div ref="previewScroll" class="preview-scroll">
          <div v-if="genStatus === 0" class="empty-state">
            <div class="empty-state__title">等待生成</div>
            <div>这里会展示生成中的实时内容，并支持编辑。</div>
          </div>
          <div v-else-if="genStatus === 1" v-html="outlineHtml" class="markdown preview-markdown"></div>
          <div v-else class="editor-stack">
            <div class="editor-hint">可直接修改章节标题与页面标题。</div>
            <OutlineEdit v-if="outlineTree" :outlineTree="outlineTree" @update="updateOutline" />
          </div>
        </div>
        <div v-if="genStatus === 2" class="preview-actions">
          <button class="button button--primary" @click="nextStep">下一步：选择模板</button>
        </div>
      </section>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, defineEmits, nextTick } from 'vue'
import OutlineEdit from './OutlineEdit.vue'
import { marked } from 'marked'
import { SSE } from '../utils/sse.js'

const props = defineProps<{
  token: string
}>()

const selectType = ref('subject')
const subject = ref('')
const text = ref('')
const dataUrl = ref()

// 生成状态: 0未开始 1生成中 2已完成
const genStatus = ref(0)
const outline = ref('')
const outlineTree = ref()
const outlineHtml = ref('')
const previewScroll = ref<HTMLElement | null>(null)
const $emit = defineEmits(['nextStep'])

marked.setOptions({
  renderer: new marked.Renderer(),
  gfm: true,
  async: false,
  breaks: false,
  pedantic: false,
  silent: true
})

function setType(type: string) {
  selectType.value = type
}

function parseFileData(formData: FormData) {
  let url = 'https://ppt-master.yfw.me/api/public/ppt/parseFileData'
  let xhr = new XMLHttpRequest()
  xhr.open('POST', url, false)
  xhr.setRequestHeader('token', props.token)
  xhr.send(formData)
  if (xhr.readyState === 4) {
    if (xhr.status === 200) {
      let resp = JSON.parse(xhr.responseText)
      if (resp.code != 0) {
        alert('解析文件异常：' + resp.message)
        return null
      }
      return resp.data.dataUrl
    } else {
      alert('解析文件网络异常, httpStatus: ' + xhr.status)
      return null
    }
  }
}

function generateOutline() {
  if (genStatus.value != 0) {
    return
  }
  genStatus.value = 1
  outlineHtml.value = '<h3>正在生成中，请稍后....</h3>'
  const inputData = {} as any
  if (selectType.value == 'subject') {
    // 根据主题
    if (!subject.value) {
      alert('请输入主题')
      genStatus.value = 0
      return
    }
    inputData.subject = subject.value
  } else if (selectType.value == 'text') {
    // 根据内容
    if (!text.value) {
      alert('请输入内容')
      genStatus.value = 0
      return
    }
    const formData = new FormData()
    formData.append('content', text.value)
    inputData.dataUrl = parseFileData(formData)
  } else if (selectType.value == 'file') {
    // 根据文件
    const file = (document.getElementById('input_file') as any)?.files[0]
    if (!file) {
      alert('请选择文件')
      genStatus.value = 0
      return
    }
    const formData = new FormData()
    formData.append('file', file)
    inputData.dataUrl = parseFileData(formData)
  }
  if (!inputData.subject && !inputData.dataUrl) {
    genStatus.value = 0
    return
  }
  genStatus.value = 1
  dataUrl.value = inputData.dataUrl
  const url = 'https://ppt-master.yfw.me/api/public/ppt/generateOutline'
  var source = new SSE(url, {
    method: 'POST',
    headers: {
      'token': props.token,
      'Cache-Control': 'no-cache',
      'Content-Type': 'application/json'
    },
    payload: JSON.stringify(inputData),
  }) as any
  source.onmessage = function (data: any) {
    let json = JSON.parse(data.data)
    if (json.status == -1) {
      alert('生成大纲异常：' + json.error)
      genStatus.value = 0
      return
    }
    if (json.status == 4 && json.result) {
      outlineTree.value = json.result
    } else {
      outline.value += json.text
      outlineHtml.value = marked.parse(outline.value.replace('```markdown', '').replace(/```/g, '')) as string
      nextTick(() => {
        if (previewScroll.value) {
          previewScroll.value.scrollTop = previewScroll.value.scrollHeight
        }
      })
    }
  }
  source.onend = function (data: any) {
    if (data.data.startsWith('{') && data.data.endsWith('}')) {
      let json = JSON.parse(data.data)
      if (json.code != 0) {
        alert('生成大纲异常：' + json.message)
        genStatus.value = 0
        return
      }
    }
    genStatus.value = 2
  }
  source.onerror = function (err: any) {
    console.error('生成大纲异常', err)
    alert('生成大纲异常')
    genStatus.value = 0
  }
  source.stream()
}

function updateOutline(md: any) {
  outline.value = md + ''
}

function nextStep() {
  $emit('nextStep', { outline: outline.value, dataUrl: dataUrl.value as string })
}
</script>

<style scoped>
.outline-view {
  display: flex;
  flex-direction: column;
  gap: 24px;
  flex: 1;
  min-height: 0;
  overflow: hidden;
}

.view-hero {
  display: grid;
  gap: 10px;
}

.view-hero h1 {
  font-size: 32px;
  font-weight: 700;
}

.view-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  grid-auto-rows: minmax(0, 1fr);
  gap: 24px;
  align-items: stretch;
  flex: 1;
  min-height: 0;
}

.form-card {
  display: flex;
  flex-direction: column;
  min-height: 0;
  overflow: auto;
}

.form-actions {
  display: flex;
  align-items: center;
  gap: 14px;
  margin-top: 6px;
}

.form-tip {
  font-size: 13px;
  color: var(--ink-400);
}

.upload {
  display: grid;
  place-items: center;
  padding: 18px;
  border-radius: var(--radius-md);
  border: 1px dashed var(--line);
  background: var(--surface-2);
  cursor: pointer;
  transition: border-color 0.2s var(--ease), background 0.2s var(--ease);
}

.upload:hover {
  border-color: var(--accent-400);
  background: #f1fbf8;
}

.upload__input {
  display: none;
}

.upload__body {
  display: grid;
  gap: 6px;
  text-align: center;
}

.upload__title {
  font-weight: 600;
  color: var(--ink-700);
}

.upload__desc {
  font-size: 13px;
  color: var(--ink-400);
}

.preview-card {
  display: flex;
  flex-direction: column;
  gap: 16px;
  min-height: 0;
  overflow: hidden;
}

.preview-scroll {
  flex: 1;
  min-height: 0;
  overflow: auto;
  padding-right: 6px;
}

.preview-markdown {
  padding-right: 6px;
}

.editor-stack {
  display: grid;
  gap: 12px;
  min-height: 0;
}

.editor-hint {
  font-size: 13px;
  color: var(--ink-400);
}

.preview-actions {
  margin-top: auto;
}

@media (max-width: 980px) {
  .view-grid {
    grid-template-columns: 1fr;
  }
}

@media (max-width: 720px) {
  .view-hero h1 {
    font-size: 26px;
  }

  .form-actions {
    flex-direction: column;
    align-items: flex-start;
  }
}
</style>
