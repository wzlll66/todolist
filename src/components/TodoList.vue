<script setup>
import { ref, computed, watch, onMounted, onUnmounted, nextTick } from 'vue'

// ── 数据 ──────────────────────────────────────────
const STORAGE_KEY = 'todolist-data'

const todos = ref([])
const newTodo = ref('')
const newCategory = ref('')
const newDueDate = ref('')
const newPriority = ref('medium')
const searchQuery = ref('')
const filter = ref('all')
const categoryFilter = ref('all')
const editingId = ref(null)
const editText = ref('')
const darkMode = ref(false)
const draggedIndex = ref(null)
const dragOverIndex = ref(null)
const showCompleted = ref(true)

// 预设分类
const categories = ['工作', '个人', '学习', '健康', '购物']

// ── 初始化 ──────────────────────────────────────
onMounted(() => {
  const saved = localStorage.getItem(STORAGE_KEY)
  if (saved) todos.value = JSON.parse(saved)
  darkMode.value = localStorage.getItem('todolist-dark') === 'true'
  applyTheme()
})

// ── 持久化 ──────────────────────────────────────
watch(todos, (val) => localStorage.setItem(STORAGE_KEY, JSON.stringify(val)), { deep: true })
watch(darkMode, (val) => {
  localStorage.setItem('todolist-dark', val)
  applyTheme()
})

function applyTheme() {
  document.body.style.background = darkMode.value ? '#0f0f1a' : '#f0f2f8'
  document.body.style.color = darkMode.value ? '#e2e8f0' : '#1e293b'
}

function toggleDark() {
  darkMode.value = !darkMode.value
}

// ── 计算属性 ────────────────────────────────────
const filteredTodos = computed(() => {
  let list = todos.value

  if (searchQuery.value) {
    const q = searchQuery.value.toLowerCase()
    list = list.filter(t => t.text.toLowerCase().includes(q))
  }

  if (filter.value === 'active') list = list.filter(t => !t.done)
  else if (filter.value === 'done') list = list.filter(t => t.done)
  else if (filter.value === 'today') {
    const today = new Date().toISOString().slice(0, 10)
    list = list.filter(t => t.dueDate === today)
  }
  else if (filter.value === 'overdue') {
    const today = new Date().toISOString().slice(0, 10)
    list = list.filter(t => t.dueDate && t.dueDate < today && !t.done)
  }

  if (categoryFilter.value !== 'all') {
    list = list.filter(t => t.category === categoryFilter.value)
  }

  // 排序: 未完成在前，按优先级排
  return [...list].sort((a, b) => {
    if (a.done !== b.done) return a.done ? 1 : -1
    const pri = { high: 0, medium: 1, low: 2 }
    return pri[a.priority] - pri[b.priority]
  })
})

const allCategories = computed(() => {
  const set = new Set(todos.value.map(t => t.category).filter(Boolean))
  return [...set]
})

const stats = computed(() => {
  const total = todos.value.length
  const done = todos.value.filter(t => t.done).length
  const active = total - done
  const today = new Date().toISOString().slice(0, 10)
  const todayTasks = todos.value.filter(t => t.dueDate === today && !t.done).length
  const overdue = todos.value.filter(t => t.dueDate && t.dueDate < today && !t.done).length
  return { total, done, active, todayTasks, overdue, percent: total ? Math.round((done / total) * 100) : 0 }
})

// ── 方法 ─────────────────────────────────────────
function addTodo() {
  const text = newTodo.value.trim()
  if (!text) return
  todos.value.push({
    id: Date.now(),
    text,
    done: false,
    priority: newPriority.value,
    category: newCategory.value || '',
    dueDate: newDueDate.value,
    createdAt: new Date().toISOString(),
  })
  newTodo.value = ''
  newDueDate.value = ''
  newPriority.value = 'medium'
  newCategory.value = ''
}

function removeTodo(id) {
  todos.value = todos.value.filter(t => t.id !== id)
  if (editingId.value === id) editingId.value = null
}

