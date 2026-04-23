<template>
  <div class="kanban-container">
    <header class="kanban-header">
      <div class="header-inner">
        <div class="header-left">
          <div class="logo-icon">
            <el-icon size="24"><Tickets /></el-icon>
          </div>
          <h1>团队任务看板</h1>
        </div>
        <el-button type="primary" @click="handleAddTask" :icon="Plus" size="large">新建任务</el-button>
      </div>
    </header>

    <main class="kanban-main">
      <div class="kanban-content">
        <div
          v-for="column in columns"
          :key="column.id"
          class="kanban-column"
        >
          <div class="column-header" :style="{ borderLeftColor: column.color }">
            <div class="column-left">
              <span class="column-dot" :style="{ backgroundColor: column.color }"></span>
              <span class="column-title">{{ column.label }}</span>
            </div>
            <span class="column-count">{{ getTasksByStatus(column.id).length }}</span>
          </div>

          <div
            class="column-tasks"
            :class="{ 'drag-over': dragStatus.active && dragStatus.targetColumn === column.id }"
            @dragover.prevent="onDragOver(column.id)"
            @dragleave="onDragLeave"
            @drop="onDrop(column.id)"
          >
            <div
              v-for="(task, index) in getTasksByStatus(column.id)"
              :key="task.id"
              class="task-card"
              draggable="true"
              :class="{ 
                'dragging': dragStatus.taskId === task.id,
                'completed': task.status === 'done'
              }"
              @dragstart="onDragStart(task, column.id, index)"
              @dragend="onDragEnd"
            >
              <div class="task-header">
                <div class="task-priority-badge" :class="'priority-' + task.priority">
                  <span>{{ getPriorityLabel(task.priority) }}</span>
                </div>
                <div class="task-actions">
                  <button class="action-btn edit-btn" @click.stop="handleEditTask(task)" title="编辑任务">
                    <el-icon :size="16"><Edit /></el-icon>
                  </button>
                  <button class="action-btn delete-btn" @click.stop="handleDeleteTask(task)" title="删除任务">
                    <el-icon :size="16"><Delete /></el-icon>
                  </button>
                </div>
              </div>

              <div class="task-content">
                <div class="task-title">{{ task.title }}</div>
                <div v-if="task.description" class="task-description">{{ task.description }}</div>
              </div>

              <div class="task-footer">
                <div class="footer-left">
                  <el-tag v-if="task.dueDate" :type="getDueDateTagType(task.dueDate)" effect="light" size="small">
                    <el-icon><Calendar /></el-icon>
                    <span>{{ formatDate(task.dueDate) }}</span>
                  </el-tag>
                </div>
                <div class="footer-right">
                  <div v-if="task.assignee" class="assignee-badge">
                    <el-avatar :size="20" class="assignee-avatar">
                      {{ getInitial(task.assignee) }}
                    </el-avatar>
                    <span class="assignee-name">{{ task.assignee }}</span>
                  </div>
                </div>
              </div>
            </div>

            <div v-if="getTasksByStatus(column.id).length === 0" class="empty-state">
              <el-icon size="40" class="empty-icon"><Document /></el-icon>
              <span class="empty-text">暂无任务</span>
              <span class="empty-hint">拖拽任务到此处</span>
            </div>
          </div>
        </div>
      </div>
    </main>

    <el-dialog
      v-model="dialogVisible"
      :title="isEdit ? '编辑任务' : '新建任务'"
      width="560px"
      :close-on-click-modal="false"
      @closed="handleDialogClosed"
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
          <el-radio-group v-model="formData.priority">
            <el-radio-button value="high">高</el-radio-button>
            <el-radio-button value="medium">中</el-radio-button>
            <el-radio-button value="low">低</el-radio-button>
          </el-radio-group>
        </el-form-item>

        <el-row :gutter="16">
          <el-col :span="12">
            <el-form-item label="负责人" prop="assignee">
              <el-input v-model="formData.assignee" placeholder="负责人姓名" maxlength="20" clearable />
            </el-form-item>
          </el-col>
          <el-col :span="12">
            <el-form-item label="截止日期" prop="dueDate">
              <el-date-picker
                v-model="formData.dueDate"
                type="date"
                placeholder="选择日期"
                style="width: 100%"
                value-format="YYYY-MM-DD"
              />
            </el-form-item>
          </el-col>
        </el-row>

        <el-form-item label="描述" prop="description">
          <el-input
            v-model="formData.description"
            type="textarea"
            :rows="3"
            placeholder="添加任务描述（可选）"
            maxlength="200"
            show-word-limit
          />
        </el-form-item>
      </el-form>

      <template #footer>
        <el-button @click="dialogVisible = false" size="large">取消</el-button>
        <el-button type="primary" @click="handleSubmit" :loading="submitting" size="large">
          {{ isEdit ? '保存修改' : '创建任务' }}
        </el-button>
      </template>
    </el-dialog>

    <el-dialog
      v-model="deleteDialogVisible"
      title="确认删除"
      width="420px"
    >
      <div class="delete-warning">
        <el-icon size="40" color="#F56C6C"><Warning /></el-icon>
        <div class="warning-content">
          <p class="warning-title">确定要删除这个任务吗？</p>
          <p class="warning-desc">任务「{{ taskToDelete?.title }}」将被永久删除，此操作无法撤销。</p>
        </div>
      </div>
      <template #footer>
        <el-button @click="deleteDialogVisible = false" size="large">取消</el-button>
        <el-button type="danger" @click="confirmDelete" size="large">确认删除</el-button>
      </template>
    </el-dialog>
  </div>
