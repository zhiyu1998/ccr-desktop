<script setup>
import { ref, onMounted, onUnmounted } from 'vue'
import { ElMessage, ElMessageBox } from 'element-plus'
import ConfigTab from './components/ConfigTab.vue'
import ServiceTab from './components/ServiceTab.vue'
import ClaudeConfigTab from './components/ClaudeConfigTab.vue'
import MultiModelConfig from './components/MultiModelConfig.vue'

// 统一消息管理函数 - 确保同时只显示一条消息
const showMessage = (message, type = 'info') => {
  ElMessage.closeAll() // 关闭所有现有消息
  return ElMessage[type](message)
}

// 响应式数据
const activeTab = ref('config')
const configData = ref(null)
const isLoading = ref(false)
const configPaths = ref(null)
const selectedProvider = ref(null)
const showProviderDialog = ref(false)
const showAddProviderDialog = ref(false)
const useProxy = ref(false)

// 服务状态全局管理
const isServiceRunning = ref(false)
const isCheckingServiceStatus = ref(false)
let servicePollingInterval = null
const newProvider = ref({
  name: '',
  api_base_url: '',
  api_key: '',
  models: [],
  transformer: null
})

// 临时模型输入
const newModelInput = ref('')
const newTransformerText = ref('')
const selectedProviderTransformerText = ref('')

// 自动加载配置
onMounted(() => {
  // 监听Claude配置保存事件
  window.addEventListener('claude-config-saved', handleClaudeConfigSaved)

  // {{ AURA-X: Add - 监听托盘菜单的网络模式变更事件. Approval: 寸止确认. }}
  // 监听来自托盘菜单的网络模式变更
  window.api.onNetworkModeChanged && window.api.onNetworkModeChanged(handleNetworkModeChanged)

  loadConfig()
  loadConfigPaths()
  detectNetworkMode() // 自动检测网络模式

  // {{ AURA-X: Modify - 应用启动时触发悬浮窗刷新. Approval: 寸止确认. }}
  // 通过主进程统一更新悬浮窗状态
  window.api.refreshFloatingWindow()

  // 启动服务状态轮询
  startServicePolling()
})

// {{ AURA-X: Modify - 简化为直接触发主进程刷新，避免重复逻辑. Approval: 寸止确认. }}
const updateFloatingWindowWithCurrentInfo = async () => {
  // 直接触发主进程刷新，由主进程统一处理模型信息获取和服务状态检测
  window.api.refreshFloatingWindow()
}

// 服务状态轮询管理
const startServicePolling = () => {
  // 立即检查一次
  checkGlobalServiceStatus()

  // 每30秒轮询一次
  servicePollingInterval = setInterval(() => {
    checkGlobalServiceStatus()
  }, 30000)
}

const stopServicePolling = () => {
  if (servicePollingInterval) {
    clearInterval(servicePollingInterval)
    servicePollingInterval = null
  }
}

// 全局服务状态检查
const checkGlobalServiceStatus = async () => {
  if (isCheckingServiceStatus.value) return

  isCheckingServiceStatus.value = true
  try {
    const result = await window.api.execCommand('ccr status')
    if (result.success && result.stdout) {
      const output = result.stdout.toLowerCase()
      const isRunning = output.includes('running') && !output.includes('not running')
      isServiceRunning.value = isRunning
    } else {
      isServiceRunning.value = false
    }
  } catch (error) {
    console.error('[App] 服务状态检查异常:', error)
    isServiceRunning.value = false
  } finally {
    isCheckingServiceStatus.value = false
  }
}

// 组件卸载时清理监听器
onUnmounted(() => {
  // 清理Claude配置保存事件监听器
  window.removeEventListener('claude-config-saved', handleClaudeConfigSaved)

  // {{ AURA-X: Add - 清理网络模式变更事件监听器. Approval: 寸止确认. }}
  // 清理网络模式变更事件监听器
  window.api.removeNetworkModeChangedListener &&
    window.api.removeNetworkModeChangedListener(handleNetworkModeChanged)

  // 清理服务状态轮询
  stopServicePolling()
})

