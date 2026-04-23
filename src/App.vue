<template>
  <div class="kanban-container">
    <header class="kanban-header">
      <h1>团队任务看板</h1>
      <el-button type="primary" @click="handleAddTask" :icon="Plus">新建任务</el-button>
    </header>

    <main class="kanban-content">
      <div
        v-for="column in columns"
        :key="column.id"
        class="kanban-column"
      >
        <div class="column-header">
          <span class="column-title">{{ column.label }}</span>
          <el-tag :type="column.tagType" size="small">
            {{ getTasksByStatus(column.id).length }}
          </el-tag>
        </div>

        <div
          class="column-tasks"
          :class="{ 'drag-over': dragStatus.column === column.id && dragStatus.active }"
          @dragover.prevent="onDragOver(column.id)"
          @dragleave="onDragLeave"
          @drop="onDrop(column.id)"
        >
          <div
            v-for="(task, index) in getTasksByStatus(column.id)"
            :key="task.id"
            class="task-card"
            draggable="true"
            :class="{ 'dragging': dragStatus.taskId === task.id }"
            @dragstart="onDragStart(task, column.id, index)"
            @dragend="onDragEnd"
          >
            <div class="task-header">
              <span class="task-priority" :class="`priority-${task.priority}`">
                {{ getPriorityLabel(task.priority) }}
              </span>
              <div class="task-actions">
                <el-button type="text" size="small" @click.stop="handleEditTask(task)" :icon="Edit" />
                <el-button type="text" size="small" class="danger" @click.stop="handleDeleteTask(task)" :icon="Delete" />
              </div>
            </div>

            <div class="task-title">{{ task.title }}</div>
            <div v-if="task.description" class="task-description">{{ task.description }}</div>

            <div class="task-footer">
              <el-tag v-if="task.dueDate" size="small" :type="getDueDateTagType(task.dueDate)">
                {{ formatDate(task.dueDate) }}
              </el-tag>
              <el-tag v-if="task.assignee" size="small" type="info">
                {{ task.assignee }}
              </el-tag>
            </div>
          </div>

          <div v-if="getTasksByStatus(column.id).length === 0" class="empty-hint">
            暂无任务，拖拽任务到此处
          </div>
        </div>
      </div>
    </main>

    <el-dialog
      v-model="dialogVisible"
      :title="isEdit ? '编辑任务' : '新建任务'"
      width="500px"
      :close-on-click-modal="false"
    >
      <el-form :model="formData" :rules="formRules" ref="formRef" label-width="80px">
        <el-form-item label="标题" prop="title">
          <el-input v-model="formData.title" placeholder="请输入任务标题" maxlength="50" show-word-limit />
        </el-form-item>

        <el-form-item label="状态" prop="status">
          <el-select v-model="formData.status" placeholder="请选择状态" style="width: 100%">
            <el-option
              v-for="column in columns"
              :key="column.id"
              :label="column.label"
              :value="column.id"
            />
          </el-select>
        </el-form-item>

        <el-form-item label="优先级" prop="priority">
          <el-select v-model="formData.priority" placeholder="请选择优先级" style="width: 100%">
            <el-option label="高" value="high" />
            <el-option label="中" value="medium" />
            <el-option label="低" value="low" />
          </el-select>
        </el-form-item>

        <el-form-item label="负责人" prop="assignee">
          <el-input v-model="formData.assignee" placeholder="请输入负责人姓名" maxlength="20" />
        </el-form-item>

        <el-form-item label="截止日期" prop="dueDate">
          <el-date-picker
            v-model="formData.dueDate"
            type="date"
            placeholder="请选择截止日期"
            style="width: 100%"
            value-format="YYYY-MM-DD"
          />
        </el-form-item>

        <el-form-item label="描述" prop="description">
          <el-input
            v-model="formData.description"
            type="textarea"
            :rows="3"
            placeholder="请输入任务描述（可选）"
            maxlength="200"
            show-word-limit
          />
        </el-form-item>
      </el-form>

      <template #footer>
        <el-button @click="dialogVisible = false">取消</el-button>
        <el-button type="primary" @click="handleSubmit" :loading="submitting">确定</el-button>
      </template>
    </el-dialog>

    <el-dialog
      v-model="deleteDialogVisible"
      title="确认删除"
      width="400px"
    >
      <p>确定要删除任务「{{ taskToDelete?.title }}」吗？</p>
      <template #footer>
        <el-button @click="deleteDialogVisible = false">取消</el-button>
        <el-button type="danger" @click="confirmDelete">确定删除</el-button>
      </template>
    </el-dialog>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'
import { ElMessage } from 'element-plus'
import { Plus, Edit, Delete } from '@element-plus/icons-vue'

const STORAGE_KEY = 'kanban-tasks'