</template>

<script setup>
import { ref, nextTick, onMounted } from 'vue'
import { ElMessage } from 'element-plus'
import { 
  Plus, Edit, Delete, Warning, Calendar, User, Tickets, Document 
} from '@element-plus/icons-vue'

const STORAGE_KEY = 'kanban-tasks'

const columns = [
  { id: 'todo', label: '待办', color: '#3B82F6' },
  { id: 'inProgress', label: '进行中', color: '#F59E0B' },
  { id: 'done', label: '已完成', color: '#10B981' }
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
  sourceColumn: null,
  targetColumn: null,
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
  const columnTasks = tasks.value.filter(task => task.status === status)
  
  columnTasks.sort((a, b) => {
    const priorityOrder = { high: 3, medium: 2, low: 1 }
    const priorityDiff = priorityOrder[b.priority] - priorityOrder[a.priority]
    if (priorityDiff !== 0) return priorityDiff
    
    return new Date(a.createdAt) - new Date(b.createdAt)
  })
  
  return columnTasks
}

const getPriorityLabel = (priority) => {
  const map = { high: '高优先级', medium: '中优先级', low: '低优先级' }
  return map[priority] || '中优先级'
}

const getInitial = (name) => {
  if (!name) return '?'
  return name.charAt(0).toUpperCase()
}

const formatDate = (date) => {
  if (!date) return ''
  const d = new Date(date)
  const month = (d.getMonth() + 1).toString().padStart(2, '0')
  const day = d.getDate().toString().padStart(2, '0')
  return month + '月' + day + '日'
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

const handleDialogClosed = () => {
  if (formRef.value) {
    formRef.value.resetFields()
  }
  resetForm()
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
    ElMessage.success('任务已删除')
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
        ElMessage.success('任务已更新')
      } else {
        const newTask = {
          ...formData.value,
          id: generateId(),
          createdAt: new Date().toISOString()
        }
        tasks.value.push(newTask)
        ElMessage.success('任务创建成功')
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
    sourceColumn: columnId,
    targetColumn: null,
    active: true
  }
}

const onDragEnd = () => {
  dragStatus.value.active = false
}