// 处理Claude配置保存事件
const handleClaudeConfigSaved = () => {
  console.log('[App] 收到Claude配置保存事件，重新检测网络模式')
  detectNetworkMode()
}

// {{ AURA-X: Add - 处理托盘菜单的网络模式变更事件. Approval: 寸止确认. }}
// 处理网络模式变更事件
const handleNetworkModeChanged = ({ isProxy }) => {
  console.log('[App] 收到网络模式变更事件:', isProxy ? '代理模式' : '直连模式')
  useProxy.value = isProxy
  // 重新检测网络模式以确保状态一致
  detectNetworkMode()
}

// 加载配置文件
const loadConfig = async () => {
  isLoading.value = true

  try {
    const result = await window.api.readSettings()

    if (result.success) {
      configData.value = result.data
      showMessage('配置加载成功', 'success')

      // {{ AURA-X: Modify - 配置加载后更新悬浮窗. Approval: 寸止确认. }}
      // 更新悬浮窗显示的模型信息和服务状态
      updateFloatingWindowWithCurrentInfo()
    } else {
      configData.value = null
      showMessage('配置文件加载失败', 'error')
      console.warn('配置加载失败:', result.error)
    }
  } catch (error) {
    configData.value = null
    showMessage(`配置加载异常: ${error.message}`, 'error')
    console.error('配置加载异常:', error)
  } finally {
    isLoading.value = false
  }
}

// 加载配置路径信息
const loadConfigPaths = async () => {
  try {
    const result = await window.api.getConfigPaths()
    if (result.success) {
      configPaths.value = result.data
    }
  } catch (error) {
    console.error('获取配置路径失败:', error)
  }
}

// 打开CCR配置文件夹
const openConfigFolder = async () => {
  try {
    const result = await window.api.openConfigFolder()
    if (result.success) {
      showMessage('CCR配置文件夹已打开', 'success')
    } else {
      showMessage(`打开CCR配置文件夹失败: ${result.error}`, 'error')
    }
  } catch (error) {
    showMessage(`打开CCR配置文件夹异常: ${error.message}`, 'error')
    console.error('打开CCR配置文件夹异常:', error)
  }
}

// 打开Claude配置文件夹
const openClaudeConfigFolder = async () => {
  try {
    const result = await window.api.openClaudeConfigFolder()
    if (result.success) {
      showMessage('Claude配置文件夹已打开', 'success')
    } else {
      showMessage(`打开Claude配置文件夹失败: ${result.error}`, 'error')
    }
  } catch (error) {
    showMessage(`打开Claude配置文件夹异常: ${error.message}`, 'error')
    console.error('打开Claude配置文件夹异常:', error)
  }
}

// 检测网络模式
const detectNetworkMode = async () => {
  try {
    const result = await window.api.detectNetworkMode()
    if (result.success) {
      useProxy.value = result.isProxy
      console.log('[NetworkMode] 检测到网络模式:', result.mode, result.message)
    } else {
      console.warn('[NetworkMode] 检测失败:', result.error)
      useProxy.value = false
    }
  } catch (error) {
    console.error('[NetworkMode] 检测异常:', error)
    useProxy.value = false
  }
}

// 刷新配置
const refreshConfig = () => {
  showMessage('正在刷新配置...', 'info')
  loadConfig()
}

// 显示Provider详情
const showProviderDetail = (provider) => {
  selectedProvider.value = { ...provider }
  // 初始化转换器文本
  selectedProviderTransformerText.value = provider.transformer
    ? JSON.stringify(provider.transformer, null, 2)
    : ''
  showProviderDialog.value = true
}

