<script setup lang="ts">
defineOptions({
  name: 'IndexPage',
})

useHead({
  title: 'Car Video Stream',
})

const baseUrl = 'http://192.168.66.120:8000'
const frameStreamUrl = `${baseUrl}/stream/frame`
const metadataStreamUrl = `${baseUrl}/stream/metadata`

// 处理视频流事件
function handleVideoError(message: string) {
  console.error('Video stream error:', message)
}

function handleVideoConnected() {
  // 视频流连接成功
}

function handleVideoDisconnected() {
  // 视频流断开连接
}

// 处理元数据事件
function handleMetadataError(message: string) {
  console.error('Metadata stream error:', message)
}

function handleMetadataReceived(data: any) {
  // 可以在这里处理接收到的元数据，比如统计、分析等
  // eslint-disable-next-line no-console
  console.log('Metadata received:', data)
}
</script>

<template>
  <div class="min-h-screen flex flex-col items-center justify-start bg-gray-100 p-5">
    <h1 class="mb-5 text-4xl text-gray-800">
      Car Video Stream
    </h1>
    <div class="max-w-7xl w-full flex items-center gap-5 max-sm:flex-col">
      <!-- 视频区域 -->
      <VideoStream
        :stream-url="frameStreamUrl"
        @error="handleVideoError"
        @connected="handleVideoConnected"
        @disconnected="handleVideoDisconnected"
      />

      <!-- 元信息区域 -->
      <MetadataPanel
        :stream-url="metadataStreamUrl"
        @error="handleMetadataError"
        @data-received="handleMetadataReceived"
      />
    </div>
  </div>
</template>

<style scoped>
/* 主页面不需要额外的样式，所有样式都在子组件中 */
</style>
