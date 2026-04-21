<template>
  <div class="flex flex-col items-center gap-2 max-w-full">
    <LoadingIndicator
      v-show="loading"
      class="w-10"
    />
    <video
      v-show="!loading"
      :key="src"
      ref="videoEl"
      class="max-h-[65vh] max-w-full rounded-lg"
      autoplay
      muted
      preload="none"
      controlslist="nodownload noremoteplayback noplaybackrate disablepictureinpicture"
      controls
      draggable="false"
      @loadedmetadata="handleMediaReady"
      @timeupdate="onTimeUpdate"
    >
      <source
        :src="src"
        :type="type"
      >
    </video>
    <div
      v-if="!loading && comments && comments.length > 0"
      class="relative w-full max-w-full h-3 bg-gray-200 dark:bg-gray-700 rounded-full cursor-pointer"
      @click="onMarkerBarClick"
    >
      <div
        v-for="comment in timestampComments"
        :key="comment.name"
        class="absolute top-1/2 -translate-y-1/2 w-2.5 h-2.5 rounded-full bg-blue-500 hover:bg-blue-600 hover:scale-150 transition-transform cursor-pointer group"
        :style="{ left: markerPosition(comment.timestamp) + '%' }"
        :title="formatTimestamp(comment.timestamp)"
        @click.stop="$emit('marker-click', comment)"
      >
        <div
          class="absolute bottom-full left-1/2 -translate-x-1/2 mb-2 hidden group-hover:block bg-gray-900 text-white text-xs rounded px-2 py-1 whitespace-nowrap pointer-events-none z-20"
        >
          {{ formatTimestamp(comment.timestamp) }}
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
/*
  Add codec evaluation currently assumes its a valid H264/5 (MP4/Webm)
  Look into the feasibility of client side mp4 moov fragmentation pre processing using
  https://github.com/gpac/gpac/wiki/MP4Box
  Server side byte is good enough for now
*/

import { LoadingIndicator } from "frappe-ui"
import { ref, computed, onBeforeUnmount, watch } from "vue"

const props = defineProps({
  previewEntity: Object,
  comments: { type: Array, default: () => [] },
})

const emit = defineEmits(["marker-click"])

const loading = ref(true)
const duration = ref(0)
const src = ref(
  `/api/method/drive.api.files.stream_file_content?entity_name=${props.previewEntity.name}`
)
const type = ref(
  props.previewEntity.mime_type === "video/quicktime"
    ? "video/mp4"
    : props.previewEntity.mime_type
)
const videoEl = ref(null)

const timestampComments = computed(() =>
  (props.comments || []).filter((c) => c.timestamp != null)
)

function markerPosition(timestamp) {
  if (!duration.value || duration.value === 0) return 0
  return Math.min((timestamp / duration.value) * 100, 100)
}

function formatTimestamp(seconds) {
  const s = Math.floor(seconds)
  const h = Math.floor(s / 3600)
  const m = Math.floor((s % 3600) / 60)
  const sec = s % 60
  if (h > 0)
    return `${h}:${String(m).padStart(2, "0")}:${String(sec).padStart(2, "0")}`
  return `${m}:${String(sec).padStart(2, "0")}`
}

const handleMediaReady = (event) => {
  const el = event.target
  if (el.readyState >= 1) {
    loading.value = false
    duration.value = el.duration || 0
  }
}

function onTimeUpdate() {
  if (videoEl.value) {
    duration.value = videoEl.value.duration || duration.value
  }
}

function onMarkerBarClick(e) {
  if (!duration.value || !videoEl.value) return
  const rect = e.currentTarget.getBoundingClientRect()
  const ratio = (e.clientX - rect.left) / rect.width
  videoEl.value.currentTime = ratio * duration.value
}

function getCurrentTime() {
  return videoEl.value?.currentTime ?? 0
}

function seekTo(seconds) {
  if (videoEl.value) {
    videoEl.value.currentTime = seconds
  }
}

function pause() {
  if (videoEl.value && !videoEl.value.paused) {
    videoEl.value.pause()
  }
}

watch(
  () => props.previewEntity,
  (newValue) => {
    loading.value = true
    duration.value = 0
    src.value = `/api/method/drive.api.files.stream_file_content?entity_name=${newValue.name}`
    type.value = newValue.mime_type
  }
)

onBeforeUnmount(() => {
  loading.value = true
  src.value = ""
  type.value = ""
})

defineExpose({ getCurrentTime, seekTo, pause })
</script>