const onDragOver = (columnId) => {
  dragStatus.value.active = true
  dragStatus.value.targetColumn = columnId
}

const onDragLeave = () => {
}

const reorderTasks = (taskId, sourceColumnId, targetColumnId) => {
  const taskIndex = tasks.value.findIndex(t => t.id === taskId)
  if (taskIndex === -1) return false
  
  const task = tasks.value[taskIndex]
  
  if (sourceColumnId === targetColumnId) {
    const columnTasks = tasks.value.filter(t => t.status === sourceColumnId)
    if (columnTasks.length <= 1) return false
    
    const currentIndex = columnTasks.findIndex(t => t.id === taskId)
    
    const taskToMove = tasks.value.splice(taskIndex, 1)[0]
    
    let newIndex = currentIndex + 1
    if (newIndex >= columnTasks.length) {
      newIndex = currentIndex - 1
    }
    if (newIndex < 0) {
      return false
    }
    
    let insertGlobalIndex = -1
    const updatedColumnTasks = tasks.value.filter(t => t.status === sourceColumnId)
    
    if (updatedColumnTasks.length === 0) {
      tasks.value.push(taskToMove)
    } else {
      if (newIndex >= updatedColumnTasks.length) {
        const lastTask = updatedColumnTasks[updatedColumnTasks.length - 1]
        insertGlobalIndex = tasks.value.findIndex(t => t.id === lastTask.id)
        if (insertGlobalIndex !== -1) {
          tasks.value.splice(insertGlobalIndex + 1, 0, taskToMove)
        } else {
          tasks.value.push(taskToMove)
        }
      } else {
        const targetTask = updatedColumnTasks[newIndex]
        insertGlobalIndex = tasks.value.findIndex(t => t.id === targetTask.id)
        if (insertGlobalIndex !== -1) {
          if (newIndex > currentIndex) {
            tasks.value.splice(insertGlobalIndex + 1, 0, taskToMove)
          } else {
            tasks.value.splice(insertGlobalIndex, 0, taskToMove)
          }
        } else {
          tasks.value.push(taskToMove)
        }
      }
    }
    
    return true
    
  } else {
    task.status = targetColumnId
    
    const otherTasks = tasks.value.filter(t => t.id !== taskId)
    
    tasks.value = [...otherTasks, task]
    
    return true
  }
}

const onDrop = (targetColumnId) => {
  const { taskId, sourceColumn } = dragStatus.value
  
  if (!taskId || !sourceColumn) {
    dragStatus.value.active = false
    return
  }
  
  const success = reorderTasks(taskId, sourceColumn, targetColumnId)
  
  if (success) {
    saveToStorage()
    
    if (sourceColumn !== targetColumnId) {
      const columnLabel = columns.find(c => c.id === targetColumnId)?.label || ''
      ElMessage.success('任务已移动到「' + columnLabel + '」')
    } else {
      ElMessage.success('任务顺序已调整')
    }
  }
  
  dragStatus.value.active = false
}

onMounted(() => {
  loadFromStorage()
})
</script>

