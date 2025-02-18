<template>
  <div class="todo-container">
    <div class="todo-header">
      <h1>待办事项</h1>

      <form @submit.prevent="addTodo" class="todo-form">
        <input
            v-model="newTodo"
            type="text"
            placeholder="输入新的待办事项"
        />
        <button type="submit">
          <plus-circle-icon class="icon" />
          添加
        </button>
      </form>

      <div class="tabs-and-sort">
        <div class="tabs">
          <button
              @click="activeTab = 'todo'"
              :class="{ active: activeTab === 'todo' }"
          >
            待办事项
          </button>
          <button
              @click="activeTab = 'completed'"
              :class="{ active: activeTab === 'completed' }"
          >
            已完成
          </button>
        </div>
        <button @click="toggleSort" class="sort-btn">
          <arrow-up-icon v-if="sortOrder === 'asc'" class="icon" />
          <arrow-down-icon v-else-if="sortOrder === 'desc'" class="icon" />
          <arrow-down-up-icon v-else class="icon" />
        </button>
      </div>
    </div>

    <div class="todo-list-container">
      <draggable
          :list="displayedTodos"
          item-key="id"
          class="todo-list"
          @change="onDragChange"
          :animation="200"
      >
        <template #item="{ element: todo }">
          <div class="todo-item">
            <button
                @click="toggleTodo(todo)"
                :class="['status-btn', { completed: todo.completed }]"
            >
              <check-icon class="check-icon" />
            </button>

            <div class="todo-content">
              <input
                  v-if="editingId === todo.id"
                  v-model="editingText"
                  @blur="saveEdit(todo)"
                  @keyup.enter="saveEdit(todo)"
                  @keyup.esc="cancelEdit"
                  class="edit-input"
              />
              <div v-else class="todo-info">
                <span
                    :class="['todo-title', { completed: todo.completed }]"
                    @dblclick="startEdit(todo)"
                    @mouseenter="showPopup($event, todo)"
                    @mouseleave="hidePopup"
                >
                  {{ todo.title }}
                </span>
                <span class="timestamp">
                  {{ formatDate(todo.timestamp) }}
                </span>
              </div>
            </div>

            <div class="actions">
              <button
                  v-if="editingId !== todo.id"
                  @click="startEdit(todo)"
                  class="action-btn edit"
              >
                <edit-2-icon class="icon" />
              </button>
              <button
                  @click="deleteTodo(todo)"
                  class="action-btn delete"
              >
                <trash-2-icon class="icon" />
              </button>
            </div>
          </div>
        </template>
      </draggable>
    </div>
    <div v-if="popupVisible" class="popup" :style="popupStyle">
      {{ popupContent }}
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch, onMounted, nextTick } from 'vue'
import { Check as CheckIcon, Edit2 as Edit2Icon, Trash2 as Trash2Icon, PlusCircle as PlusCircleIcon, ArrowUp as ArrowUpIcon, ArrowDown as ArrowDownIcon, ArrowDownUp as ArrowDownUpIcon } from 'lucide-vue-next'
import draggable from 'vuedraggable'

const STORAGE_KEY = 'vue-todo-list'

const todos = ref([])
const newTodo = ref('')
const activeTab = ref('todo')
const editingId = ref(null)
const editingText = ref('')
const sortOrder = ref('default')

const loadTodos = () => {
  const storedTodos = localStorage.getItem(STORAGE_KEY)
  if (storedTodos) {
    todos.value = JSON.parse(storedTodos)
  }
}

const saveTodos = () => {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(todos.value))
}

watch(todos, saveTodos, { deep: true })

onMounted(loadTodos)

const filteredTodos = computed(() => {
  return todos.value.filter(todo =>
      activeTab.value === 'todo' ? !todo.completed : todo.completed
  )
})

const sortedTodos = computed(() => {
  let sorted = [...filteredTodos.value]
  if (sortOrder.value === 'asc') {
    sorted.sort((a, b) => new Date(a.timestamp) - new Date(b.timestamp))
  } else if (sortOrder.value === 'desc') {
    sorted.sort((a, b) => new Date(b.timestamp) - new Date(a.timestamp))
  }
  return sorted
})

