<template>
  <div class="template-view">
    <header class="view-hero reveal">
      <span class="pill">Step 02</span>
      <h1>选择模板</h1>
      <p class="muted">挑选与你的主题匹配的视觉风格。</p>
    </header>

    <div class="template-toolbar reveal delay-1">
      <div class="template-toolbar__hint">AI推荐模板可在右侧更新。</div>
      <button class="button button--ghost" @click="loadTemplates">换一批模板</button>
    </div>

    <div class="template-grid reveal delay-2">
      <button
        v-for="template in templates"
        :key="template.id"
        type="button"
        :class="['template-card', { 'is-selected': template.id === templateId }]"
        @click="selectTemplate(template)"
      >
        <img :src="template.coverUrl + '?token=' + token" :alt="template.name || 'template'" />
        <div v-if="template.id === templateId" class="template-card__overlay">
          <span>已选择</span>
        </div>
      </button>
      <div v-if="templates.length === 0" class="empty-state template-empty">
        <div class="empty-state__title">模板加载中</div>
        <div>正在为你挑选推荐模板。</div>
      </div>
    </div>

    <div class="template-actions reveal delay-3">
      <button class="button button--primary" @click="nextStep">下一步：生成PPT</button>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, defineEmits } from 'vue'
const props = defineProps<{
  token: string
}>()

const templates = ref([] as any)
const templateId = ref('')
const $emit = defineEmits(['nextStep'])

async function loadTemplates() {
  const url = 'https://ppt-master.yfw.me/api/public/ppt/randomTemplates?apiKey=4u2Fo50Alk1ym2Os'
  const resp = await (await fetch(url, {
    method: 'POST',
    headers: {
      'token': props.token,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ page: 1, size: 28, filters: { type: 1 } })
  })).json()
  if (resp.code != 0) {
    alert('获取模板异常：' + resp.message)
    return
  }
  templates.value = resp.data || []
  if (templates.value.length > 0) {
    selectTemplate(templates.value[0])
  }
}

function selectTemplate(template: any) {
  if (!template) {
    return
  }
  templateId.value = template.id
  const src = template.coverUrl + '?token=' + props.token
  calcSubjectColor(src).then((color) => {
    if (color) {
      document.documentElement.style.setProperty('--template-accent', color)
    }
  })
}

async function calcSubjectColor(src: string) {
  let img = new Image()
  const loaded = await new Promise<boolean>((resolve) => {
    let eqOrigin =
      src.startsWith('data:') ||
      src.startsWith(document.location.origin) ||
      (src.startsWith('//') && (document.location.protocol + src).startsWith(document.location.origin))
    if (!eqOrigin) {
      img.crossOrigin = 'anonymous'
    }
    img.src = src
    img.onload = function () {
      resolve(true)
    }
    img.onerror = function () {
      resolve(false)
    }
  })
  if (!loaded || img.width === 0 || img.height === 0) {
    return null
  }
  let canvas = document.createElement('canvas')
  let ctx = canvas.getContext('2d')
  if (!ctx) {
    return null
  }
  canvas.width = img.width
  canvas.height = img.height
  ctx.drawImage(img, 0, 0)
  let imageData = ctx.getImageData(0, 0, canvas.width, canvas.height)
  let data = imageData.data
  let map = {} as any
  for (let i = 0; i < data.length; i += 4) {
    let r = data[i]
    let g = data[i + 1]
    let b = data[i + 2]
    let rgb = r + g + b
    if (rgb <= 50 || rgb >= 660) {
      // 忽略黑白
      continue
    }
    let color = `${r},${g},${b}`
    map[color] = (map[color] || 0) + 1
  }
  let valueMap = {} as any
  for (let k in map) {
    let v = map[k]
    let ks = valueMap[v]
    if (ks) {
      ks.push(k)
    } else {
      valueMap[v] = [k]
    }
  }
  let colors = [] as any
  let values = Object.values(map).sort() as any
  for (let i = values.length - 1; i >= 0; i--) {
    let ks = valueMap[values[i]]
    for (let j = 0; j < ks.length; j++) {
      colors.push(ks[j])
      if (colors.length >= 3) {
        break
      }
    }
    if (colors.length >= 3) {
      break
    }
  }
  if (!colors.length) {
    return null
  }
  return `rgb(${colors[0]})`
}

function nextStep() {
  if (!templateId.value) {
    alert('请选择模板')
    return
  }
  document.documentElement.style.removeProperty('--template-accent')
  $emit('nextStep', templateId.value)
}

loadTemplates()
</script>

<style scoped>
.template-view {
  display: flex;
  flex-direction: column;
  gap: 24px;
  flex: 1;
  min-height: 0;
  overflow: hidden;
  height: 100%;
}

.view-hero {
  display: grid;
  gap: 10px;
  flex-shrink: 0;
}

.view-hero h1 {
  font-size: 32px;
  font-weight: 700;
}

.template-toolbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 16px;
  flex-shrink: 0;
}

.template-toolbar__hint {
  color: var(--ink-500);
}

.template-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(220px, 1fr));
  gap: 18px;
  flex: 1;
  min-height: 0;
  overflow: auto;
  padding-right: 4px;
  align-content: start;
}

.template-card {
  position: relative;
  width: 100%;
  height: clamp(120px, 22vw, 120px);
  padding: 0;
  border: 1px solid var(--line);
  border-radius: var(--radius-lg);
  overflow: hidden;
  background: #fff;
  cursor: pointer;
  display: grid;
  place-items: center;
  transition: transform 0.2s var(--ease), box-shadow 0.2s var(--ease), border-color 0.2s var(--ease);
}

.template-card img {
  width: 100%;
  height: 100%;
  object-fit: contain;
  display: block;
  transition: transform 0.2s var(--ease);
}

.template-card:hover {
  transform: translateY(-2px);
  box-shadow: var(--shadow-sm);
  border-color: var(--accent-200);
}

.template-card:hover img {
  transform: scale(1.01);
}

.template-card.is-selected {
  border-color: var(--template-accent);
  box-shadow: 0 0 0 2px rgba(47, 154, 143, 0.18);
}

.template-card__overlay {
  position: absolute;
  inset: auto 12px 12px auto;
  background: rgba(255, 255, 255, 0.92);
  color: var(--template-accent);
  padding: 6px 10px;
  border-radius: var(--radius-pill);
  font-size: 12px;
  font-weight: 600;
  border: 1px solid rgba(47, 154, 143, 0.2);
}

.template-empty {
  grid-column: 1 / -1;
}

.template-actions {
  display: flex;
  justify-content: flex-end;
  flex-shrink: 0;
}

@media (max-width: 900px) {
  .template-toolbar {
    flex-direction: column;
    align-items: flex-start;
  }

  .template-actions {
    justify-content: flex-start;
  }
}

@media (max-width: 720px) {
  .view-hero h1 {
    font-size: 26px;
  }
}
</style>