// 显示添加Provider弹窗
const showAddProvider = () => {
  newProvider.value = {
    name: '',
    api_base_url: '',
    api_key: '',
    models: [],
    transformer: null
  }
  newModelInput.value = ''
  newTransformerText.value = ''
  showAddProviderDialog.value = true
}

// 关闭弹窗
const closeProviderDialog = () => {
  showProviderDialog.value = false
  selectedProvider.value = null
  selectedProviderTransformerText.value = ''
}

// 关闭添加Provider弹窗
const closeAddProviderDialog = () => {
  showAddProviderDialog.value = false
  newProvider.value = {
    name: '',
    api_base_url: '',
    api_key: '',
    models: [],
    transformer: null
  }
  newModelInput.value = ''
  newTransformerText.value = ''
}

// 添加模型到现有Provider
const addModelToProvider = () => {
  if (newModelInput.value.trim()) {
    if (!selectedProvider.value.models) {
      selectedProvider.value.models = []
    }
    selectedProvider.value.models.push(newModelInput.value.trim())
    newModelInput.value = ''
  }
}

// 从Provider删除模型
const removeModelFromProvider = (model) => {
  const index = selectedProvider.value.models.indexOf(model)
  if (index > -1) {
    selectedProvider.value.models.splice(index, 1)
  }
}

// 添加模型到新Provider
const addModelToNewProvider = () => {
  if (newModelInput.value.trim()) {
    newProvider.value.models.push(newModelInput.value.trim())
    newModelInput.value = ''
  }
}

// 从新Provider删除模型
const removeModelFromNewProvider = (model) => {
  const index = newProvider.value.models.indexOf(model)
  if (index > -1) {
    newProvider.value.models.splice(index, 1)
  }
}

// 删除Provider
const deleteProvider = async () => {
  if (!selectedProvider.value) return

  try {
    await ElMessageBox.confirm(
      `确定要删除服务商 "${selectedProvider.value.name}" 吗？此操作无法撤销。`,
      '删除确认',
      {
        confirmButtonText: '确定删除',
        cancelButtonText: '取消',
        type: 'warning',
        confirmButtonClass: 'el-button--danger'
      }
    )

    try {
      // 使用细粒度API删除Provider
      const providerName = selectedProvider.value.name
      const result = await window.api.deleteProvider(providerName)

      if (result.success) {
        // 从本地缓存中移除
        if (configData.value?.Providers) {
          const index = configData.value.Providers.findIndex((p) => p.name === providerName)
          if (index > -1) {
            configData.value.Providers.splice(index, 1)
          }

          // 如果删除的Provider是当前默认模型的Provider，清空默认路由
          if (configData.value.Router?.default?.startsWith(providerName + ',')) {
            configData.value.Router.default = ''
          }
        }

        showMessage(`服务商 "${providerName}" 已删除`, 'success')
        closeProviderDialog()
      } else {
        showMessage(`删除失败: ${result.error}`, 'error')
      }
    } catch (error) {
      showMessage(`删除异常: ${error.message}`, 'error')
      console.error('删除配置异常:', error)
    }
  } catch {
    // 用户取消删除
    showMessage('已取消删除', 'info')
  }
}

// 保存Provider配置
const saveProviderConfig = async () => {
  // 验证并解析transformer JSON
  if (selectedProviderTransformerText.value.trim()) {
    try {
      selectedProvider.value.transformer = JSON.parse(selectedProviderTransformerText.value)
    } catch {
      showMessage('Transformer JSON格式错误', 'error')
      return
    }
  } else {
    // 如果转换器配置为空，删除该字段而不是设置为null
    delete selectedProvider.value.transformer
  }

  try {
    // 使用细粒度API更新Provider
    const providerName = selectedProvider.value.name
    // 将响应式对象转换为普通对象，避免序列化错误
    const providerData = JSON.parse(JSON.stringify(selectedProvider.value))
    const result = await window.api.updateProvider(providerName, providerData)

    if (result.success) {
      // 更新本地缓存数据
      if (configData.value?.Providers) {
        const index = configData.value.Providers.findIndex((p) => p.name === providerName)
        if (index > -1) {
          configData.value.Providers[index] = { ...selectedProvider.value }
        }
      }

      showMessage('配置已保存', 'success')
      closeProviderDialog()
    } else {
      showMessage(`保存失败: ${result.error}`, 'error')
    }
  } catch (error) {
    showMessage(`保存异常: ${error.message}`, 'error')
    console.error('保存配置异常:', error)
  }
}