const columns = [
  { id: 'todo', label: '待办', tagType: 'info' },
  { id: 'inProgress', label: '进行中', tagType: 'warning' },
  { id: 'done', label: '已完成', tagType: 'success' }
]

const tasks = ref([])
const dialogVisible = ref(false)
const deleteDialogVisible = ref(false)
const isEdit = ref(false)
const submitting = ref(false)
const taskToDelete = ref(null)
const formRef = ref(null)

const dragStatus = ref({
  taskId: null,
  column: null,
  index: null,
  active: false
})

const formData = ref({
  id: null,
  title: '',
  status: 'todo',
  priority: 'medium',
  assignee: '',
  dueDate: null,
  description: ''
})

const formRules = {
  title: [{ required: true, message: '请输入任务标题', trigger: 'blur' }],
  status: [{ required: true, message: '请选择状态', trigger: 'change' }],
  priority: [{ required: true, message: '请选择优先级', trigger: 'change' }]
}

const getTasksByStatus = (status) => {
  return tasks.value.filter(task => task.status === status)
}

const getPriorityLabel = (priority) => {
  const map = { high: '高', medium: '中', low: '低' }
  return map[priority] || '中'
}

const formatDate = (date) => {
  if (!date) return ''
  return date
}

const getDueDateTagType = (dueDate) => {
  if (!dueDate) return 'info'
  const today = new Date()
  today.setHours(0, 0, 0, 0)
  const due = new Date(dueDate)
  due.setHours(0, 0, 0, 0)
  
  const diffTime = due.getTime() - today.getTime()
  const diffDays = Math.ceil(diffTime / (1000 * 60 * 60 * 24))
  
  if (diffDays < 0) return 'danger'
  if (diffDays <= 3) return 'warning'
  return 'success'
}

const saveToStorage = () => {
  try {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(tasks.value))
  } catch (e) {
    console.error('保存到本地存储失败:', e)
  }
}

const loadFromStorage = () => {
  try {
    const data = localStorage.getItem(STORAGE_KEY)
    if (data) {
      tasks.value = JSON.parse(data)
    }
  } catch (e) {
    console.error('从本地存储读取失败:', e)
  }
}

const generateId = () => {
  return 'task_' + Date.now() + '_' + Math.random().toString(36).substr(2, 9)
}

const resetForm = () => {
  formData.value = {
    id: null,
    title: '',
    status: 'todo',
    priority: 'medium',
    assignee: '',
    dueDate: null,
    description: ''
  }
  isEdit.value = false
  submitting.value = false
}

const handleAddTask = () => {
  resetForm()
  dialogVisible.value = true
}

const handleEditTask = (task) => {
  isEdit.value = true
  formData.value = { ...task }
  dialogVisible.value = true
}

const handleDeleteTask = (task) => {
  taskToDelete.value = task
  deleteDialogVisible.value = true
}

const confirmDelete = () => {
  const index = tasks.value.findIndex(t => t.id === taskToDelete.value.id)
  if (index > -1) {
    tasks.value.splice(index, 1)
    saveToStorage()
    ElMessage.success('删除成功')
  }
  deleteDialogVisible.value = false
  taskToDelete.value = null
}

const handleSubmit = async () => {
  if (!formRef.value) return
  
  await formRef.value.validate((valid) => {
    if (valid) {
      submitting.value = true
      
      if (isEdit.value) {
        const index = tasks.value.findIndex(t => t.id === formData.value.id)
        if (index > -1) {
          tasks.value[index] = { ...formData.value }
        }
        ElMessage.success('编辑成功')
      } else {
        const newTask = {
          ...formData.value,
          id: generateId(),
          createdAt: new Date().toISOString()
        }
        tasks.value.push(newTask)
        ElMessage.success('创建成功')
      }
      
      saveToStorage()
      dialogVisible.value = false
      submitting.value = false
    }
  })
}

const onDragStart = (task, columnId, index) => {
  dragStatus.value = {
    taskId: task.id,
    column: columnId,
    index: index,
    active: true
  }
}

const onDragEnd = () => {
  dragStatus.value.active = false
}

const onDragOver = (columnId) => {
  dragStatus.value.active = true
}

const onDragLeave = () => {
  dragStatus.value.active = false
}