const displayedTodos = computed(() => sortedTodos.value)

const toggleSort = () => {
  if (sortOrder.value === 'default') {
    sortOrder.value = 'asc'
  } else if (sortOrder.value === 'asc') {
    sortOrder.value = 'desc'
  } else {
    sortOrder.value = 'default'
  }
}

const formatDate = (timestamp) => {
  return new Date(timestamp).toLocaleString('zh-CN', {
    hour12: false,
    year: 'numeric',
    month: 'numeric',
    day: 'numeric',
    hour: '2-digit',
    minute: '2-digit'
  })
}

const addTodo = () => {
  if (!newTodo.value.trim()) return

  const newTodoItem = {
    id: Date.now(),
    title: newTodo.value,
    completed: false,
    timestamp: new Date().toISOString()
  }

  todos.value.unshift(newTodoItem)
  newTodo.value = ''
}

const toggleTodo = (todo) => {
  todo.completed = !todo.completed
}

const startEdit = async (todo) => {
  editingId.value = todo.id
  editingText.value = todo.title
  await nextTick()
  const editInput = document.querySelector('.edit-input')
  if (editInput) {
    editInput.focus()
  }
}

const saveEdit = (todo) => {
  if (editingText.value.trim()) {
    todo.title = editingText.value
  }
  editingId.value = null
}

const cancelEdit = () => {
  editingId.value = null
}

const deleteTodo = (todo) => {
  todos.value = todos.value.filter(t => t.id !== todo.id)
}

const popupVisible = ref(false)
const popupContent = ref('')
const popupStyle = ref({
  left: '0px',
  top: '0px'
})

const showPopup = (event, todo) => {
  const element = event.target
  const rect = element.getBoundingClientRect()

  if (element.scrollWidth > element.clientWidth) {
    popupContent.value = todo.title
    popupStyle.value = {
      left: `${rect.left}px`,
      top: `${rect.top - 5}px`
    }
    popupVisible.value = true
  }
}

const hidePopup = () => {
  popupVisible.value = false
}

const onDragChange = (event) => {
  if (event.added) {
    const { newIndex, element } = event.added
    todos.value.splice(newIndex, 0, element)
  } else if (event.removed) {
    const { oldIndex } = event.removed
    todos.value.splice(oldIndex, 1)
  } else if (event.moved) {
    const { oldIndex, newIndex } = event.moved
    const [movedItem] = todos.value.splice(oldIndex, 1)
    todos.value.splice(newIndex, 0, movedItem)
  }
  saveTodos()
}
</script>

<style scoped>
/* 样式保持不变 */
.todo-container {
  width: 400px;
  height: 500px;
  background: white;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  position: relative;
}

.todo-container::after {
  content: '';
  position: absolute;
  left: 0;
  right: 0;
  top: 0;
  bottom: 0;
  border-radius: 16px;
  pointer-events: none;
  box-shadow: 0 0 0 1px rgba(255, 255, 255, 0.1);
}

.todo-header {
  flex-shrink: 0;
  padding: 16px 16px 0;
}

h1 {
  font-size: 20px;
  color: #374151;
  text-align: center;
  margin-bottom: 16px;
  font-weight: 500;
}

.todo-form {
  display: flex;
  gap: 8px;
  margin-bottom: 16px;
  padding: 8px 0;
}

.todo-form input {
  flex: 1;
  padding: 12px 16px;
  border: 1px solid #e5e7eb;
  border-radius: 6px;
  font-size: 14px;
  transition: all 0.2s;
}

.todo-form input:focus {
  outline: none;
  border-color: #60a5fa;
  box-shadow: 0 0 0 3px rgba(96, 165, 250, 0.2);
}

.todo-form button {
  display: flex;
  align-items: center;
  gap: 4px;
  padding: 12px 16px;
  background: #0ea5e9;
  color: white;
  border: none;
  border-radius: 6px;
  font-size: 14px;
  cursor: pointer;
  transition: background 0.2s;
}

.todo-form button:hover {
  background: #0284c7;
}

.tabs-and-sort {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 16px;
  border-bottom: 1px solid #e5e7eb;
}

.tabs {
  display: flex;
  gap: 16px;
}