<style scoped>
.kanban-container {
  min-height: 100vh;
  background: linear-gradient(180deg, #F8FAFC 0%, #F1F5F9 100%);
  display: flex;
  flex-direction: column;
}

.kanban-header {
  position: sticky;
  top: 0;
  z-index: 100;
  background: rgba(255, 255, 255, 0.95);
  backdrop-filter: blur(10px);
  border-bottom: 1px solid #E2E8F0;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.04);
}

.header-inner {
  max-width: 1400px;
  margin: 0 auto;
  padding: 16px 24px;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.header-left {
  display: flex;
  align-items: center;
  gap: 12px;
}

.logo-icon {
  width: 40px;
  height: 40px;
  background: linear-gradient(135deg, #3B82F6 0%, #6366F1 100%);
  border-radius: 10px;
  display: flex;
  align-items: center;
  justify-content: center;
  color: white;
  box-shadow: 0 4px 12px rgba(59, 130, 246, 0.3);
}

.kanban-header h1 {
  font-size: 20px;
  font-weight: 600;
  color: #1E293B;
  margin: 0;
  letter-spacing: -0.02em;
}

.kanban-main {
  flex: 1;
  overflow: auto;
  padding: 24px;
}

.kanban-content {
  display: flex;
  gap: 24px;
  max-width: 1400px;
  margin: 0 auto;
  min-width: fit-content;
}

.kanban-column {
  flex: 1;
  min-width: 340px;
  max-width: 420px;
  background: #FFFFFF;
  border-radius: 12px;
  border: 1px solid #E2E8F0;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.06);
  display: flex;
  flex-direction: column;
  height: fit-content;
  max-height: calc(100vh - 140px);
  overflow: hidden;
}

.column-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 16px 16px 12px;
  border-bottom: 1px solid #F1F5F9;
  border-left: 4px solid;
  background: #FAFBFC;
  border-radius: 12px 12px 0 0;
  margin: -1px -1px 0 -1px;
}

.column-left {
  display: flex;
  align-items: center;
  gap: 8px;
}

.column-dot {
  width: 10px;
  height: 10px;
  border-radius: 50%;
}

.column-title {
  font-size: 14px;
  font-weight: 600;
  color: #334155;
}

.column-count {
  font-size: 12px;
  font-weight: 600;
  color: #64748B;
  background: #F1F5F9;
  padding: 4px 10px;
  border-radius: 12px;
  min-width: 24px;
  text-align: center;
}

.column-tasks {
  flex: 1;
  min-height: 80px;
  padding: 12px;
  overflow-y: auto;
  transition: all 0.2s ease;
}

.column-tasks.drag-over {
  background: #EEF2FF;
  border: 2px dashed #6366F1;
  border-radius: 8px;
  margin: 4px;
}

.task-card {
  background: #FFFFFF;
  border-radius: 10px;
  padding: 16px;
  margin-bottom: 12px;
  border: 1px solid #E2E8F0;
  cursor: grab;
  transition: all 0.2s ease;
  position: relative;
}

.task-card:last-child {
  margin-bottom: 0;
}

.task-card:hover {
  border-color: #CBD5E1;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.08);
  transform: translateY(-1px);
}

.task-card:active {
  cursor: grabbing;
}

.task-card.dragging {
  opacity: 0.3;
  transform: scale(1.02);
}

.task-card.completed {
  background: #FCFDFE;
}

.task-card.completed .task-title {
  color: #94A3B8;
}

.task-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 10px;
}

.task-priority-badge {
  font-size: 11px;
  font-weight: 500;
  padding: 4px 10px;
  border-radius: 6px;
}

.task-priority-badge.priority-high {
  background: #FEF2F2;
  color: #DC2626;
}

.task-priority-badge.priority-medium {
  background: #FFFBEB;
  color: #D97706;
}

.task-priority-badge.priority-low {
  background: #F0FDF4;
  color: #16A34A;
}

.task-actions {
  display: flex;
  gap: 6px;
  opacity: 0;
  transition: opacity 0.2s ease;
}

.task-card:hover .task-actions {
  opacity: 1;
}

.action-btn {
  width: 28px;
  height: 28px;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.2s ease;
}

.edit-btn {
  background: #EFF6FF;
  color: #3B82F6;
}

.edit-btn:hover {
  background: #DBEAFE;
  color: #2563EB;
}

.delete-btn {
  background: #FEF2F2;
  color: #EF4444;
}

.delete-btn:hover {
  background: #FEE2E2;
  color: #DC2626;
}

.task-content {
  margin-bottom: 12px;
}

.task-title {
  font-size: 14px;
  font-weight: 500;
  color: #1E293B;
  line-height: 1.5;
  margin-bottom: 6px;
}

.task-description {
  font-size: 12px;
  color: #64748B;
  line-height: 1.5;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
}

.task-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding-top: 10px;
  border-top: 1px solid #F1F5F9;
}