// 保存新Provider
const saveNewProvider = async () => {
  // 验证必填字段
  if (!newProvider.value.name.trim()) {
    showMessage('请输入服务商名称', 'error')
    return
  }
  if (!newProvider.value.api_base_url.trim()) {
    showMessage('请输入API地址', 'error')
    return
  }

  // 解析transformer JSON
  if (newTransformerText.value.trim()) {
    try {
      newProvider.value.transformer = JSON.parse(newTransformerText.value)
    } catch {
      showMessage('Transformer JSON格式错误', 'error')
      return
    }
  } else {
    // 如果转换器配置为空，删除该字段而不是设置为null
    delete newProvider.value.transformer
  }

  try {
    // 使用细粒度API添加Provider
    // 将响应式对象转换为普通对象，避免序列化错误
    const providerData = JSON.parse(JSON.stringify(newProvider.value))
    const result = await window.api.addProvider(providerData)

    if (result.success) {
      // 添加到本地缓存数据
      if (!configData.value) {
        configData.value = { LOG: false, Providers: [], Router: { default: '' } }
      }
      if (!configData.value.Providers) {
        configData.value.Providers = []
      }

      configData.value.Providers.push({ ...newProvider.value })

      showMessage(`服务商 "${newProvider.value.name}" 已添加`, 'success')
      closeAddProviderDialog()
    } else {
      showMessage(`添加失败: ${result.error}`, 'error')
    }
  } catch (error) {
    showMessage(`添加异常: ${error.message}`, 'error')
    console.error('添加配置异常:', error)
  }
}

// 保存默认模型选择
const saveDefaultModel = async (selectedModel) => {
  try {
    // 使用细粒度API更新默认模型
    const result = await window.api.updateDefaultModel(selectedModel)

    if (result.success) {
      // 更新本地缓存数据
      if (!configData.value.Router) {
        configData.value.Router = {}
      }
      configData.value.Router.default = selectedModel

      showMessage('默认模型已保存', 'success')

      // {{ AURA-X: Modify - 默认模型保存后更新悬浮窗. Approval: 寸止确认. }}
      // 更新悬浮窗显示的模型信息和服务状态
      updateFloatingWindowWithCurrentInfo()
    } else {
      showMessage(`保存失败: ${result.error}`, 'error')
    }
  } catch (error) {
    showMessage(`保存异常: ${error.message}`, 'error')
    console.error('保存默认模型异常:', error)
  }
}

// 保存路由器模型配置
const saveRouterModel = async ({ type, value }) => {
  try {
    // 使用细粒度API更新路由器模型
    const result = await window.api.updateRouterModel(type, value)

    if (result.success) {
      // 更新本地缓存数据
      if (!configData.value.Router) {
        configData.value.Router = {}
      }
      configData.value.Router[type] = value

      showMessage(`${type}模型已保存`, 'success')

      // {{ AURA-X: Modify - 模型保存后更新悬浮窗. Approval: 寸止确认. }}
      // 更新悬浮窗显示的模型信息和服务状态
      updateFloatingWindowWithCurrentInfo()
    } else {
      showMessage(`保存失败: ${result.error}`, 'error')
    }
  } catch (error) {
    showMessage(`保存异常: ${error.message}`, 'error')
    console.error(`保存${type}模型异常:`, error)
  }
}

