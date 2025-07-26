<script setup lang="ts">
defineOptions({
  name: 'VideoStream',
})

const props = withDefaults(defineProps<Props>(), {
  timeout: 5000,
})

const emit = defineEmits<{
  error: [message: string]
  connected: []
  disconnected: []
  statusChanged: [isRunning: boolean]
}>()

interface Props {
  streamUrl: string
  timeout?: number
}

const isVideoUnavailable = ref(false)
const hasLastFrame = ref(false)
const isRunning = ref(false)
const aggressiveMode = ref<'idle' | 'chasing' | 'explosion'>('idle')
const isResetting = ref(false)
const isTogglingMode = ref(false)
let timeoutId: NodeJS.Timeout | null = null
let frameSource: EventSource | null = null

function startStream() {
  if (frameSource) {
    frameSource.close()
  }

  frameSource = new EventSource(props.streamUrl)
  isRunning.value = true
  emit('statusChanged', true)

  frameSource.onmessage = function (event) {
    try {
      const data = JSON.parse(event.data)
      const frameElement = document.getElementById('frame') as HTMLImageElement
      if (frameElement) {
        frameElement.src = `data:image/jpeg;base64,${data.frame}`
        hasLastFrame.value = true
        isVideoUnavailable.value = false
      }

      // 重置超时计时器
      if (timeoutId) {
        clearTimeout(timeoutId)
      }

      // 设置新的超时计时器
      timeoutId = setTimeout(() => {
        isVideoUnavailable.value = true
        emit('disconnected')
      }, props.timeout)
    }
    catch (error) {
      console.error('Failed to parse frame data:', error)
      isVideoUnavailable.value = true
      emit('error', 'Failed to parse frame data')
    }
  }

  frameSource.onerror = function (event) {
    console.error('EventSource error:', event)
    isVideoUnavailable.value = true
    emit('error', 'EventSource connection error')
  }

  frameSource.onopen = function () {
    isVideoUnavailable.value = false
    emit('connected')
  }
}

function stopStream() {
  if (frameSource) {
    frameSource.close()
    frameSource = null
  }

  if (timeoutId) {
    clearTimeout(timeoutId)
    timeoutId = null
  }

  isRunning.value = false
  isVideoUnavailable.value = true
  emit('statusChanged', false)
  emit('disconnected')
}

function toggleStream() {
  if (isRunning.value) {
    stopStream()
  }
  else {
    startStream()
  }
}

async function resetState() {
  if (isResetting.value)
    return

  try {
    isResetting.value = true
    const response = await fetch('http://localhost:8000/increment', {
      method: 'POST',
    })

    if (!response.ok) {
      throw new Error(`Reset failed: ${response.status}`)
    }

    emit('statusChanged', isRunning.value)
  }
  catch (error) {
    console.error('Reset state error:', error)
    emit('error', `Reset state failed: ${error instanceof Error ? error.message : 'Unknown error'}`)
  }
  finally {
    isResetting.value = false
  }
}

async function toggleAggressiveMode() {
  if (isTogglingMode.value)
    return

  try {
    isTogglingMode.value = true

    // 循环切换状态：idle -> chasing -> explosion -> idle
    const modes: Array<'idle' | 'chasing' | 'explosion'> = ['idle', 'chasing', 'explosion']
    const currentIndex = modes.indexOf(aggressiveMode.value)
    const nextIndex = (currentIndex + 1) % modes.length
    const newMode = modes[nextIndex]

    const response = await fetch('http://localhost:8000/config', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        config_aggressive: newMode,
      }),
    })

    if (!response.ok) {
      throw new Error(`Config update failed: ${response.status}`)
    }

    aggressiveMode.value = newMode
    emit('statusChanged', isRunning.value)
  }
  catch (error) {
    console.error('Toggle aggressive mode error:', error)
    emit('error', `Toggle aggressive mode failed: ${error instanceof Error ? error.message : 'Unknown error'}`)
  }
  finally {
    isTogglingMode.value = false
  }
}

function getAggressiveModeButtonClass() {
  switch (aggressiveMode.value) {
    case 'idle':
      return 'bg-gray-600 hover:bg-gray-700'
    case 'chasing':
      return 'bg-orange-600 hover:bg-orange-700'
    case 'explosion':
      return 'bg-red-600 hover:bg-red-700'
    default:
      return 'bg-gray-600 hover:bg-gray-700'
  }
}

function getAggressiveModeLabel() {
  switch (aggressiveMode.value) {
    case 'idle':
      return '待机模式'
    case 'chasing':
      return '追踪模式'
    case 'explosion':
      return '爆破模式'
    default:
      return '未知模式'
  }
}