.footer-left,
.footer-right {
  display: flex;
  align-items: center;
  gap: 8px;
}

.assignee-badge {
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 3px 8px;
  background: #F8FAFC;
  border-radius: 6px;
  border: 1px solid #E2E8F0;
}

.assignee-avatar {
  background: linear-gradient(135deg, #6366F1 0%, #8B5CF6 100%);
  color: white;
  font-size: 10px;
  font-weight: 600;
}

.assignee-name {
  font-size: 11px;
  color: #475569;
  font-weight: 500;
}

.empty-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 40px 20px;
  color: #94A3B8;
}

.empty-icon {
  margin-bottom: 12px;
  color: #CBD5E1;
}

.empty-text {
  font-size: 14px;
  font-weight: 500;
  margin-bottom: 4px;
}

.empty-hint {
  font-size: 12px;
  color: #CBD5E1;
}

.delete-warning {
  display: flex;
  align-items: flex-start;
  gap: 16px;
  padding: 8px 0;
}

.warning-content {
  flex: 1;
}

.warning-title {
  font-size: 14px;
  font-weight: 600;
  color: #1E293B;
  margin: 0 0 4px;
}

.warning-desc {
  font-size: 13px;
  color: #64748B;
  margin: 0;
  line-height: 1.5;
}
</style>

<style>
.el-dialog {
  border-radius: 16px;
  box-shadow: 0 25px 50px rgba(0, 0, 0, 0.12);
}

.el-dialog__header {
  padding: 20px 24px;
  border-bottom: 1px solid #F1F5F9;
  margin-right: 0;
}

.el-dialog__title {
  font-size: 16px;
  font-weight: 600;
  color: #1E293B;
}

.el-dialog__body {
  padding: 24px;
}

.el-dialog__footer {
  padding: 16px 24px;
  border-top: 1px solid #F1F5F9;
}

.el-form-item__label {
  font-weight: 500;
  color: #475569;
  font-size: 13px;
}

.el-input__wrapper,
.el-select__wrapper,
.el-textarea__inner {
  border-radius: 8px;
  border: 1px solid #D1D5DB !important;
  box-shadow: none;
}

.el-input__wrapper:hover,
.el-select__wrapper:hover {
  border-color: #9CA3AF !important;
}

.el-input__wrapper.is-focus,
.el-select__wrapper.is-focus {
  border-color: #3B82F6 !important;
  box-shadow: 0 0 0 2px rgba(59, 130, 246, 0.1) !important;
}

.el-date-editor .el-input__wrapper {
  border: 1px solid #D1D5DB !important;
}

.el-date-editor .el-input__wrapper:hover {
  border-color: #9CA3AF !important;
}

.el-date-editor .el-input__wrapper.is-focus {
  border-color: #3B82F6 !important;
}

.el-radio-button__original-radio:checked + .el-radio-button__inner {
  background: #3B82F6;
  border-color: #3B82F6;
}

.el-radio-button:first-child .el-radio-button__inner {
  border-radius: 8px 0 0 8px;
}

.el-radio-button:last-child .el-radio-button__inner {
  border-radius: 0 8px 8px 0;
}

.el-button--primary {
  background: linear-gradient(135deg, #3B82F6 0%, #2563EB 100%);
  border: none;
  box-shadow: 0 2px 6px rgba(59, 130, 246, 0.35);
}

.el-button--primary:hover {
  background: linear-gradient(135deg, #2563EB 0%, #1D4ED8 100%);
  box-shadow: 0 4px 12px rgba(59, 130, 246, 0.45);
}

.el-button--danger {
  box-shadow: 0 2px 6px rgba(245, 108, 108, 0.35);
}

.el-button--danger:hover {
  box-shadow: 0 4px 12px rgba(245, 108, 108, 0.45);
}
</style>