// 保存长文本阈值配置
const saveLongContextThreshold = async (threshold) => {
  try {
    // 使用细粒度API更新长文本阈值
    const result = await window.api.updateLongContextThreshold(threshold)

    if (result.success) {
      // 更新本地缓存数据
      if (!configData.value.Router) {
        configData.value.Router = {}
      }
      configData.value.Router.longContextThreshold = threshold

      showMessage('长文本阈值已保存', 'success')

      // {{ AURA-X: Modify - 阈值保存后更新悬浮窗. Approval: 寸止确认. }}
      // 更新悬浮窗显示的模型信息和服务状态
      updateFloatingWindowWithCurrentInfo()
    } else {
      showMessage(`保存失败: ${result.error}`, 'error')
    }
  } catch (error) {
    showMessage(`保存异常: ${error.message}`, 'error')
    console.error('保存长文本阈值异常:', error)
  }
}

// 保存所有路由器配置
const saveAllRouterConfig = async (routerConfig) => {
  try {
    // 使用整体保存API更新路由器配置
    // 创建完整的配置数据对象，确保使用原始数据而不是Vue ref
    // 深度克隆以确保所有数据都是可序列化的
    const fullConfigData = JSON.parse(
      JSON.stringify({
        ...configData.value,
        Router: routerConfig
      })
    )

    const result = await window.api.saveSettings(fullConfigData)

    if (result.success) {
      // 更新本地缓存数据
      configData.value.Router = { ...routerConfig }

      showMessage('所有路由器配置已保存', 'success')

      // {{ AURA-X: Modify - 配置保存后更新悬浮窗. Approval: 寸止确认. }}
      // 更新悬浮窗显示的模型信息和服务状态
      updateFloatingWindowWithCurrentInfo()
    } else {
      showMessage(`保存失败: ${result.error}`, 'error')
    }
  } catch (error) {
    showMessage(`保存异常: ${error.message}`, 'error')
    console.error('保存所有路由器配置异常:', error)
  }
}

// 处理服务消息
const handleServiceMessage = ({ text, type }) => {
  showMessage(text, type)
}

// 处理服务状态变化
const handleServiceStatusChange = (isRunning) => {
  console.log('[App] 收到服务状态变化:', isRunning)
  isServiceRunning.value = isRunning
  // 更新悬浮窗
  updateFloatingWindowWithCurrentInfo()
}

// {{ AURA-X: Modify - 完善全局代理切换逻辑，实现实际的网络模式切换. Approval: 寸止确认. }}
// 处理全局代理切换
const handleGlobalProxyChange = async (value) => {
  try {
    console.log('[App] 开始切换网络模式:', value ? '代理模式' : '直连模式')

    // 显示切换提示
    showMessage(`正在切换到${value ? '代理模式' : '直连模式'}...`, 'info')

    // 调用主进程进行网络模式切换
    const result = await window.api.switchNetworkMode(value)

    if (result.success) {
      // 切换成功，显示成功消息
      showMessage(`${result.message}`, 'success')

      // 重新检测网络模式以确保UI状态正确
      setTimeout(() => {
        detectNetworkMode()
      }, 500)

      // 更新悬浮窗状态
      updateFloatingWindowWithCurrentInfo()

      // 发送配置保存事件，触发相关组件更新
      window.dispatchEvent(new CustomEvent('claude-config-saved'))
    } else {
      // 切换失败，恢复开关状态
      useProxy.value = !value
      showMessage(`切换失败: ${result.error}`, 'error')
      console.error('[App] 网络模式切换失败:', result.error)
    }
  } catch (error) {
    // 异常处理，恢复开关状态
    useProxy.value = !value
    showMessage(`切换异常: ${error.message}`, 'error')
    console.error('[App] 网络模式切换异常:', error)
  }
}
</script>

