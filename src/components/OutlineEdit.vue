<template>
  <div class="outline-editor">
    <div class="outline-editor__subject">{{ props.outlineTree.name }}</div>
    <div v-for="(chapter, chapterIdx) in props.outlineTree.children" :key="version + '_' + chapterIdx">
      <div class="outline-editor__row outline-editor__chapter">
        <div class="outline-editor__badge">第{{ chapterIdx + 1 }}章</div>
        <div class="outline-editor__input">
          <input v-model="chapter.name" @change="update" />
        </div>
        <div class="outline-editor__actions">
          <button type="button" class="outline-editor__action" title="在上方新增同级" @click="operate(props.outlineTree.children, chapterIdx, 1)">=+</button>
          <button type="button" class="outline-editor__action" title="在下方新增下属" @click="operate(props.outlineTree.children, chapterIdx, 2)">+</button>
          <button type="button" class="outline-editor__action" title="删除" @click="operate(props.outlineTree.children, chapterIdx, 3)">x</button>
        </div>
      </div>
      <div v-for="(page, pageIdx) in chapter.children" :key="chapterIdx + '-' + pageIdx">
        <div class="outline-editor__row outline-editor__page">
          <span class="outline-editor__index">{{ chapterIdx + 1 }}.{{ pageIdx + 1 }}</span>
          <div class="outline-editor__input">
            <input v-model="page.name" @change="update" />
          </div>
          <div class="outline-editor__actions">
            <button type="button" class="outline-editor__action" title="在上方新增同级" @click="operate(chapter.children, pageIdx, 1)">=+</button>
            <button type="button" class="outline-editor__action" title="在下方新增下属" @click="operate(chapter.children, pageIdx, 2)">+</button>
            <button type="button" class="outline-editor__action" title="删除" @click="operate(chapter.children, pageIdx, 3)">x</button>
          </div>
        </div>
        <div v-for="(title, titleIdx) in page.children" :key="chapterIdx + '-' + pageIdx + '-' + titleIdx">
          <div class="outline-editor__row outline-editor__title">
            <span class="outline-editor__index">{{ chapterIdx + 1 }}.{{ pageIdx + 1 }}.{{ titleIdx + 1 }}</span>
            <div class="outline-editor__input">
              <input v-model="title.name" @change="update" />
            </div>
            <div class="outline-editor__actions">
              <button type="button" class="outline-editor__action" title="在上方新增同级" @click="operate(page.children, titleIdx, 1)">=+</button>
              <button type="button" class="outline-editor__action" title="删除" @click="operate(page.children, titleIdx, 3)">x</button>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, defineEmits } from 'vue'

const props = defineProps<{
  outlineTree: any
}>()

const version = ref(0)
const $emit = defineEmits(['update'])

function appendMd(children: any) {
  let str = ''
  for (let i = 0; i < children.length; i++) {
    let level = children[i].level
    if (level == 0) {
      str += '- '
    } else {
      for (let j = 0; j < level; j++) {
        str += '#'
      }
      str += ' '
    }
    str += children[i].name + '\n'
    if (children[i].children) {
      str += appendMd(children[i].children)
    }
  }
  return str
}

function operate(children: any, idx: number, type: number) {
  let current = children[idx]
  if (type == 1) {
    // 在上方新增同级
    children.splice(idx, 0, {
      level: current.level,
      name: '请输入文字',
      children: []
    })
  } else if (type == 2) {
    // 在下方新增下属
    current.children.splice(0, 0, {
      level: current.level + 1,
      name: '请输入文字',
      children: []
    })
  } else if (type == 3) {
    // 删除
    children.splice(idx, 1)
  }
  update()
}

function update() {
  let outlineMd = ''
  outlineMd += '# ' + props.outlineTree.name + '\n'
  outlineMd += appendMd(props.outlineTree.children)
  version.value++
  $emit('update', outlineMd)
}
</script>

<style scoped>
.outline-editor {
  display: grid;
  gap: 8px;
  padding: 6px 0;
}

.outline-editor__subject {
  text-align: center;
  margin-bottom: 10px;
  color: var(--ink-900);
  font-weight: 700;
  font-size: 20px;
}

.outline-editor__row {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 6px 8px;
  border-radius: 10px;
  transition: background 0.2s var(--ease);
}

.outline-editor__row:hover {
  background: var(--surface-2);
}

.outline-editor__badge {
  padding: 4px 10px;
  border-radius: var(--radius-pill);
  font-size: 12px;
  font-weight: 600;
  color: var(--accent-600);
  background: var(--accent-100);
}

.outline-editor__index {
  width: 56px;
  text-align: right;
  font-weight: 600;
  font-size: 12px;
  color: var(--ink-400);
}

.outline-editor__input {
  flex: 1;
}

.outline-editor__input input {
  width: 100%;
  border: 1px solid transparent;
  border-radius: 8px;
  padding: 6px 8px;
  background: transparent;
  color: var(--ink-900);
}

.outline-editor__input input:focus {
  outline: none;
  border-color: var(--accent-200);
  background: #fff;
  box-shadow: var(--ring);
}

.outline-editor__actions {
  display: flex;
  gap: 6px;
  opacity: 0;
  pointer-events: none;
  transition: opacity 0.2s var(--ease);
}

.outline-editor__row:hover .outline-editor__actions {
  opacity: 1;
  pointer-events: auto;
}

.outline-editor__action {
  border: 1px solid var(--line);
  background: #fff;
  border-radius: 8px;
  padding: 4px 6px;
  font-size: 12px;
  cursor: pointer;
  color: var(--ink-500);
  transition: all 0.2s var(--ease);
}

.outline-editor__action:hover {
  border-color: var(--accent-400);
  color: var(--accent-600);
}

.outline-editor__page {
  padding-left: 10px;
}

.outline-editor__title {
  padding-left: 22px;
}

@media (max-width: 900px) {
  .outline-editor__actions {
    opacity: 1;
    pointer-events: auto;
  }
}
</style>
