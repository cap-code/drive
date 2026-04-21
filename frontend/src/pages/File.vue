<template>
  <div class="flex w-full h-full">
    <div class="w-full h-full flex flex-col">
      <Navbar
        v-if="!file?.error"
        :root-resource="file"
      />
      <ErrorPage
        v-if="file.error"
        :error="file.error"
      />
      <div
        v-else
        class="flex-grow flex overflow-hidden"
      >
        <div
          id="renderContainer"
          :draggable="false"
          class="flex-grow px-10 py-5 flex justify-center align-center items-center relative"
        >
          <Button
            class="text-ink-gray-8 absolute top-4 left-4 z-10"
            :variant="'ghost'"
            icon="arrow-left"
            @click="closePreview"
          />
          <LoadingIndicator
            v-if="file.loading"
            class="w-10 h-full text-neutral-100"
          />
          <FileRender
            v-else-if="file.data"
            ref="fileRenderRef"
            :preview-entity="file.data"
            :comments="mediaComments"
            @marker-click="onMarkerClick"
          />
        </div>
        <MediaComments
          v-if="isMediaType"
          ref="mediaCommentsRef"
          v-model:show-comments="showComments"
          v-model:comments="mediaComments"
          :entity="file.data"
          :file-type="file.data?.file_type"
          @seek-to="onSeekTo"
          @request-timestamp="onRequestTimestamp"
        />
      </div>
      <div
        class="hidden sm:flex absolute bottom-4 left-1/2 transform -translate-x-1/2 w-fit items-center justify-center p-1 gap-1 rounded shadow-xl bg-surface-white z-10"
      >
        <Button
          :disabled="!prevEntity?.name"
          :variant="'ghost'"
          icon="arrow-left"
          @click="scrollEntity(true)"
        />
        <Button
          v-if="isMediaType"
          :variant="showComments ? 'subtle' : 'ghost'"
          class="relative"
          @click="showComments = !showComments"
        >
          <LucideMessageSquare class="size-4" />
          <span
            v-if="mediaComments.length > 0"
            class="absolute -top-1.5 -right-1.5 min-w-[18px] h-[18px] flex items-center justify-center rounded-full bg-surface-gray-7 text-ink-white text-[10px] font-semibold leading-none px-1"
          >{{ mediaComments.length > 9 ? '9+' : mediaComments.length }}</span>
        </Button>
        <Button
          :variant="'ghost'"
          @click="enterFullScreen"
        >
          <LucideScan class="size-4" />
        </Button>
        <Button
          :disabled="!nextEntity?.name"
          :variant="'ghost'"
          icon="arrow-right"
          @click="scrollEntity()"
        />
      </div>
    </div>
  </div>
</template>

<script setup>
import { useStore } from "vuex"
import Navbar from "@/components/Navbar.vue"
import { ref, computed, onMounted } from "vue"
import { Button, LoadingIndicator } from "frappe-ui"
import FileRender from "@/components/FileRender.vue"
import MediaComments from "@/components/MediaComments.vue"
import { createResource } from "frappe-ui"
import { useRouter } from "vue-router"
import LucideScan from "~icons/lucide/scan"
import LucideMessageSquare from "~icons/lucide/message-square"
import { onKeyStroke } from "@vueuse/core"
import {
  prettyData,
  setBreadCrumbs,
  enterFullScreen,
  updateURLSlug,
} from "@/utils/files"
import ErrorPage from "@/components/ErrorPage.vue"

const router = useRouter()
const store = useStore()
const props = defineProps({
  entityName: String,
  slug: String,
})

const currentEntity = ref(props.entityName)
const showComments = ref(false)
const mediaComments = ref([])
const fileRenderRef = ref(null)
const mediaCommentsRef = ref(null)

const isMediaType = computed(() => {
  const ft = file.data?.file_type
  return ft === "Video" || ft === "Image"
})

const filteredEntities = computed(() =>
  store.state.currentFolder.entities.filter(
    (item) => !item.is_group && !item.document && !item.is_link
  )
)

const index = computed(() => {
  return filteredEntities.value.findIndex(
    (item) => item.name === props.entityName
  )
})
const prevEntity = computed(() => filteredEntities.value[index.value - 1])
const nextEntity = computed(() => filteredEntities.value[index.value + 1])

function fetchFile(currentEntity) {
  file.fetch({ entity_name: currentEntity })
  router.push({
    params: {
      entityName: currentEntity,
    },
  })
}

onKeyStroke("ArrowLeft", (e) => {
  if (!e.shiftKey) return
  e.preventDefault()
  scrollEntity(true)
})
onKeyStroke("ArrowRight", (e) => {
  if (!e.shiftKey) return
  e.preventDefault()
  scrollEntity()
})

const onSuccess = async (entity) => {
  document.title = entity.title
  setBreadCrumbs(entity)
  updateURLSlug(entity.title)
  mediaComments.value = entity.comments || []
}

const file = createResource({
  url: "drive.api.permissions.get_entity_with_permissions",
  params: { entity_name: props.entityName },
  transform(entity) {
    store.commit("setActiveEntity", entity)
    return prettyData([entity])[0]
  },
  onSuccess,
})
store.commit("setCurrentResource", file)

function scrollEntity(negative = false) {
  currentEntity.value = negative ? prevEntity.value : nextEntity.value
  if (currentEntity.value) fetchFile(currentEntity.value.name)
}

function closePreview() {
  router.push({
    name: "Folder",
    params: { entityName: file.data.parent_entity },
  })
}

function onSeekTo(timestamp) {
  const preview = fileRenderRef.value?.previewRef
  if (preview?.seekTo) preview.seekTo(timestamp)
}

function onMarkerClick(comment) {
  showComments.value = true
  if (mediaCommentsRef.value) {
    mediaCommentsRef.value.activeComment = comment.name
  }
  const preview = fileRenderRef.value?.previewRef
  if (preview?.seekTo && comment.timestamp != null) {
    preview.seekTo(comment.timestamp)
  }
}

function onRequestTimestamp() {
  const preview = fileRenderRef.value?.previewRef
  if (preview?.pause) preview.pause()
  const timestamp = preview?.getCurrentTime?.() ?? 0
  if (mediaCommentsRef.value) {
    mediaCommentsRef.value.setComposeMeta({ timestamp })
  }
}

onMounted(() => {
  fetchFile(props.entityName)
})
</script>

<style scoped>
.center-transform {
  transform: translate(-50%, -50%);
}

#renderContainer::backdrop {
  background-color: rgb(0, 0, 0);
  min-width: 100vw;
  min-height: 100vh;
  position: fixed;
  width: 100%;
  height: 100%;
  left: 0;
  top: 0;
}
</style>