function toggleTodo(id) {
  const todo = todos.value.find(t => t.id === id)
  if (todo) todo.done = !todo.done
}

function startEdit(todo) {
  editingId.value = todo.id
  editText.value = todo.text
}

function finishEdit(id) {
  const todo = todos.value.find(t => t.id === id)
  if (todo) {
    const text = editText.value.trim()
    text ? todo.text = text : removeTodo(id)
  }
  editingId.value = null
}

function cancelEdit() {
  editingId.value = null
}

function clearDone() {
  todos.value = todos.value.filter(t => !t.done)
}

function clearAll() {
  if (confirm('确定要删除所有任务吗？此操作不可恢复。')) {
    todos.value = []
  }
}

// ── 拖拽排序 ─────────────────────────────────────
function onDragStart(index) {
  draggedIndex.value = index
}

function onDragOver(e, index) {
  e.preventDefault()
  dragOverIndex.value = index
}

function onDragEnd() {
  if (draggedIndex.value !== null && dragOverIndex.value !== null) {
    const from = filteredTodos.value[draggedIndex.value]
    const to = filteredTodos.value[dragOverIndex.value]
    const fromIdx = todos.value.indexOf(from)
    const toIdx = todos.value.indexOf(to)
    const item = todos.value.splice(fromIdx, 1)[0]
    todos.value.splice(toIdx, 0, item)
  }
  draggedIndex.value = null
  dragOverIndex.value = null
}

// ── 键盘快捷键 ───────────────────────────────
function onKeydown(e) {
  if (e.ctrlKey && e.key === 'k') {
    e.preventDefault()
    document.querySelector('.new-input')?.focus()
  }
}

onMounted(() => window.addEventListener('keydown', onKeydown))
onUnmounted(() => window.removeEventListener('keydown', onKeydown))

// ── 时间格式化 ─────────────────────────────────
function formatDate(d) {
  if (!d) return ''
  const today = new Date().toISOString().slice(0, 10)
  const tomorrow = new Date(Date.now() + 86400000).toISOString().slice(0, 10)
  if (d === today) return '今天'
  if (d === tomorrow) return '明天'
  return d
}

function isOverdue(d) {
  return d && d < new Date().toISOString().slice(0, 10)
}

// ── 优先级颜色 ─────────────────────────────────
const priorityConfig = {
  high: { label: '高', color: '#ef4444', bg: 'rgba(239,68,68,0.12)' },
  medium: { label: '中', color: '#f59e0b', bg: 'rgba(245,158,11,0.12)' },
  low: { label: '低', color: '#22c55e', bg: 'rgba(34,197,94,0.12)' },
}
</script>

