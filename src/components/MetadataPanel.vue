<script setup lang="ts">
defineOptions({
  name: 'MetadataPanel',
})

const props = defineProps<Props>()

const emit = defineEmits<{
  error: [message: string]
  dataReceived: [data: Metadata]
  statusChanged: [isRunning: boolean]
}>()

interface TrackedTarget {
  bbox: number[] | null
  target_id: number | null
  require_badge: boolean
}

interface ExtraMetadata {
  eval_times: number[]
  camera_id: number
  all_boxes: number[][][]
  tracked_target: TrackedTarget | null
}

//
interface Metadata {
  frame_count: number
  fps: number
  timestamp: number
  detected_objects: number
  detected_classes: Record<string, number>
  processing_time: number
  extra_metadata: ExtraMetadata
  state_count: number
  config_aggressive: boolean
}

interface Props {
  streamUrl: string
}

const metadata = ref<Metadata | null>(null)
const isRunning = ref(false)
let metadataSource: EventSource | null = null

function startStream() {
  if (metadataSource) {
    metadataSource.close()
  }

  metadataSource = new EventSource(props.streamUrl)
  isRunning.value = true
  emit('statusChanged', true)

  metadataSource.onmessage = function (event) {
    try {
      const data = JSON.parse(event.data)
      metadata.value = data
      emit('dataReceived', data)
    }
    catch (error) {
      console.error('Failed to parse metadata:', error)
      emit('error', 'Failed to parse metadata')
    }
  }

  metadataSource.onerror = function (event) {
    console.error('Metadata EventSource error:', event)
    emit('error', 'Metadata connection error')
  }
}

