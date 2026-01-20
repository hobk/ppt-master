<template>
  <div class="shell">
    <header class="shell__header reveal">
      <div class="brand">
        <div class="brand__logo">PPT</div>
        <div class="brand__text">
          <div class="brand__title">Master</div>
          <div class="brand__subtitle">一句话即可让AI为您生成美观的演示文稿</div>
        </div>
      </div>
      <div class="progress">
        <div class="progress__label">流程</div>
        <div class="progress__steps">
          <div
            v-for="item in steps"
            :key="item.id"
            :class="['progress__step', { 'is-active': item.id === step, 'is-done': item.id < step }]"
          >
            <span class="progress__index">{{ item.id }}</span>
            <span class="progress__text">{{ item.label }}</span>
          </div>
        </div>
      </div>
    </header>

    <main class="shell__main">
      <GenerateOutline v-if="step === 1" :token="token" @nextStep="handleOutline" />
      <SelectTemplate v-if="step === 2" :token="token" @nextStep="selectTemplate" />
      <GeneratePpt
        v-if="step === 3"
        :token="token"
        :templateId="templateId"
        :outline="outline"
        :dataUrl="dataUrl"
      />
    </main>

    <footer class="shell__footer">
      <span class="muted">Open API</span>
      <a class="shell__link"rel="noreferrer">PPT Master©️2026</a>
    </footer>
  </div>
</template>

<script setup lang="ts">
import { ref, watchEffect } from 'vue'
import GenerateOutline from './GenerateOutline.vue'
import SelectTemplate from './SelectTemplate.vue'
import GeneratePpt from './GeneratePpt.vue'

// ppt-master

// api key
const apiKey = '4u2Fo50Alk1ym2Os'
// 用户ID（数据隔离）
const uid = 'test'

const step = ref(1)
const outline = ref('')
const dataUrl = ref()
const templateId = ref('')
const token = ref(localStorage.getItem('token') || '')

const steps = [
  { id: 1, label: '生成大纲' },
  { id: 2, label: '选择模板' },
  { id: 3, label: '生成PPT' }
]

// 这里只是demo演示，创建token请在服务端调用
async function createApiToken() {
  const url = 'https://ppt-master.yfw.me/api/public/user/createApiToken?apiKey=4u2Fo50Alk1ym2Os'
  const resp = await (await fetch(url, {
    method: 'POST',
    headers: {
      'Api-Key': apiKey,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      uid,
      limit: null
    })
  })).json()
  if (resp.code != 0) {
    // alert('创建接口token异常：' + resp.message)
    return
  }
  token.value = resp.data.token
  watchEffect(() => {
    localStorage.setItem('token', token.value || '')
  })
}

function handleOutline(params: any) {
  step.value++
  outline.value = params.outline
  dataUrl.value = params.dataUrl
}

function selectTemplate(id: string) {
  step.value++
  templateId.value = id
}

createApiToken()
</script>

<style scoped>
.shell {
  height: 100%;
  display: flex;
  flex-direction: column;
  gap: 24px;
  padding: 32px 32px 24px;
  overflow: hidden;
}

.shell__header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 24px;
  padding: 20px 24px;
  border-radius: var(--radius-xl);
  background: rgba(255, 255, 255, 0.82);
  border: 1px solid var(--line);
  box-shadow: var(--shadow-sm);
  backdrop-filter: blur(12px);
}

.brand {
  display: flex;
  align-items: center;
  gap: 14px;
}

.brand__logo {
  width: 48px;
  height: 48px;
  border-radius: 14px;
  background: linear-gradient(135deg, #2f9a8f, #6ad1be);
  color: #fff;
  font-weight: 700;
  display: grid;
  place-items: center;
  letter-spacing: 0.12em;
}

.brand__title {
  font-size: 20px;
  font-weight: 700;
  letter-spacing: 0.02em;
}

.brand__subtitle {
  font-size: 13px;
  color: var(--ink-500);
}

.progress {
  display: flex;
  align-items: center;
  gap: 16px;
  flex-wrap: wrap;
}

.progress__label {
  font-size: 12px;
  text-transform: uppercase;
  letter-spacing: 0.2em;
  color: var(--ink-400);
}

.progress__steps {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
}

.progress__step {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  padding: 6px 10px;
  border-radius: var(--radius-pill);
  background: var(--surface-2);
  color: var(--ink-500);
  font-size: 13px;
  border: 1px solid transparent;
}

.progress__step.is-active {
  background: var(--accent-100);
  color: var(--ink-900);
  border-color: var(--accent-200);
}

.progress__step.is-done {
  background: rgba(47, 154, 143, 0.12);
  color: var(--accent-600);
}

.progress__index {
  width: 18px;
  height: 18px;
  border-radius: 50%;
  display: grid;
  place-items: center;
  background: #fff;
  font-size: 11px;
  font-weight: 700;
  border: 1px solid var(--line);
}

.shell__main {
  width: min(var(--max-width), 100%);
  margin: 0 auto;
  flex: 1;
  display: flex;
  min-height: 0;
  overflow: hidden;
}

.shell__footer {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  padding: 0 6px 8px;
  font-size: 13px;
}

.shell__link {
  color: var(--accent-600);
  font-weight: 600;
}

@media (max-width: 900px) {
  .shell {
    padding: 24px 18px 20px;
  }

  .shell__header {
    flex-direction: column;
    align-items: flex-start;
  }

  .progress {
    width: 100%;
  }
}

@media (max-width: 600px) {
  .shell__footer {
    flex-direction: column;
    align-items: flex-start;
  }
}
</style>