const onDrop = (targetColumnId) => {
  const { taskId, column, index } = dragStatus.value
  
  if (!taskId || !column) {
    dragStatus.value.active = false
    return
  }
  
  const taskIndex = tasks.value.findIndex(t => t.id === taskId)
  if (taskIndex === -1) {
    dragStatus.value.active = false
    return
  }
  
  const task = tasks.value[taskIndex]
  
  if (column === targetColumnId) {
    const sameColumnTasks = getTasksByStatus(column)
    if (sameColumnTasks.length <= 1) {
      dragStatus.value.active = false
      return
    }
    
    const currentIndex = sameColumnTasks.findIndex(t => t.id === taskId)
    if (currentIndex === -1) {
      dragStatus.value.active = false
      return
    }
    
    let newIndex = currentIndex
    if (index < currentIndex) {
      newIndex = Math.max(0, currentIndex - 1)
    } else {
      newIndex = Math.min(sameColumnTasks.length - 1, currentIndex + 1)
    }
    
    if (newIndex !== currentIndex) {
      const taskToMove = sameColumnTasks[currentIndex]
      const allTasks = tasks.value
      
      const globalCurrentIndex = allTasks.findIndex(t => t.id === taskToMove.id)
      const removed = allTasks.splice(globalCurrentIndex, 1)[0]
      
      const targetTask = sameColumnTasks[newIndex]
      const globalNewIndex = allTasks.findIndex(t => t.id === targetTask.id)
      
      if (newIndex < currentIndex) {
        allTasks.splice(globalNewIndex, 0, removed)
      } else {
        allTasks.splice(globalNewIndex + 1, 0, removed)
      }
    }
  } else {
    task.status = targetColumnId
  }
  
  saveToStorage()
  dragStatus.value.active = false
  
  ElMessage.success('任务已移动')
}

onMounted(() => {
  loadFromStorage()
})
</script>

<style scoped>
.kanban-container {
  min-height: 100vh;
  background-color: #f0f2f5;
}

.kanban-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 16px 24px;
  background-color: #fff;
  border-bottom: 1px solid #e8e8e8;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.04);
}

.kanban-header h1 {
  font-size: 22px;
  font-weight: 600;
  color: #1f2937;
  margin: 0;
}

.kanban-content {
  display: flex;
  gap: 16px;
  padding: 24px;
  overflow-x: auto;
  min-height: calc(100vh - 72px);
}

.kanban-column {
  flex: 1;
  min-width: 300px;
  max-width: 400px;
  background-color: #f9fafb;
  border-radius: 8px;
  border: 1px solid #e5e7eb;
  padding: 12px;
  display: flex;
  flex-direction: column;
}

.column-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 8px 0 12px;
  border-bottom: 1px solid #e5e7eb;
  margin-bottom: 12px;
}

.column-title {
  font-size: 15px;
  font-weight: 600;
  color: #374151;
}

.column-tasks {
  flex: 1;
  min-height: 100px;
  padding: 4px;
  border-radius: 4px;
  transition: background-color 0.2s;
}

.column-tasks.drag-over {
  background-color: #e0e7ff;
  border: 2px dashed #6366f1;
}

.task-card {
  background-color: #fff;
  border-radius: 6px;
  padding: 12px;
  margin-bottom: 8px;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.08);
  border: 1px solid #e5e7eb;
  cursor: grab;
  transition: box-shadow 0.2s, transform 0.2s;
}

.task-card:hover {
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.12);
  transform: translateY(-2px);
}

.task-card.dragging {
  opacity: 0.5;
  cursor: grabbing;
}

.task-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 8px;
}

.task-priority {
  font-size: 12px;
  padding: 2px 8px;
  border-radius: 4px;
  font-weight: 500;
}

.priority-high {
  background-color: #fef2f2;
  color: #dc2626;
}

.priority-medium {
  background-color: #fffbeb;
  color: #d97706;
}

.priority-low {
  background-color: #f0fdf4;
  color: #16a34a;
}

.task-actions {
  display: flex;
  gap: 4px;
  opacity: 0;
  transition: opacity 0.2s;
}

.task-card:hover .task-actions {
  opacity: 1;
}

.task-actions .el-button--text {
  padding: 2px 4px;
}

.task-actions .el-button--text.danger {
  color: #dc2626;
}

.task-actions .el-button--text:hover {
  color: #4f46e5;
}

.task-title {
  font-size: 14px;
  font-weight: 500;
  color: #111827;
  margin-bottom: 6px;
  line-height: 1.4;
}

.task-description {
  font-size: 13px;
  color: #6b7280;
  line-height: 1.5;
  margin-bottom: 10px;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
}

.task-footer {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
}

.empty-hint {
  text-align: center;
  color: #9ca3af;
  font-size: 13px;
  padding: 20px;
  border: 2px dashed #e5e7eb;
  border-radius: 6px;
}

:deep(.el-dialog) {
  border-radius: 8px;
}

:deep(.el-dialog__header) {
  padding: 16px 20px;
  border-bottom: 1px solid #e5e7eb;
  margin-right: 0;
}

:deep(.el-dialog__body) {
  padding: 20px;
}

:deep(.el-dialog__footer) {
  padding: 12px 20px;
  border-top: 1px solid #e5e7eb;
}

:deep(.el-form-item__label) {
  font-weight: 500;
  color: #374151;
}
</style>