<template>
  <div :class="['app-shell', { dark: darkMode }]">
    <!-- 背景装饰 -->
    <div class="bg-blob bg-blob-1"></div>
    <div class="bg-blob bg-blob-2"></div>
    <div class="bg-blob bg-blob-3"></div>

    <!-- 主容器 -->
    <div class="main-card">
      <!-- 头部 -->
      <header class="header">
        <div class="header-left">
          <div class="logo-icon">
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <path d="M9 11l3 3L22 4"/>
              <path d="M21 12v7a2 2 0 01-2 2H5a2 2 0 01-2-2V5a2 2 0 012-2h11"/>
            </svg>
          </div>
          <div>
            <h1 class="app-title">TodoList</h1>
            <p class="app-subtitle">{{ stats.active }} 项待办</p>
          </div>
        </div>
        <div class="header-actions">
          <button class="icon-btn" @click="toggleDark" :title="darkMode ? '浅色模式' : '深色模式'">
            <!-- 月亮 -->
            <svg v-if="!darkMode" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" width="20" height="20">
              <path d="M21 12.79A9 9 0 1111.21 3 7 7 0 0021 12.79z"/>
            </svg>
            <!-- 太阳 -->
            <svg v-else viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" width="20" height="20">
              <circle cx="12" cy="12" r="5"/><path d="M12 1v2M12 21v2M4.22 4.22l1.42 1.42M18.36 18.36l1.42 1.42M1 12h2M21 12h2M4.22 19.78l1.42-1.42M18.36 5.64l1.42-1.42"/>
            </svg>
          </button>
        </div>
      </header>

      <!-- 进度条 -->
      <div class="progress-section" v-if="stats.total > 0">
        <div class="progress-bar">
          <div class="progress-fill" :style="{ width: stats.percent + '%' }"></div>
        </div>
        <div class="progress-stats">
          <span>{{ stats.percent }}% 已完成</span>
          <div class="stat-tags">
            <span v-if="stats.todayTasks" class="stat-tag today">📅 {{ stats.todayTasks }}</span>
            <span v-if="stats.overdue" class="stat-tag overdue">⚠ {{ stats.overdue }}</span>
          </div>
        </div>
      </div>

      <!-- 输入区 -->
      <form class="input-section" @submit.prevent="addTodo">
        <div class="input-main-row">
          <input
            v-model="newTodo"
            class="new-input"
            placeholder="添加新任务... (Ctrl+K 聚焦)"
            autofocus
          />
          <button class="btn-add" type="submit">
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" width="20" height="20">
              <line x1="12" y1="5" x2="12" y2="19"/><line x1="5" y1="12" x2="19" y2="12"/>
            </svg>
          </button>
        </div>
        <div class="input-meta-row">
          <select v-model="newPriority" class="meta-select">
            <option value="high">高优先级</option>
            <option value="medium">中优先级</option>
            <option value="low">低优先级</option>
          </select>
          <div class="date-wrap">
            <input type="date" v-model="newDueDate" class="meta-date" />
            <span class="date-placeholder" :class="{ hide: newDueDate }">截止日期</span>
          </div>
          <select v-model="newCategory" class="meta-select meta-select-full">
            <option value="">选择分类</option>
            <option v-for="c in categories" :key="c" :value="c">{{ c }}</option>
          </select>
        </div>
      </form>

      <!-- 工具栏 -->
      <div class="toolbar">
        <div class="toolbar-left">
          <div class="search-box">
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" width="16" height="16">
              <circle cx="11" cy="11" r="8"/><line x1="21" y1="21" x2="16.65" y2="16.65"/>
            </svg>
            <input v-model="searchQuery" placeholder="搜索..." class="search-input" />
          </div>
        </div>
        <div class="toolbar-right">
          <button v-if="todos.some(t => t.done)" class="tool-btn danger" @click="clearDone">
            清除已完成
          </button>
          <button v-if="todos.length" class="tool-btn danger" @click="clearAll">
            清空全部
          </button>
        </div>
      </div>

      <!-- 筛选 tabs -->
      <div class="filter-tabs">
        <button
          v-for="f in ['all', 'active', 'done', 'today', 'overdue']"
          :key="f"
          :class="['filter-tab', { active: filter === f }]"
          @click="filter = f"
        >
          {{ { all: '全部', active: '未完成', done: '已完成', today: '今天', overdue: '已逾期' }[f] }}
        </button>
      </div>

      <!-- 分类筛选 -->
      <div class="category-chips" v-if="allCategories.length">
        <button
          :class="['chip', { active: categoryFilter === 'all' }]"
          @click="categoryFilter = 'all'"
        >全部</button>
        <button
          v-for="c in allCategories"
          :key="c"
          :class="['chip', { active: categoryFilter === c }]"
          @click="categoryFilter = c"
        >{{ c }}</button>
      </div>

      <!-- 任务列表 -->
      <div class="list-area">
        <TransitionGroup name="todo" tag="ul" class="todo-list">
          <li
            v-for="(todo, index) in filteredTodos"
            :key="todo.id"
            :class="['todo-item', {
              done: todo.done,
              editing: editingId === todo.id,
              dragging: draggedIndex === index,
              dragOver: dragOverIndex === index,
              overdue: !todo.done && isOverdue(todo.dueDate),
            }]"
            draggable="true"
            @dragstart="onDragStart(index)"
            @dragover="onDragOver($event, index)"
            @dragend="onDragEnd"
          >
            <!-- 拖拽手柄 -->
            <span class="drag-handle">
              <svg viewBox="0 0 24 24" fill="currentColor" width="16" height="16">
                <circle cx="9" cy="5" r="1.5"/><circle cx="15" cy="5" r="1.5"/>
                <circle cx="9" cy="12" r="1.5"/><circle cx="15" cy="12" r="1.5"/>
                <circle cx="9" cy="19" r="1.5"/><circle cx="15" cy="19" r="1.5"/>
              </svg>
            </span>

            <!-- 复选框 -->
            <button
              :class="['checkbox-custom', { checked: todo.done }]"
              @click="toggleTodo(todo.id)"
            >
              <svg v-if="todo.done" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="3" width="14" height="14">
                <polyline points="20 6 9 17 4 12"/>
              </svg>
            </button>

            <!-- 内容区 -->
            <div class="todo-content" @dblclick="startEdit(todo)" v-if="editingId !== todo.id">
              <span class="todo-text">{{ todo.text }}</span>
              <span class="todo-meta">
                <span v-if="todo.category" class="meta-badge category">{{ todo.category }}</span>
                <span
                  v-if="todo.dueDate"
                  :class="['meta-badge', isOverdue(todo.dueDate) && !todo.done ? 'overdue' : 'duedate']"
                >
                  {{ isOverdue(todo.dueDate) && !todo.done ? '⏰' : '📅' }} {{ formatDate(todo.dueDate) }}
                </span>
                <span
                  class="meta-badge priority"
                  :style="{ color: priorityConfig[todo.priority].color, background: priorityConfig[todo.priority].bg }"
                >
                  {{ priorityConfig[todo.priority].label }}
                </span>
              </span>
            </div>

            <!-- 编辑态 -->
            <input
              v-else
              v-model="editText"
              class="edit-input-inline"
              @blur="finishEdit(todo.id)"
              @keyup.enter="finishEdit(todo.id)"
              @keyup.escape="cancelEdit"
              ref="editInputRef"
            />

            <!-- 删除 -->
            <button class="btn-delete" @click="removeTodo(todo.id)">
              <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" width="16" height="16">
                <line x1="18" y1="6" x2="6" y2="18"/><line x1="6" y1="6" x2="18" y2="18"/>
              </svg>
            </button>
          </li>
        </TransitionGroup>

        <!-- 空状态 -->
        <div class="empty-state" v-if="!filteredTodos.length">
          <div class="empty-icon">
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1" width="64" height="64">
              <path d="M9 11l3 3L22 4"/>
              <path d="M21 12v7a2 2 0 01-2 2H5a2 2 0 01-2-2V5a2 2 0 012-2h11"/>
            </svg>
          </div>
          <p class="empty-title">{{ searchQuery ? '没有匹配的任务' : '还没有任务' }}</p>
          <p class="empty-desc">{{ searchQuery ? '换个关键词试试' : '在上方输入框添加你的第一个任务吧' }}</p>
        </div>
      </div>

      <!-- 底部 -->
      <footer class="footer-bar">
        <span>{{ stats.done }} / {{ stats.total }} 已完成</span>
        <span class="footer-hint">双击编辑 · 拖拽排序 · Ctrl+K 聚焦</span>
      </footer>
    </div>
  </div>