function stopStream() {
  if (metadataSource) {
    metadataSource.close()
    metadataSource = null
  }

  isRunning.value = false
  emit('statusChanged', false)
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
  <div class="max-h-80vh w-100 flex-none overflow-y-auto rounded-2 bg-white p-5 shadow-lg">
    <div class="mb-4 flex items-center justify-between">
      <h2 class="m-0 border-b-2 border-indigo-500 pb-2.5 text-xl text-gray-800">
        实时元信息
      </h2>
      <button
        class="text-sm btn"
        :class="isRunning ? 'bg-red-600 hover:bg-red-700' : 'bg-green-600 hover:bg-green-700'"
        @click="toggleStream"
      >
        {{ isRunning ? '停止' : '开始' }}
      </button>
    </div>
    <div class="text-sm">
      <div v-if="metadata && isRunning" class="flex flex-col gap-5">
        <!-- 基础信息 -->
        <div class="border border-gray-300 rounded-1.5 bg-gray-50 p-3.75">
          <h3 class="m-0 mb-3 text-base text-gray-600 font-semibold">
            基础信息
          </h3>
          <div class="mb-2 flex items-start justify-between py-1">
            <span class="min-w-20 text-gray-600 font-medium">帧数:</span>
            <span class="text-right text-gray-800 font-semibold">{{ metadata.frame_count }}</span>
          </div>
          <div class="mb-2 flex items-start justify-between py-1">
            <span class="min-w-20 text-gray-600 font-medium">FPS:</span>
            <span class="text-right text-gray-800 font-semibold">{{ metadata.fps?.toFixed(2) }}</span>
          </div>
          <div class="mb-2 flex items-start justify-between py-1">
            <span class="min-w-20 text-gray-600 font-medium">时间戳:</span>
            <span class="text-right text-gray-800 font-semibold">{{ new Date(metadata.timestamp * 1000).toLocaleTimeString() }}</span>
          </div>
          <div class="flex items-start justify-between py-1">
            <span class="min-w-20 text-gray-600 font-medium">处理时间:</span>
            <span class="text-right text-gray-800 font-semibold">{{ (metadata.processing_time * 1000).toFixed(2) }}ms</span>
          </div>
        </div>

        <!-- 配置信息 -->
        <div class="border border-gray-300 rounded-1.5 bg-gray-50 p-3.75">
          <h3 class="m-0 mb-3 text-base text-gray-600 font-semibold">
            配置信息
          </h3>
          <div class="mb-2 flex items-start justify-between py-1">
            <span class="min-w-20 text-gray-600 font-medium">状态计数:</span>
            <span class="text-right text-gray-800 font-semibold">{{ metadata.state_count }}</span>
          </div>
          <div class="mb-2 flex items-start justify-between py-1">
            <span class="min-w-20 text-gray-600 font-medium">激进模式:</span>
            <span class="text-right text-gray-800 font-semibold">{{ metadata.config_aggressive }}</span>
          </div>
        </div>

        <!-- 检测信息 -->
        <div class="border border-gray-300 rounded-1.5 bg-gray-50 p-3.75">
          <h3 class="m-0 mb-3 text-base text-gray-600 font-semibold">
            目标检测
          </h3>
          <div class="mb-2 flex items-start justify-between py-1">
            <span class="min-w-20 text-gray-600 font-medium">检测对象数:</span>
            <span class="text-right text-gray-800 font-semibold">{{ metadata.detected_objects }}</span>
          </div>
          <div v-if="metadata.detected_classes && Object.keys(metadata.detected_classes).length > 0" class="flex items-start justify-between py-1">
            <span class="min-w-20 text-gray-600 font-medium">检测类别:</span>
            <div class="flex flex-col gap-1">
              <div v-for="(count, className) in metadata.detected_classes" :key="className" class="rounded-1 bg-blue-100 px-2 py-1 text-xs text-blue-700">
                {{ className }}: {{ count }}
              </div>
            </div>
          </div>
        </div>

        <!-- 跟踪目标信息 -->
        <div v-if="metadata.extra_metadata?.tracked_target" class="border border-gray-300 rounded-1.5 bg-gray-50 p-3.75">
          <h3 class="m-0 mb-3 text-base text-gray-600 font-semibold">
            跟踪目标
          </h3>
          <div class="mb-2 flex items-start justify-between py-1">
            <span class="min-w-20 text-gray-600 font-medium">目标ID:</span>
            <span class="text-right text-gray-800 font-semibold">{{ metadata.extra_metadata.tracked_target.target_id || 'N/A' }}</span>
          </div>
          <div v-if="metadata.extra_metadata.tracked_target.bbox" class="mb-2 flex items-start justify-between py-1">
            <span class="min-w-20 text-gray-600 font-medium">边界框:</span>
            <div class="grid grid-cols-2 gap-1 text-xs text-gray-800">
              <div>X: {{ metadata.extra_metadata.tracked_target.bbox[0]?.toFixed(1) }}</div>
              <div>Y: {{ metadata.extra_metadata.tracked_target.bbox[1]?.toFixed(1) }}</div>
              <div>W: {{ (metadata.extra_metadata.tracked_target.bbox[2] - metadata.extra_metadata.tracked_target.bbox[0])?.toFixed(1) }}</div>
              <div>H: {{ (metadata.extra_metadata.tracked_target.bbox[3] - metadata.extra_metadata.tracked_target.bbox[1])?.toFixed(1) }}</div>
            </div>
          </div>
          <div class="flex items-start justify-between py-1">
            <span class="min-w-20 text-gray-600 font-medium">需要标识:</span>
            <span class="text-right text-gray-800 font-semibold">{{ metadata.extra_metadata.tracked_target.require_badge ? '是' : '否' }}</span>
          </div>
        </div>

        <!-- 原始数据 -->
        <div class="border border-gray-300 rounded-1.5 bg-gray-50 p-3.75">
          <h3 class="m-0 mb-3 text-base text-gray-600 font-semibold">
            原始数据
          </h3>
          <pre class="max-h-50 overflow-y-auto whitespace-pre-wrap border border-gray-400 rounded-1 bg-gray-100 p-3 text-xs text-gray-600 leading-5.6">{{ JSON.stringify(metadata, null, 2) }}</pre>
        </div>
      </div>
      <div v-else class="px-5 py-10 text-center text-gray-500 italic">
        {{ isRunning ? '等待元信息数据...' : '元信息流已停止' }}
      </div>
    </div>
  </div>
</template>

<style scoped>
/* 自定义的行高 */
.leading-5\.6 {
  line-height: 1.4;
}
</style>