.tabs button {
  padding: 8px 4px;
  background: none;
  border: none;
  color: #6b7280;
  font-size: 14px;
  cursor: pointer;
  position: relative;
}

.tabs button.active {
  color: #0ea5e9;
}

.tabs button.active::after {
  content: '';
  position: absolute;
  bottom: -1px;
  left: 0;
  right: 0;
  height: 2px;
  background: #0ea5e9;
}

.sort-btn {
  background: none;
  border: none;
  cursor: pointer;
  color: #6b7280;
  padding: 4px;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: color 0.2s;
}

.sort-btn:hover {
  color: #0ea5e9;
}

.todo-list-container {
  flex: 1;
  overflow-y: auto;
  padding: 0 16px 16px;
}

.todo-list {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.todo-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px;
  background: #f9fafb;
  border-radius: 6px;
  transition: all 0.3s ease;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.05);
  cursor: pointer; /* 从 move 改为 pointer */
}

.todo-item:hover {
  transform: translateY(-3px);
  box-shadow: 0 3px 5px rgba(0, 0, 0, 0.1);
  background: #f3f4f6;
}

.status-btn {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  border: 2px solid #d1d5db;
  background-color: white !important;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  transition: all 0.3s ease;
  position: relative;
  padding: 0;
  margin: 0;
  appearance: none;
  -webkit-appearance: none;
}

.status-btn:hover {
  border-color: #0ea5e9;
}

.status-btn.completed {
  background-color: #22c55e !important;
  border-color: #22c55e;
}

.status-btn .check-icon {
  width: 14px;
  height: 14px;
  color: white;
  transform: scale(0);
  opacity: 0;
  transition: all 0.3s ease;
  pointer-events: none;
}

.status-btn.completed .check-icon {
  transform: scale(1);
  opacity: 1;
}

.todo-content {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  min-width: 0;
}

.todo-info {
  display: flex;
  flex-direction: column;
  gap: 4px;
  min-width: 0;
  flex: 1;
}

.todo-title {
  font-size: 14px;
  color: #374151;
  transition: color 0.3s ease, text-decoration 0.3s ease;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  max-width: 200px;
  position: relative;
}

.todo-title.completed {
  color: #9ca3af;
  text-decoration: line-through;
}

.edit-input {
  flex: 1;
  padding: 6px 8px;
  border: 1px solid #60a5fa;
  border-radius: 4px;
  font-size: 14px;
  background: white;
  min-width: 0;
}

.edit-input:focus {
  outline: none;
  box-shadow: 0 0 0 2px rgba(96, 165, 250, 0.2);
}

.timestamp {
  font-size: 12px;
  color: #9ca3af;
  white-space: nowrap;
}

.actions {
  display: flex;
  gap: 4px;
  opacity: 0;
  transition: opacity 0.2s;
}

.todo-item:hover .actions {
  opacity: 1;
}

.action-btn {
  padding: 4px;
  border: none;
  background: none;
  border-radius: 4px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: background 0.2s;
}

.action-btn.edit {
  color: #0ea5e9;
}

.action-btn.edit:hover {
  background: #e0f2fe;
}

.action-btn.delete {
  color: #ef4444;
}

.action-btn.delete:hover {
  background: #fee2e2;
}

.icon {
  width: 16px;
  height: 16px;
}

.todo-list-container::-webkit-scrollbar {
  width: 6px;
}

.todo-list-container::-webkit-scrollbar-track {
  background: transparent;
}

.todo-list-container::-webkit-scrollbar-thumb {
  background-color: #d1d5db;
  border-radius: 3px;
}

.todo-list-container::-webkit-scrollbar-thumb:hover {
  background-color: #9ca3af;
}

.popup {
  position: fixed;
  transform: translateY(-100%);
  background: white;
  padding: 8px 12px;
  border-radius: 6px;
  font-size: 14px;
  color: #374151;
  box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1), 0 2px 4px -1px rgba(0, 0, 0, 0.06);
  z-index: 1000;
  max-width: 300px;
  word-wrap: break-word;
  white-space: normal;
  animation: fadeIn 0.2s ease;
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(-90%);
  }
  to {
    opacity: 1;
    transform: translateY(-100%);
  }
}
</style>
