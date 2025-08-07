<template>
  <div id="app" class="desktop-app">
    <!-- 标题栏 -->
    <div class="title-bar">
      <div class="title-bar-left">
        <div class="app-icon">
          <el-icon><Monitor /></el-icon>
        </div>
        <span class="app-title">程序项目助手</span>
      </div>
      <div class="title-bar-right">
        <div class="window-controls">
          <div class="control-btn minimize" @click="minimizeWindow"></div>
          <div class="control-btn maximize" @click="maximizeWindow"></div>
          <div class="control-btn close" @click="closeWindow"></div>
        </div>
      </div>
    </div>

    <!-- 菜单栏 -->
    <div class="menu-bar">
      <div class="menu-item" @click="showAddDialog = true">
        <el-icon><Plus /></el-icon>
        <span>新建程序</span>
      </div>
      <div class="menu-item" @click="refreshPrograms">
        <el-icon><Refresh /></el-icon>
        <span>刷新</span>
      </div>
      <div class="menu-item" @click="showSettings = true">
        <el-icon><Setting /></el-icon>
        <span>设置</span>
      </div>
      <div class="menu-item" @click="showAbout = true">
        <el-icon><InfoFilled /></el-icon>
        <span>关于</span>
      </div>
    </div>

    <!-- 工具栏 -->
    <div class="toolbar">
      <div class="toolbar-left">
        <el-button-group>
          <!-- <el-button size="small" @click="showAddDialog = true" :icon="Plus">新建</el-button> -->
          <el-button size="small" @click="runAllPrograms" :icon="VideoPlay">全部运行</el-button>
          <el-button size="small" @click="stopAllPrograms" :icon="VideoPause">全部停止</el-button>
        </el-button-group>
      </div>
      <div class="toolbar-right">
        <el-input
          v-model="searchText"
          placeholder="搜索程序..."
          size="small"
          style="width: 200px"
          :prefix-icon="Search"
          clearable
        />
      </div>
    </div>

    <!-- 主内容区域 -->
    <div class="main-content">
      <!-- 侧边栏 -->
      <div class="sidebar">
        <div class="sidebar-section">
          <div class="section-title">程序分类</div>
          <div class="category-list">
            <div 
              class="category-item" 
              :class="{ active: selectedCategory === 'all' }"
              @click="selectedCategory = 'all'"
            >
              <el-icon><Grid /></el-icon>
              <span>全部程序 ({{ programs.length }})</span>
            </div>
            <div 
              class="category-item"
              :class="{ active: selectedCategory === 'running' }"
              @click="selectedCategory = 'running'"
            >
              <el-icon><VideoPlay /></el-icon>
              <span>运行中 ({{ runningCount }})</span>
            </div>
            <div 
              class="category-item"
              :class="{ active: selectedCategory === 'stopped' }"
              @click="selectedCategory = 'stopped'"
            >
              <el-icon><VideoPause /></el-icon>
              <span>已停止 ({{ stoppedCount }})</span>
            </div>
          </div>
        </div>
      </div>

      <!-- 程序列表区域 -->
      <div class="content-area">
        <div v-if="filteredPrograms.length === 0" class="empty-state">
          <div class="empty-icon">
            <el-icon><DocumentAdd /></el-icon>
          </div>
          <div class="empty-text">暂无程序</div>
          <div class="empty-desc">点击"新建程序"开始添加您的第一个程序</div>
          <el-button type="primary" @click="showAddDialog = true" :icon="Plus">
            新建程序
          </el-button>
        </div>

        <div v-else class="program-list">
          <div 
            v-for="program in filteredPrograms" 
            :key="program.id" 
            class="program-item"
            :class="{ running: program.running }"
          >
            <div class="program-icon">
              <el-icon v-if="program.running"><VideoPlay /></el-icon>
              <el-icon v-else><Document /></el-icon>
            </div>
            
            <div class="program-details">
              <div class="program-name">{{ program.name }}</div>
              <div class="program-path">{{ program.path }}</div>
              <div class="program-command">{{ program.command }}</div>
              <div class="program-description">{{ program.description }}</div>
            </div>
            
            <div class="program-status">
              <div class="status-indicator" :class="{ active: program.running }"></div>
              <span class="status-text">{{ program.running ? '运行中' : '已停止' }}</span>
            </div>
            
            <div class="program-actions">
              <el-button 
                size="small" 
                :type="program.running ? 'danger' : 'primary'"
                @click="runProgram(program)"
                :icon="program.running ? VideoPause : VideoPlay"
              >
                {{ program.running ? '停止' : '运行' }}
              </el-button>
              <el-button 
                size="small" 
                :type="program.running ? 'primary' : 'info'"
                @click="showProgramLog(program)"
                :icon="Document"
              >
                运行日志
              </el-button>
              <el-button size="small" @click="editProgram(program)" :icon="Edit">
                编辑
              </el-button>
              <el-button 
                size="small" 
                type="danger" 
                @click="deleteProgram(program.id)"
                :icon="Delete"
              >
                删除
              </el-button>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- 状态栏 -->
    <div class="status-bar">
      <div class="status-left">
        <span>总计: {{ programs.length }} 个程序</span>
        <span class="separator">|</span>
        <span>运行中: {{ runningCount }} 个</span>
      </div>
      <div class="status-right">
        <span>{{ currentTime }}</span>
      </div>
    </div>

    <!-- 新增/编辑程序对话框 -->
    <el-dialog 
      v-model="showAddDialog" 
      :title="editingProgram ? '编辑程序' : '新建程序'"
      width="520px"
      :show-close="false"
      class="program-dialog"
      :modal="true"
      :close-on-click-modal="false"
      :close-on-press-escape="false"
    >
      <template #header="{ titleId, titleClass }">
        <div class="dialog-header">
          <div class="dialog-title">
            <el-icon class="dialog-icon">
              <Edit v-if="editingProgram" />
              <Plus v-else />
            </el-icon>
            <span :id="titleId" :class="titleClass">
              {{ editingProgram ? '编辑程序' : '新建程序' }}
            </span>
          </div>
          <el-button 
            class="dialog-close-btn" 
            :icon="Close" 
            size="small" 
            text 
            @click="cancelEdit"
          />
        </div>
      </template>
      
      <div class="dialog-content">
        <el-form 
          :model="programForm" 
          :rules="formRules" 
          ref="formRef" 
          label-width="70px"
          label-position="left"
          class="program-form"
        >
          <div class="form-section">
            <div class="section-title">基本信息</div>
            
            <el-form-item label="名称" prop="name" class="form-item">
              <el-input 
                v-model="programForm.name" 
                placeholder="请输入程序名称"
                size="default"
                :prefix-icon="Document"
              />
            </el-form-item>
            
            <el-form-item label="描述" prop="description" class="form-item">
              <el-input 
                v-model="programForm.description" 
                placeholder="请输入程序描述（可选）"
                type="textarea"
                :rows="2"
                resize="none"
                maxlength="200"
                show-word-limit
              />
            </el-form-item>
          </div>
          
          <div class="form-section">
            <div class="section-title">运行配置</div>
            
            <el-form-item label="路径" prop="path" class="form-item">
              <el-input 
                v-model="programForm.path" 
                placeholder="请输入程序运行路径"
                size="default"
                :prefix-icon="Folder"
              >
                <template #append>
                  <el-button @click="selectPath" :icon="FolderOpened" size="small">
                    浏览
                  </el-button>
                </template>
              </el-input>
            </el-form-item>
            
            <el-form-item label="命令" prop="command" class="form-item">
              <el-input 
                v-model="programForm.command" 
                placeholder="请输入程序运行命令"
                type="textarea"
                :rows="3"
                resize="none"
                maxlength="500"
                show-word-limit
              />
              <div class="form-tip">
                <el-icon><InfoFilled /></el-icon>
                <span>支持单行或多行命令。例如: npm start 或多行命令（每行一个命令）</span>
              </div>
            </el-form-item>
          </div>
        </el-form>
      </div>
      
      <template #footer>
        <div class="dialog-footer">
          <el-button @click="cancelEdit" size="default">
            <el-icon><Close /></el-icon>
            <span>取消</span>
          </el-button>
          <el-button type="primary" @click="saveProgram" size="default">
            <el-icon><Check /></el-icon>
            <span>{{ editingProgram ? '更新' : '保存' }}</span>
          </el-button>
        </div>
      </template>
    </el-dialog>

    <!-- 设置对话框 -->
    <el-dialog v-model="showSettings" title="设置" width="400px">
      <div class="settings-content">
        <div class="setting-item">
          <span>自动保存</span>
          <el-switch v-model="autoSave" />
        </div>
        <div class="setting-item">
          <span>启动时最小化</span>
          <el-switch v-model="minimizeOnStart" />
        </div>
      </div>
      <template #footer>
        <el-button type="primary" @click="showSettings = false">确定</el-button>
      </template>
    </el-dialog>

    <!-- 关于对话框 -->
    <el-dialog v-model="showAbout" title="关于" width="350px">
      <div class="about-content">
        <div class="about-icon">
          <el-icon size="48"><Monitor /></el-icon>
        </div>
        <h3>程序项目助手</h3>
        <p>版本: 1.0.0</p>
        <p>作者: DogeJian</p>
        <p>一个简单易用的程序管理工具</p>
      </div>
      <!-- <template #footer>
        <el-button type="primary" @click="showAbout = false">确定</el-button>
      </template> -->
    </el-dialog>
  </div>
