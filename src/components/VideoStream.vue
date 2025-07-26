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
    <div class="mb-4 flex gap-2">
      <button
        class="btn"
        :class="isRunning ? 'bg-red-600 hover:bg-red-700' : 'bg-green-600 hover:bg-green-700'"
        @click="toggleStream"
      >
        {{ isRunning ? '停止视频' : '开始视频' }}
      </button>
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