<template>
  <div class="min-h-screen bg-gradient-to-br from-blue-50 to-purple-50 overflow-x-hidden">
    <!-- 头部区域 -->
    <div class="bg-white shadow-sm border-b border-gray-200 p-4 sm:p-6 sticky top-0 z-10">
      <div class="max-w-6xl mx-auto">
        <div class="flex flex-col sm:flex-row items-start sm:items-center justify-between gap-4">
          <div class="flex items-center space-x-4">
            <img alt="logo" class="w-10 h-10 sm:w-12 sm:h-12" src="./assets/icon.svg" />
            <div>
              <h1 class="text-xl sm:text-2xl font-bold text-gray-900">Claude Code Router</h1>
              <p class="text-sm text-gray-600">管理面板</p>
            </div>
          </div>

          <!-- 顶部操作按钮 -->
          <div class="flex items-center space-x-3 w-full sm:w-auto">
            <!-- 网络连接模式切换 -->
            <div class="flex items-center gap-2 px-3 py-1 bg-gray-50 rounded-lg">
              <span class="text-sm text-gray-600">网络模式:</span>
              <el-switch
                v-model="useProxy"
                active-text="代理"
                inactive-text="直连"
                size="small"
                @change="handleGlobalProxyChange"
              />
              <span class="text-xs text-gray-500" title="基于Claude配置自动检测"> 自动检测 </span>
            </div>

            <!-- 仅在启动服务tab时只显示网络模式，其他tab显示对应按钮 -->
            <template v-if="activeTab !== 'service'">
              <el-button
                v-if="activeTab === 'config'"
                type="success"
                @click="showAddProvider"
                class="flex-1 sm:flex-none"
              >
                添加服务商
              </el-button>
              <el-button
                v-if="activeTab === 'config'"
                type="primary"
                :loading="isLoading"
                @click="refreshConfig"
                class="flex-1 sm:flex-none"
              >
                {{ isLoading ? '加载中...' : '刷新配置' }}
              </el-button>
              <el-button
                v-if="activeTab === 'config'"
                type="info"
                @click="openConfigFolder"
                class="flex-1 sm:flex-none"
              >
                打开CCR配置文件夹
              </el-button>
              <el-button
                v-if="activeTab === 'claude'"
                type="info"
                @click="openClaudeConfigFolder"
                class="flex-1 sm:flex-none"
              >
                打开Claude配置文件夹
              </el-button>
            </template>
          </div>
        </div>

        <!-- Tab导航 -->
        <div class="mt-4">
          <el-tabs v-model="activeTab" class="ccr-tabs">
            <el-tab-pane label="配置管理" name="config"></el-tab-pane>
            <el-tab-pane label="Claude配置" name="claude"></el-tab-pane>
            <el-tab-pane label="启动服务" name="service"></el-tab-pane>
          </el-tabs>
        </div>
      </div>
    </div>

    <!-- 主内容区域 -->
    <div class="max-w-6xl mx-auto p-4 sm:p-6 pb-8">
      <!-- 配置管理Tab内容 -->
      <div v-if="activeTab === 'config'">
        <MultiModelConfig
          v-if="configData"
          :providers="configData.Providers"
          :router="configData.Router"
          @save-model="saveRouterModel"
          @save-threshold="saveLongContextThreshold"
          @save-all="saveAllRouterConfig"
        />
        <ConfigTab
          :config-data="configData"
          :is-loading="isLoading"
          @refresh="refreshConfig"
          @save-default-model="saveDefaultModel"
          @provider-click="showProviderDetail"
        />
      </div>

      <!-- Claude配置Tab内容 -->
      <div v-if="activeTab === 'claude'">
        <ClaudeConfigTab />
      </div>

      <!-- 启动服务Tab内容 -->
      <div v-if="activeTab === 'service'">
        <ServiceTab
          :active-tab="activeTab"
          :is-service-running="isServiceRunning"
          @message="handleServiceMessage"
          @service-status-changed="handleServiceStatusChange"
        />
      </div>
    </div>

    <!-- Provider 详情弹窗 -->
    <el-dialog
      v-model="showProviderDialog"
      :title="`${selectedProvider?.name || ''} 详细配置`"
      :width="'90vw'"
      :style="{ 'max-width': '600px' }"
      :lock-scroll="false"
      @closed="closeProviderDialog"
      append-to-body
      height="600px"
    >
      <div v-if="selectedProvider" class="space-y-4 h-[500px] flex flex-col">
        <!-- 基本信息 -->
        <div class="bg-gray-50 rounded-lg p-4 flex-shrink-0">
          <h4 class="font-medium text-gray-900 mb-3">基本信息</h4>
          <div class="grid grid-cols-1 sm:grid-cols-2 gap-4">
            <div>
              <label class="block text-sm font-medium text-gray-700 mb-1">服务名称</label>
              <el-input v-model="selectedProvider.name" placeholder="请输入服务名称" />
            </div>
            <div>
              <label class="block text-sm font-medium text-gray-700 mb-1">API地址</label>
              <el-input v-model="selectedProvider.api_base_url" placeholder="请输入API地址" />
            </div>
          </div>
        </div>

        <!-- API Key配置 -->
        <div class="flex-shrink-0">
          <label class="block text-sm font-medium text-gray-700 mb-2">API Key</label>
          <el-input
            v-model="selectedProvider.api_key"
            type="password"
            placeholder="请输入API Key..."
            show-password
          />
        </div>

        <!-- 模型列表 -->
        <div class="flex-1 min-h-0 flex flex-col">
          <label class="block text-sm font-medium text-gray-700 mb-2">支持的模型</label>
          <div class="bg-gray-50 rounded-lg p-3 flex-1 min-h-0 overflow-y-auto mb-3">
            <div v-if="selectedProvider.models?.length" class="space-y-1">
              <div
                v-for="model in selectedProvider.models"
                :key="model"
                class="flex items-center justify-between bg-white rounded px-3 py-2"
              >
                <span class="text-sm font-mono break-all pr-2">{{ model }}</span>
                <el-button
                  type="danger"
                  size="small"
                  text
                  @click="removeModelFromProvider(model)"
                  class="flex-shrink-0"
                >
                  删除
                </el-button>
              </div>
            </div>
            <div v-else class="text-center text-gray-500 text-sm py-4">暂无配置模型</div>
          </div>
          <div class="flex gap-2 flex-shrink-0">
            <el-input
              v-model="newModelInput"
              placeholder="输入新模型名称"
              @keyup.enter="addModelToProvider"
              size="small"
              class="flex-1"
            />
            <el-button
              type="primary"
              size="small"
              @click="addModelToProvider"
              class="flex-shrink-0"
            >
              添加
            </el-button>
          </div>
        </div>

        <!-- 转换器配置 -->
        <div class="flex-shrink-0">
          <label class="block text-sm font-medium text-gray-700 mb-2">转换器配置</label>
          <el-input
            v-model="selectedProviderTransformerText"
            type="textarea"
            :rows="3"
            placeholder="请输入转换器配置 (JSON)，留空则不使用转换器"
          />
        </div>
      </div>

      <template #footer>
        <div class="flex justify-between">
          <el-button type="danger" @click="deleteProvider" class="mr-auto">删除服务</el-button>
          <div class="flex space-x-3">
            <el-button @click="closeProviderDialog">取消</el-button>
            <el-button type="primary" @click="saveProviderConfig">保存配置</el-button>
          </div>
        </div>
      </template>
    </el-dialog>

    <!-- 添加服务商弹窗 -->
    <el-dialog
      v-model="showAddProviderDialog"
      title="添加新服务商"
      :width="'90vw'"
      :style="{ 'max-width': '600px' }"
      :lock-scroll="false"
      @closed="closeAddProviderDialog"
      append-to-body
      height="600px"
    >
      <div class="space-y-4 h-[500px] flex flex-col">
        <!-- 基本信息 -->
        <div class="bg-gray-50 rounded-lg p-4 flex-shrink-0">
          <h4 class="font-medium text-gray-900 mb-3">基本信息</h4>
          <div class="grid grid-cols-1 sm:grid-cols-2 gap-4 text-sm">
            <div>
              <label class="block text-sm font-medium text-gray-700 mb-1">服务商名称</label>
              <el-input v-model="newProvider.name" placeholder="请输入服务商名称" />
            </div>
            <div>
              <label class="block text-sm font-medium text-gray-700 mb-1">API地址</label>
              <el-input v-model="newProvider.api_base_url" placeholder="请输入API地址" />
            </div>
          </div>
        </div>

        <!-- API Key配置 -->
        <div class="flex-shrink-0">
          <label class="block text-sm font-medium text-gray-700 mb-2">API Key</label>
          <el-input
            v-model="newProvider.api_key"
            type="password"
            placeholder="请输入API Key..."
            show-password
          />
        </div>

        <!-- 模型列表 -->
        <div class="flex-1 min-h-0 flex flex-col">
          <label class="block text-sm font-medium text-gray-700 mb-2">支持的模型</label>
          <div class="bg-gray-50 rounded-lg p-3 flex-1 min-h-0 overflow-y-auto mb-3">
            <div v-if="newProvider.models?.length" class="space-y-1">
              <div
                v-for="model in newProvider.models"
                :key="model"
                class="flex items-center justify-between bg-white rounded px-3 py-2"
              >
                <span class="text-sm font-mono break-all pr-2">{{ model }}</span>
                <el-button
                  type="danger"
                  size="small"
                  text
                  @click="removeModelFromNewProvider(model)"
                  class="flex-shrink-0"
                >
                  删除
                </el-button>
              </div>
            </div>
            <div v-else class="text-center text-gray-500 text-sm py-4">暂无配置模型</div>
          </div>
          <div class="flex items-center gap-2 flex-shrink-0">
            <el-input
              v-model="newModelInput"
              placeholder="输入新模型名称 (按回车添加)"
              @keyup.enter="addModelToNewProvider"
              class="flex-1"
            />
            <el-button
              type="primary"
              size="small"
              @click="addModelToNewProvider"
              class="flex-shrink-0"
            >
              添加
            </el-button>
          </div>
        </div>

        <!-- 转换器配置 -->
        <div class="flex-shrink-0">
          <label class="block text-sm font-medium text-gray-700 mb-2">转换器配置 (JSON)</label>
          <el-input
            v-model="newTransformerText"
            type="textarea"
            :rows="3"
            placeholder="请输入转换器配置 (JSON)，留空则不使用转换器"
          />
        </div>
      </div>

      <template #footer>
        <div class="flex justify-end space-x-3">
          <el-button @click="closeAddProviderDialog">取消</el-button>
          <el-button type="primary" @click="saveNewProvider">保存配置</el-button>
        </div>
      </template>
    </el-dialog>
  </div>
</template>

<style scoped>
.logo {
  transition: filter 0.3s ease;
}

.logo:hover {
  filter: drop-shadow(0 0 2em #42b883aa);
}

/* Tab样式优化 */
:deep(.ccr-tabs .el-tabs__header) {
  margin: 0;
  border-bottom: 1px solid #e5e7eb;
}

:deep(.ccr-tabs .el-tabs__nav-wrap::after) {
  display: none;
}

:deep(.ccr-tabs .el-tabs__item) {
  color: #6b7280;
  font-weight: 500;
}

:deep(.ccr-tabs .el-tabs__item.is-active) {
  color: #3b82f6;
}

:deep(.ccr-tabs .el-tabs__active-bar) {
  background-color: #3b82f6;
}
</style>