</template>

<script>
import { ref, reactive, onMounted, onUnmounted, computed } from 'vue'
import { ElMessage, ElMessageBox } from 'element-plus'
import { 
  Plus, VideoPlay, VideoPause, Edit, Delete, Folder, 
  Monitor, Refresh, Setting, InfoFilled, Search, Grid,
  Document, DocumentAdd, Close, Check, FolderOpened
} from '@element-plus/icons-vue'

export default {
  name: 'App',
  setup() {
    const programs = ref([])
    const showAddDialog = ref(false)
    const showSettings = ref(false)
    const showAbout = ref(false)
    const editingProgram = ref(null)
    const formRef = ref()
    const searchText = ref('')
    const selectedCategory = ref('all')
    const currentTime = ref('')
    const autoSave = ref(true)
    const minimizeOnStart = ref(false)
    
    const programForm = reactive({
      name: '',
      path: '',
      command: '',
      description: ''
    })
    
    const formRules = {
      name: [
        { required: true, message: '请输入程序名称', trigger: 'blur' }
      ],
      path: [
        { required: true, message: '请输入运行路径', trigger: 'blur' }
      ],
      command: [
        { required: true, message: '请输入运行命令', trigger: 'blur' }
      ]
    }

    // 计算属性
    const runningCount = computed(() => programs.value.filter(p => p.running).length)
    const stoppedCount = computed(() => programs.value.filter(p => !p.running).length)
    
    const filteredPrograms = computed(() => {
      let filtered = programs.value
      
      // 按分类筛选
      if (selectedCategory.value === 'running') {
        filtered = filtered.filter(p => p.running)
      } else if (selectedCategory.value === 'stopped') {
        filtered = filtered.filter(p => !p.running)
      }
      
      // 按搜索文本筛选
      if (searchText.value) {
        const search = searchText.value.toLowerCase()
        filtered = filtered.filter(p => 
          p.name.toLowerCase().includes(search) ||
          p.path.toLowerCase().includes(search) ||
          p.command.toLowerCase().includes(search) ||
          p.description.toLowerCase().includes(search)
        )
      }
      
      return filtered
    })

    // 更新时间
    const updateTime = () => {
      const now = new Date()
      currentTime.value = now.toLocaleTimeString('zh-CN', { 
        hour12: false,
        hour: '2-digit',
        minute: '2-digit',
        second: '2-digit'
      })
    }
    
    // 加载保存的程序数据
    const loadPrograms = () => {
      const saved = localStorage.getItem('programs')
      if (saved) {
        programs.value = JSON.parse(saved).map(p => ({
          ...p,
          running: false
        }))
      }
    }
    
    // 保存程序数据
    const savePrograms = () => {
      if (!autoSave.value) return
      
      const dataToSave = programs.value.map(p => ({
        id: p.id,
        name: p.name,
        path: p.path,
        command: p.command,
        description: p.description
      }))
      localStorage.setItem('programs', JSON.stringify(dataToSave))
    }
    
    // 重置表单
    const resetForm = () => {
      programForm.name = ''
      programForm.path = ''
      programForm.command = ''
      programForm.description = ''
      editingProgram.value = null
    }
    
    // 保存程序
    const saveProgram = async () => {
      if (!formRef.value) return
      
      try {
        await formRef.value.validate()
        
        if (editingProgram.value) {
          const index = programs.value.findIndex(p => p.id === editingProgram.value.id)
          if (index !== -1) {
            programs.value[index] = {
              ...programs.value[index],
              ...programForm
            }
          }
          ElMessage.success('程序信息更新成功')
        } else {
          const newProgram = {
            id: Date.now(),
            ...programForm,
            running: false
          }
          programs.value.push(newProgram)
          ElMessage.success('程序添加成功')
        }
        
        savePrograms()
        showAddDialog.value = false
        resetForm()
      } catch (error) {
        console.error('表单验证失败:', error)
      }
    }
    
    // 编辑程序
    const editProgram = (program) => {
      editingProgram.value = program
      programForm.name = program.name
      programForm.path = program.path
      programForm.command = program.command
      programForm.description = program.description
      showAddDialog.value = true
    }
    
    // 取消编辑
    const cancelEdit = () => {
      showAddDialog.value = false
      resetForm()
    }
    
    // 删除程序
    const deleteProgram = async (id) => {
      try {
        await ElMessageBox.confirm('确定要删除这个程序吗？', '确认删除', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        })
        
        programs.value = programs.value.filter(p => p.id !== id)
        savePrograms()
        ElMessage.success('删除成功')
      } catch {
        // 用户取消删除
      }
    }
    
    // 选择路径
    const selectPath = () => {
      if (window.electronAPI) {
        window.electronAPI.selectDirectory().then(path => {
          if (path) {
            programForm.path = path
          }
        })
      } else {
        ElMessage.warning('请在Electron环境中使用此功能')
      }
    }
    
    // 运行程序
    const runProgram = (program) => {
      if (program.running) {
        // 停止程序
        if (window.electronAPI) {
          window.electronAPI.stopProgram(program.id)
            .then((result) => {
              if (result.success) {
                // 状态更新会通过 onProgramStopped 事件处理
                console.log(`程序 ${program.name} 停止请求已发送`)
              } else {
                ElMessage.error(`停止失败: ${result.error || '未知错误'}`)
              }
            })
            .catch(error => {
              ElMessage.error(`停止失败: ${error.message || '未知错误'}`)
            })
        } else {
          program.running = false
          ElMessage.success(`程序 ${program.name} 已停止`)
        }
        return
      }
      
      // 启动程序
      if (window.electronAPI) {
        window.electronAPI.runProgram({
          id: program.id,
          name: program.name,
          path: program.path,
          command: program.command
        }).then(() => {
          program.running = true
          ElMessage.success(`程序 ${program.name} 已启动`)
        }).catch(error => {
          program.running = false
          ElMessage.error(`启动失败: ${error.message || '未知错误'}`)
        })
      } else {
        program.running = true
        setTimeout(() => {
          ElMessage.success(`程序 ${program.name} 已启动`)
        }, 500)
      }
    }

    // 运行所有程序
    const runAllPrograms = () => {
      const stoppedPrograms = programs.value.filter(p => !p.running)
      if (stoppedPrograms.length === 0) {
        ElMessage.warning('没有可运行的程序')
        return
      }
      
      stoppedPrograms.forEach(program => {
        runProgram(program)
      })
    }

    // 停止所有程序
    const stopAllPrograms = async () => {
      const runningPrograms = programs.value.filter(p => p.running)
      if (runningPrograms.length === 0) {
        ElMessage.warning('没有正在运行的程序')
        return
      }
      
      if (window.electronAPI) {
        // 使用Promise.all等待所有程序停止
        const stopPromises = runningPrograms.map(program => 
          window.electronAPI.stopProgram(program.id)
            .then(() => {
              program.running = false
              return true
            })
            .catch(error => {
              console.error(`停止程序 ${program.name} 失败:`, error)
              return false
            })
        )
        
        try {
          await Promise.all(stopPromises)
          ElMessage.success('所有程序已停止')
        } catch (error) {
          ElMessage.error('部分程序停止失败')
        }
      } else {
        // 非Electron环境
        runningPrograms.forEach(program => {
          program.running = false
        })
        ElMessage.success('所有程序已停止')
      }
    }

    // 显示程序日志
    const showProgramLog = async (program) => {
      if (window.electronAPI && window.electronAPI.showProgramLog) {
        try {
          await window.electronAPI.showProgramLog({
            id: program.id,
            name: program.name
          })
        } catch (error) {
          console.error('显示日志窗口失败:', error)
          ElMessage.error('无法显示日志窗口')
        }
      } else {
        ElMessage.warning('请在Electron环境中使用此功能')
      }
    }

    // 刷新程序列表
    const refreshPrograms = async () => {
      loadPrograms()
      
      // 获取当前运行中的程序
      if (window.electronAPI) {
        try {
          const runningProgramIds = await window.electronAPI.getRunningPrograms()
          
          // 更新程序运行状态
          programs.value.forEach(program => {
            program.running = runningProgramIds.includes(program.id)
          })
          
          ElMessage.success('程序列表已刷新')
        } catch (error) {
          console.error('获取运行中程序失败:', error)
          ElMessage.error('刷新失败')
        }
      } else {
        ElMessage.success('程序列表已刷新')
      }
    }

    // 窗口控制方法
    const minimizeWindow = () => {
      if (window.electronAPI) {
        window.electronAPI.minimizeWindow()
      }
    }

    const maximizeWindow = () => {
      if (window.electronAPI) {
        window.electronAPI.maximizeWindow()
      }
    }

    const closeWindow = () => {
      if (window.electronAPI) {
        window.electronAPI.closeWindow()
      }
    }
    
    onMounted(() => {
      loadPrograms()
      updateTime()
      setInterval(updateTime, 1000)
      
      // 刷新程序列表并获取运行状态
      refreshPrograms()
      
      // 监听程序停止事件
      if (window.electronAPI) {
        window.electronAPI.onProgramStopped(data => {
          const program = programs.value.find(p => p.id === data.id)
          if (program) {
            program.running = false
            console.log(`程序 ${program.name} 已停止，退出码: ${data.code}`)
            
            // 显示停止消息
            if (data.code === 'STOPPED') {
              ElMessage.success(`程序 ${program.name} 已停止`)
            } else if (data.code === 'TERMINATED') {
              ElMessage.info(`程序 ${program.name} 已终止（日志窗口关闭）`)
            }
          }
        })
      }
    })
    
    onUnmounted(() => {
      // 移除所有监听器
      if (window.electronAPI && window.electronAPI.removeAllListeners) {
        window.electronAPI.removeAllListeners()
      }
    })
    
    return {
      programs,
      showAddDialog,
      showSettings,
      showAbout,
      editingProgram,
      programForm,
      formRules,
      formRef,
      searchText,
      selectedCategory,
      currentTime,
      autoSave,
      minimizeOnStart,
      runningCount,
      stoppedCount,
      filteredPrograms,
      saveProgram,
      editProgram,
      cancelEdit,
      deleteProgram,
      selectPath,
      runProgram,
      runAllPrograms,
      stopAllPrograms,
      showProgramLog,
      refreshPrograms,
      minimizeWindow,
      maximizeWindow,
      closeWindow,
      Plus, VideoPlay, VideoPause, Edit, Delete, Folder,
      Monitor, Refresh, Setting, InfoFilled, Search, Grid,
      Document, DocumentAdd, Close, Check, FolderOpened
    }
  }
}
</script>