</template>

<style scoped>
/* ── 全局变量 ─────────────────────────────────── */
.app-shell {
  --bg-primary: #ffffff;
  --bg-secondary: #f8fafc;
  --bg-tertiary: #f1f5f9;
  --text-primary: #1e293b;
  --text-secondary: #64748b;
  --text-tertiary: #94a3b8;
  --border: #e2e8f0;
  --border-light: #f1f5f9;
  --accent: #6366f1;
  --accent-light: #eef2ff;
  --accent-hover: #4f46e5;
  --danger: #ef4444;
  --danger-light: #fef2f2;
  --shadow-sm: 0 1px 2px rgba(0,0,0,0.05);
  --shadow-md: 0 4px 6px -1px rgba(0,0,0,0.07), 0 2px 4px -2px rgba(0,0,0,0.05);
  --shadow-lg: 0 10px 25px -5px rgba(0,0,0,0.08), 0 4px 10px -6px rgba(0,0,0,0.04);
  --radius-sm: 8px;
  --radius-md: 12px;
  --radius-lg: 20px;
  --radius-xl: 24px;
}

.app-shell.dark {
  --bg-primary: #1e1e2e;
  --bg-secondary: #252540;
  --bg-tertiary: #2a2a45;
  --text-primary: #e2e8f0;
  --text-secondary: #94a3b8;
  --text-tertiary: #64748b;
  --border: #323255;
  --border-light: #2a2a48;
  --accent: #818cf8;
  --accent-light: #1e1b4b;
  --accent-hover: #6366f1;
  --danger: #f87171;
  --danger-light: #451a1a;
  --shadow-sm: 0 1px 2px rgba(0,0,0,0.2);
  --shadow-md: 0 4px 6px -1px rgba(0,0,0,0.3);
  --shadow-lg: 0 10px 25px -5px rgba(0,0,0,0.4);
}