async function setAggressiveMode(mode: 'idle' | 'chasing' | 'explosion') {
  if (isTogglingMode.value || aggressiveMode.value === mode)
    return

  try {
    isTogglingMode.value = true

    const response = await fetch('http://localhost:8000/config', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        config_aggressive: mode,
      }),
    })

    if (!response.ok) {
      throw new Error(`Config update failed: ${response.status}`)
    }

    aggressiveMode.value = mode
    emit('statusChanged', isRunning.value)
  }
  catch (error) {
    console.error('Set aggressive mode error:', error)
    emit('error', `Set aggressive mode failed: ${error instanceof Error ? error.message : 'Unknown error'}`)
  }
  finally {
    isTogglingMode.value = false
  }
}

onMounted(() => {
  startStream()
})

onUnmounted(() => {
  stopStream()
})
</script>

<template>
  <div class="relative min-h-112.5 min-w-160 flex flex-1 flex-col items-center justify-center">
    <!-- 控制按钮 -->
    <div class="mb-4 flex flex-col gap-3">
      <!-- 第一行：主要控制按钮 -->
      <div class="flex gap-2">
        <button
          class="btn"
          :class="isRunning ? 'bg-red-600 hover:bg-red-700' : 'bg-green-600 hover:bg-green-700'"
          @click="toggleStream"
        >
          {{ isRunning ? '停止视频' : '开始视频' }}
        </button>

        <button
          class="bg-blue-600 btn hover:bg-blue-700"
          :disabled="isResetting"
          @click="resetState"
        >
          {{ isResetting ? '重置中...' : '重置状态' }}
        </button>

        <button
          class="btn"
          :class="getAggressiveModeButtonClass()"
          :disabled="isTogglingMode"
          @click="toggleAggressiveMode"
        >
          {{ isTogglingMode ? '切换中...' : getAggressiveModeLabel() }}
        </button>
      </div>

      <!-- 第二行：攻击模式选择器 -->
      <div class="flex items-center gap-2">
        <label for="aggressive-select" class="text-sm text-gray-700 font-medium">
          攻击模式:
        </label>
        <select
          id="aggressive-select"
          v-model="aggressiveMode"
          class="h-8 border-gray-300 rounded bg-white px-2 text-sm text-gray-700 focus:border-orange-500 focus:ring-orange-500"
          :disabled="isTogglingMode"
          @change="setAggressiveMode(aggressiveMode)"
        >
          <option value="idle">
            待机模式
          </option>
          <option value="chasing">
            追踪模式
          </option>
          <option value="explosion">
            爆破模式
          </option>
        </select>
        <span v-if="isTogglingMode" class="text-sm text-gray-500">(更新中...)</span>
      </div>
    </div>

    <!-- 视频显示区域 -->
    <div class="relative min-h-112.5 min-w-160 flex flex-1 items-center justify-center">
      <!-- 当没有上一帧时显示渐变色背景 -->
      <div
        v-if="!hasLastFrame"
        class="h-112.5 w-full flex items-center justify-center rounded-2 from-indigo-500 to-purple-600 bg-gradient-to-br shadow-lg"
      >
        <div v-if="isVideoUnavailable" class="border-2 border-white border-opacity-20 rounded-2 bg-black bg-opacity-50 px-8 py-4 text-2xl text-white font-bold text-shadow-lg">
          {{ isRunning ? '视频不可用' : '视频已停止' }}
        </div>
      </div>

      <!-- 视频帧 -->
      <img
        v-show="hasLastFrame"
        id="frame"
        alt="Video Stream"
        class="max-h-80vh max-w-full border-2 border-gray-800 rounded-2 shadow-lg transition-filter duration-300"
        :class="{ 'blur-2': isVideoUnavailable }"
      >

      <!-- 当有上一帧但视频不可用时显示的遮罩 -->
      <div
        v-if="hasLastFrame && isVideoUnavailable"
        class="absolute inset-0 flex items-center justify-center rounded-2 bg-black bg-opacity-30 backdrop-blur-sm"
      >
        <div class="border-2 border-white border-opacity-20 rounded-2 bg-black bg-opacity-50 px-8 py-4 text-2xl text-white font-bold text-shadow-lg">
          {{ isRunning ? '视频不可用' : '视频已停止' }}
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
/* 自定义的阴影效果，UnoCSS 默认不包含 */
.text-shadow-lg {
  text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.7);
}
</style>