<style scoped>
/* 桌面应用整体样式 */
.desktop-app {
  height: 100vh;
  background: #f5f5f5;
  display: flex;
  flex-direction: column;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  overflow: hidden;
}

/* 标题栏 */
.title-bar {
  height: 32px;
  background: linear-gradient(to bottom, #e8e8e8, #d0d0d0);
  border-bottom: 1px solid #b0b0b0;
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0 12px;
  user-select: none;
  -webkit-app-region: drag;
}

.title-bar-left {
  display: flex;
  align-items: center;
  gap: 8px;
}

.app-icon {
  color: #666;
  font-size: 16px;
}

.app-title {
  font-size: 13px;
  color: #333;
  font-weight: 500;
}

.window-controls {
  display: flex;
  gap: 8px;
  -webkit-app-region: no-drag;
}

.control-btn {
  width: 12px;
  height: 12px;
  border-radius: 50%;
  cursor: pointer;
}

.control-btn.close { background: #ff5f57; }
.control-btn.minimize { background: #ffbd2e; }
.control-btn.maximize { background: #28ca42; }

/* 菜单栏 */
.menu-bar {
  height: 28px;
  background: #f0f0f0;
  border-bottom: 1px solid #d0d0d0;
  display: flex;
  align-items: center;
  padding: 0 8px;
}

.menu-item {
  display: flex;
  align-items: center;
  gap: 4px;
  padding: 4px 12px;
  font-size: 13px;
  color: #333;
  cursor: pointer;
  border-radius: 4px;
  transition: background-color 0.2s;
}

.menu-item:hover {
  background: #e0e0e0;
}

/* 工具栏 */
.toolbar {
  height: 40px;
  background: #fafafa;
  border-bottom: 1px solid #e0e0e0;
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0 16px;
}

.toolbar-left, .toolbar-right {
  display: flex;
  align-items: center;
  gap: 8px;
}

/* 主内容区域 */
.main-content {
  flex: 1;
  display: flex;
  overflow: hidden;
}

/* 侧边栏 */
.sidebar {
  width: 200px;
  background: #f8f8f8;
  border-right: 1px solid #e0e0e0;
  padding: 16px 0;
}

.sidebar-section {
  margin-bottom: 24px;
}

.section-title {
  font-size: 12px;
  color: #666;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  margin-bottom: 8px;
  padding: 0 16px;
}

.category-list {
  display: flex;
  flex-direction: column;
}

.category-item {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 8px 16px;
  font-size: 13px;
  color: #555;
  cursor: pointer;
  transition: all 0.2s;
}

.category-item:hover {
  background: #f0f0f0;
}

.category-item.active {
  background: #e3f2fd;
  color: #1976d2;
  border-right: 3px solid #1976d2;
}

/* 内容区域 */
.content-area {
  flex: 1;
  background: #fff;
  overflow-y: auto;
  padding: 16px;
}

/* 空状态 */
.empty-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: 400px;
  color: #999;
}

.empty-icon {
  font-size: 64px;
  color: #ddd;
  margin-bottom: 16px;
}

.empty-text {
  font-size: 18px;
  font-weight: 500;
  margin-bottom: 8px;
}

.empty-desc {
  font-size: 14px;
  margin-bottom: 24px;
}

/* 程序列表 */
.program-list {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.program-item {
  display: flex;
  align-items: center;
  gap: 16px;
  padding: 16px;
  background: #fff;
  border: 1px solid #e0e0e0;
  border-radius: 8px;
  transition: all 0.2s;
}

.program-item:hover {
  border-color: #1976d2;
  box-shadow: 0 2px 8px rgba(25, 118, 210, 0.1);
}

.program-item.running {
  border-color: #4caf50;
  background: #f8fff8;
}

.program-icon {
  width: 40px;
  height: 40px;
  background: #f5f5f5;
  border-radius: 8px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 20px;
  color: #666;
}

.program-item.running .program-icon {
  background: #e8f5e8;
  color: #4caf50;
}

.program-details {
  flex: 1;
  min-width: 0;
}

.program-name {
  font-size: 16px;
  font-weight: 600;
  color: #333;
  margin-bottom: 4px;
}

.program-path {
  font-size: 12px;
  color: #666;
  margin-bottom: 2px;
  font-family: 'Courier New', monospace;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.program-command {
  font-size: 12px;
  color: #888;
  margin-bottom: 4px;
  font-family: 'Courier New', monospace;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.program-description {
  font-size: 13px;
  color: #555;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.program-status {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-right: 16px;
}

.status-indicator {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: #ccc;
  transition: background-color 0.2s;
}

.status-indicator.active {
  background: #4caf50;
  box-shadow: 0 0 4px rgba(76, 175, 80, 0.5);
}

.status-text {
  font-size: 12px;
  color: #666;
  min-width: 40px;
}

.program-actions {
  display: flex;
  gap: 8px;
}

/* 状态栏 */
.status-bar {
  height: 24px;
  background: #f0f0f0;
  border-top: 1px solid #d0d0d0;
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0 16px;
  font-size: 12px;
  color: #666;
}

.status-left {
  display: flex;
  gap: 16px;
}

.separator {
  color: #ccc;
}

/* 对话框样式 - 简洁软件风格 */
.program-dialog :deep(.el-dialog) {
  border-radius: 6px;
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.15);
  border: 1px solid #d0d0d0;
  overflow: hidden;
  background: #ffffff;
}

.program-dialog :deep(.el-dialog__header) {
  padding: 0;
  margin: 0;
}

.program-dialog :deep(.el-dialog__body) {
  padding: 0;
  overflow: hidden;
}

.program-dialog :deep(.el-dialog__footer) {
  padding: 0;
  margin: 0;
}

/* 对话框头部 - 简洁风格 */
.dialog-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 12px 16px;
  background: #f6f6f6;
  border-bottom: 1px solid #d0d0d0;
  min-height: 44px;
  user-select: none;
}

.dialog-title {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 14px;
  font-weight: 500;
  color: #333;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
}

.dialog-icon {
  width: 16px;
  height: 16px;
  color: #666;
}

.dialog-close-btn {
  width: 24px;
  height: 24px;
  min-height: 24px;
  padding: 0;
  color: #666;
  border: none;
  background: transparent;
  border-radius: 3px;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: background 0.15s;
}

.dialog-close-btn:hover {
  color: #333;
  background: rgba(0, 0, 0, 0.08);
}

.dialog-close-btn:active {
  background: rgba(0, 0, 0, 0.12);
}

/* 对话框内容 - 无滚动条 */
.dialog-content {
  padding: 20px;
  background: #fff;
  overflow: hidden;
}

.program-form {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.form-section {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.section-title {
  font-size: 13px;
  font-weight: 600;
  color: #555;
  padding-bottom: 6px;
  border-bottom: 1px solid #e8e8e8;
  margin-bottom: 6px;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
}

.form-item {
  margin-bottom: 0;
}

.form-item :deep(.el-form-item__label) {
  font-weight: 500;
  color: #555;
  line-height: 32px;
  font-size: 13px;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
}

.form-item :deep(.el-input__wrapper) {
  border-radius: 4px;
  border: 1px solid #d0d0d0;
  transition: border-color 0.15s;
  box-shadow: none;
}

.form-item :deep(.el-input__wrapper:hover) {
  border-color: #b0b0b0;
}

.form-item :deep(.el-input__wrapper.is-focus) {
  border-color: #007acc;
  box-shadow: 0 0 0 2px rgba(0, 122, 204, 0.2);
}

.form-item :deep(.el-input__inner) {
  font-size: 13px;
  color: #333;
}

.form-item :deep(.el-textarea__inner) {
  border-radius: 4px;
  border: 1px solid #d0d0d0;
  font-family: 'Consolas', 'Monaco', 'Courier New', monospace;
  font-size: 12px;
  transition: border-color 0.15s;
  box-shadow: none;
  color: #333;
}

.form-item :deep(.el-textarea__inner:hover) {
  border-color: #b0b0b0;
}

.form-item :deep(.el-textarea__inner:focus) {
  border-color: #007acc;
  box-shadow: 0 0 0 2px rgba(0, 122, 204, 0.2);
}

.form-item :deep(.el-input-group__append) {
  background: #f8f8f8;
  border: 1px solid #d0d0d0;
  border-left: none;
  border-radius: 0 4px 4px 0;
}

.form-item :deep(.el-input-group__append .el-button) {
  background: transparent;
  border: none;
  color: #007acc;
  font-weight: 500;
  padding: 8px 12px;
  font-size: 12px;
}

.form-item :deep(.el-input-group__append .el-button:hover) {
  background: rgba(0, 122, 204, 0.1);
}

.form-tip {
  display: flex;
  align-items: center;
  gap: 4px;
  margin-top: 4px;
  font-size: 11px;
  color: #888;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
}

.form-tip .el-icon {
  font-size: 12px;
  color: #007acc;
}

/* 对话框底部 - 简洁风格 */
.dialog-footer {
  display: flex;
  gap: 8px;
  justify-content: flex-end;
  padding: 12px 16px;
  background: #f6f6f6;
  border-top: 1px solid #d0d0d0;
  min-height: 52px;
  align-items: center;
}

.dialog-footer .el-button {
  display: flex;
  align-items: center;
  gap: 4px;
  padding: 6px 16px;
  border-radius: 4px;
  font-weight: 400;
  font-size: 13px;
  min-height: 28px;
  transition: all 0.15s;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
}

.dialog-footer .el-button:not(.el-button--primary) {
  background: #fff;
  border: 1px solid #d0d0d0;
  color: #333;
}

.dialog-footer .el-button:not(.el-button--primary):hover {
  background: #f8f8f8;
  border-color: #b0b0b0;
}

.dialog-footer .el-button:not(.el-button--primary):active {
  background: #f0f0f0;
  border-color: #a0a0a0;
}

.dialog-footer .el-button--primary {
  background: #007acc;
  border: 1px solid #007acc;
  color: #fff;
}

.dialog-footer .el-button--primary:hover {
  background: #005a9e;
  border-color: #005a9e;
}

.dialog-footer .el-button--primary:active {
  background: #004578;
  border-color: #004578;
}

.dialog-footer .el-button .el-icon {
  font-size: 12px;
}

/* 设置对话框 */
.settings-content {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.setting-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 8px 0;
}

/* 关于对话框 */
.about-content {
  text-align: center;
  padding: 16px;
}

.about-icon {
  margin-bottom: 16px;
  color: #1976d2;
}

.about-content h3 {
  margin: 0 0 8px 0;
  color: #333;
}

.about-content p {
  margin: 4px 0;
  color: #666;
  font-size: 14px;
}

/* 响应式设计 */
@media (max-width: 768px) {
  .sidebar {
    width: 160px;
  }
  
  .program-item {
    flex-direction: column;
    align-items: flex-start;
    gap: 12px;
  }
  
  .program-actions {
    align-self: stretch;
    justify-content: flex-end;
  }
}

/* 滚动条样式 */
::-webkit-scrollbar {
  width: 8px;
}

::-webkit-scrollbar-track {
  background: #f1f1f1;
}

::-webkit-scrollbar-thumb {
  background: #c1c1c1;
  border-radius: 4px;
}

::-webkit-scrollbar-thumb:hover {
  background: #a8a8a8;
}
</style>