/* ── 背景装饰 ─────────────────────────────────── */
.bg-blob {
  position: fixed;
  border-radius: 50%;
  filter: blur(80px);
  opacity: 0.15;
  pointer-events: none;
  z-index: 0;
  transition: opacity 0.5s;
}
.bg-blob-1 { width: 400px; height: 400px; background: #818cf8; top: -100px; right: -100px; }
.bg-blob-2 { width: 300px; height: 300px; background: #c084fc; bottom: -50px; left: -80px; }
.bg-blob-3 { width: 250px; height: 250px; background: #34d399; top: 40%; right: -60px; }
.dark .bg-blob { opacity: 0.08; }

/* ── 主卡片 ───────────────────────────────────── */
.app-shell {
  display: flex;
  justify-content: center;
  padding: 30px 16px 60px;
  min-height: 100vh;
  position: relative;
  z-index: 1;
}

.main-card {
  width: 100%;
  max-width: 620px;
  background: var(--bg-primary);
  border-radius: var(--radius-xl);
  box-shadow: var(--shadow-lg);
  padding: 28px 32px 20px;
  border: 1px solid var(--border);
  transition: background 0.3s, border-color 0.3s, box-shadow 0.3s;
  position: relative;
  z-index: 1;
}

/* ── 头部 ─────────────────────────────────────── */
.header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 20px;
}
.header-left {
  display: flex;
  align-items: center;
  gap: 14px;
}
.logo-icon {
  width: 44px;
  height: 44px;
  background: var(--accent);
  color: #fff;
  border-radius: var(--radius-md);
  display: flex;
  align-items: center;
  justify-content: center;
}
.logo-icon svg { width: 24px; height: 24px; }
.app-title {
  font-size: 22px;
  font-weight: 700;
  color: var(--text-primary);
  letter-spacing: -0.5px;
}
.app-subtitle {
  font-size: 13px;
  color: var(--text-secondary);
  margin-top: 1px;
}
.icon-btn {
  width: 40px; height: 40px;
  border: 1px solid var(--border);
  border-radius: var(--radius-sm);
  background: var(--bg-secondary);
  color: var(--text-secondary);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.2s;
}
.icon-btn:hover { background: var(--bg-tertiary); color: var(--text-primary); }

/* ── 进度条 ───────────────────────────────────── */
.progress-section { margin-bottom: 20px; }
.progress-bar {
  height: 6px;
  background: var(--bg-tertiary);
  border-radius: 3px;
  overflow: hidden;
}
.progress-fill {
  height: 100%;
  background: linear-gradient(90deg, #6366f1, #8b5cf6);
  border-radius: 3px;
  transition: width 0.5s ease;
}
.progress-stats {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: 6px;
  font-size: 12px;
  color: var(--text-tertiary);
}
.stat-tags { display: flex; gap: 8px; }
.stat-tag {
  font-size: 11px;
  padding: 2px 8px;
  border-radius: 10px;
  font-weight: 500;
}
.stat-tag.today { background: #dbeafe; color: #2563eb; }
.dark .stat-tag.today { background: #1e3a5f; color: #60a5fa; }
.stat-tag.overdue { background: #fee2e2; color: #dc2626; }
.dark .stat-tag.overdue { background: #451a1a; color: #f87171; }

/* ── 输入区 ───────────────────────────────────── */
.input-section {
  display: flex;
  flex-direction: column;
  gap: 10px;
  margin-bottom: 16px;
}
.input-main-row {
  display: flex;
  gap: 8px;
}
.new-input {
  flex: 1;
  padding: 12px 16px;
  border: 2px solid var(--border);
  border-radius: var(--radius-md);
  font-size: 15px;
  background: var(--bg-secondary);
  color: var(--text-primary);
  outline: none;
  transition: all 0.2s;
}
.new-input:focus { border-color: var(--accent); box-shadow: 0 0 0 3px var(--accent-light); }
.new-input::placeholder { color: var(--text-tertiary); }
.btn-add {
  width: 48px;
  background: var(--accent);
  color: #fff;
  border: none;
  border-radius: var(--radius-md);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.2s;
  flex-shrink: 0;
}
.btn-add:hover { background: var(--accent-hover); transform: scale(1.03); }
.btn-add:active { transform: scale(0.97); }

.input-meta-row {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 8px;
}
.meta-select, .meta-date {
  padding: 8px 12px;
  border: 1px solid var(--border);
  border-radius: var(--radius-sm);
  font-size: 13px;
  background: var(--bg-secondary);
  color: var(--text-primary);
  outline: none;
  cursor: pointer;
  transition: border-color 0.2s;
  width: 100%;
}
.meta-select:focus, .meta-date:focus { border-color: var(--accent); }
.meta-select-full { grid-column: 1 / -1; }

/* 日期输入包装 */
.date-wrap { position: relative; }
.date-placeholder {
  position: absolute;
  left: 12px;
  top: 50%;
  transform: translateY(-50%);
  font-size: 13px;
  color: var(--text-tertiary);
  pointer-events: none;
  white-space: nowrap;
}
.date-placeholder.hide { display: none; }
.meta-date {
  color-scheme: var(--bg-primary);
}
.meta-date::-webkit-calendar-picker-indicator { cursor: pointer; }

/* ── 工具栏 ───────────────────────────────────── */
.toolbar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 12px;
  margin-bottom: 12px;
  flex-wrap: wrap;
}
.search-box {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 7px 14px;
  border: 1px solid var(--border);
  border-radius: var(--radius-sm);
  background: var(--bg-secondary);
  flex: 1;
  max-width: 220px;
  transition: border-color 0.2s;
}
.search-box:focus-within { border-color: var(--accent); }
.search-box svg { color: var(--text-tertiary); flex-shrink: 0; }
.search-input {
  border: none;
  background: none;
  outline: none;
  font-size: 13px;
  color: var(--text-primary);
  width: 100%;
}
.search-input::placeholder { color: var(--text-tertiary); }
.tool-btn {
  padding: 6px 14px;
  border: 1px solid var(--border);
  border-radius: var(--radius-sm);
  font-size: 12px;
  background: var(--bg-secondary);
  color: var(--text-secondary);
  cursor: pointer;
  transition: all 0.2s;
  white-space: nowrap;
}
.tool-btn:hover { background: var(--bg-tertiary); }
.tool-btn.danger:hover { background: var(--danger-light); color: var(--danger); border-color: var(--danger); }

/* ── 筛选 Tabs ─────────────────────────────── */
.filter-tabs {
  display: flex;
  gap: 4px;
  margin-bottom: 12px;
  background: var(--bg-tertiary);
  border-radius: var(--radius-sm);
  padding: 4px;
}
.filter-tab {
  flex: 1;
  padding: 7px 0;
  border: none;
  border-radius: 6px;
  background: transparent;
  color: var(--text-secondary);
  font-size: 13px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s;
}
.filter-tab:hover { color: var(--text-primary); }
.filter-tab.active {
  background: var(--bg-primary);
  color: var(--accent);
  box-shadow: var(--shadow-sm);
}

/* ── 分类 Chips ─────────────────────────────── */
.category-chips {
  display: flex;
  gap: 6px;
  flex-wrap: wrap;
  margin-bottom: 14px;
}
.chip {
  padding: 4px 14px;
  border: 1px solid var(--border);
  border-radius: 20px;
  background: var(--bg-secondary);
  color: var(--text-secondary);
  font-size: 12px;
  cursor: pointer;
  transition: all 0.2s;
}
.chip:hover { border-color: var(--accent); color: var(--accent); }
.chip.active { background: var(--accent); color: #fff; border-color: var(--accent); }

/* ── 任务列表 ─────────────────────────────────── */
.list-area { min-height: 200px; }
.todo-list { list-style: none; }

.todo-item {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 12px 10px;
  border-radius: var(--radius-sm);
  border: 1px solid transparent;
  transition: all 0.2s;
  cursor: default;
}
.todo-item:hover {
  background: var(--bg-secondary);
  border-color: var(--border);
}
.todo-item.done { opacity: 0.55; }
.todo-item.dragging { opacity: 0.4; }
.todo-item.dragOver { border-color: var(--accent); background: var(--accent-light); }
.todo-item.overdue { border-left: 3px solid var(--danger); }

/* 拖拽手柄 */
.drag-handle {
  cursor: grab;
  color: var(--text-tertiary);
  display: flex;
  align-items: center;
  opacity: 0;
  transition: opacity 0.2s;
  flex-shrink: 0;
}
.todo-item:hover .drag-handle { opacity: 1; }

/* 自定义复选框 */
.checkbox-custom {
  width: 22px; height: 22px;
  border: 2px solid var(--border);
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  flex-shrink: 0;
  transition: all 0.25s;
  background: var(--bg-primary);
}
.checkbox-custom:hover { border-color: var(--accent); }
.checkbox-custom.checked {
  background: var(--accent);
  border-color: var(--accent);
  color: #fff;
}

/* 内容 */
.todo-content {
  flex: 1;
  min-width: 0;
  display: flex;
  flex-direction: column;
  gap: 3px;
}
.todo-text {
  font-size: 14px;
  color: var(--text-primary);
  word-break: break-word;
}
.done .todo-text { text-decoration: line-through; }

.todo-meta {
  display: flex;
  gap: 6px;
  flex-wrap: wrap;
}
.meta-badge {
  font-size: 11px;
  padding: 1px 8px;
  border-radius: 10px;
  font-weight: 500;
}
.meta-badge.category { background: var(--accent-light); color: var(--accent); }
.meta-badge.duedate { background: var(--bg-tertiary); color: var(--text-secondary); }
.meta-badge.overdue { background: var(--danger-light); color: var(--danger); }
.meta-badge.priority { font-size: 10px; }

/* 编辑输入 */
.edit-input-inline {
  flex: 1;
  padding: 6px 10px;
  border: 2px solid var(--accent);
  border-radius: 6px;
  font-size: 14px;
  background: var(--bg-primary);
  color: var(--text-primary);
  outline: none;
}

/* 删除按钮 */
.btn-delete {
  background: none;
  border: none;
  color: var(--text-tertiary);
  cursor: pointer;
  padding: 4px;
  border-radius: 6px;
  display: flex;
  align-items: center;
  opacity: 0;
  transition: all 0.2s;
  flex-shrink: 0;
}
.todo-item:hover .btn-delete { opacity: 1; }
.btn-delete:hover { background: var(--danger-light); color: var(--danger); }

/* ── 空状态 ───────────────────────────────────── */
.empty-state {
  text-align: center;
  padding: 48px 20px;
}
.empty-icon { color: var(--text-tertiary); opacity: 0.4; margin-bottom: 16px; }
.empty-title { font-size: 16px; font-weight: 600; color: var(--text-secondary); margin-bottom: 6px; }
.empty-desc { font-size: 13px; color: var(--text-tertiary); }

/* ── 底部 ─────────────────────────────────────── */
.footer-bar {
  display: flex;
  justify-content: space-between;
  margin-top: 16px;
  padding-top: 14px;
  border-top: 1px solid var(--border-light);
  font-size: 12px;
  color: var(--text-tertiary);
}
.footer-hint { font-size: 11px; }

/* ── 过渡动画 ─────────────────────────────────── */
.todo-enter-active { transition: all 0.4s ease-out; }
.todo-leave-active { transition: all 0.3s ease-in; }
.todo-enter-from { opacity: 0; transform: translateY(-12px) scale(0.95); }
.todo-leave-to { opacity: 0; transform: translateX(30px); }
.todo-move { transition: transform 0.3s ease; }

/* ── 移动端适配 ───────────────────────────────── */
@media (max-width: 640px) {
  .bg-blob { display: none; }

  .app-shell {
    padding: 0;
  }
  .main-card {
    max-width: 100%;
    border-radius: 0;
    padding: 20px 16px 24px;
    border: none;
    box-shadow: none;
    min-height: 100vh;
  }

  .header { margin-bottom: 16px; }
  .logo-icon { width: 36px; height: 36px; }
  .logo-icon svg { width: 20px; height: 20px; }
  .app-title { font-size: 19px; }
  .app-subtitle { font-size: 12px; }

  .new-input { padding: 12px 14px; font-size: 14px; }
  .btn-add { width: 46px; }
  .meta-select, .meta-date { padding: 8px 8px; font-size: 12px; }
  .date-placeholder { font-size: 12px; left: 8px; }

  .toolbar {
    flex-direction: column;
    align-items: stretch;
  }
  .toolbar-left { width: 100%; }
  .search-box { max-width: 100%; }
  .toolbar-right { display: flex; gap: 8px; }

  .filter-tabs {
    overflow-x: auto;
    -webkit-overflow-scrolling: touch;
    scrollbar-width: none;
  }
  .filter-tabs::-webkit-scrollbar { display: none; }
  .filter-tab {
    flex-shrink: 0;
    padding: 7px 12px;
    white-space: nowrap;
  }

  .todo-item {
    padding: 14px 8px;
    gap: 8px;
  }
  .drag-handle {
    opacity: 0.35;
    width: 24px;
  }
  .btn-delete {
    opacity: 0.6;
    width: 36px;
    height: 36px;
    justify-content: center;
  }
  .checkbox-custom {
    width: 26px;
    height: 26px;
    min-width: 26px;
  }
  .todo-text { font-size: 14px; }
  .todo-meta { gap: 4px; }
  .meta-badge { font-size: 10px; padding: 1px 7px; }

  .empty-state { padding: 60px 16px; }

  .footer-bar {
    flex-direction: column;
    align-items: center;
    gap: 4px;
    text-align: center;
  }
  .footer-hint { display: none; }
}

@media (max-width: 380px) {
  .input-meta-row { grid-template-columns: 1fr; }
  .meta-select-full { grid-column: 1; }
  .filter-tab { font-size: 11px; padding: 7px 8px; }
  .app-title { font-size: 17px; }
}
</style>
