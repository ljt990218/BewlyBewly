<script setup lang="ts">
import { onKeyStroke } from '@vueuse/core'

import { isHomePage } from '~/utils/main'

// TODO: support shortcuts like `Ctrl+Alt+T` to open in new tab, `Esc` to close

const props = defineProps<{
  url: string
  title?: string
}>()

const emit = defineEmits<{
  (e: 'close'): void
}>()

const show = ref(false)
const iframeRef = ref<HTMLIFrameElement | null>(null)
const currentUrl = ref<string>(props.url)
const delayCloseTimer = ref<NodeJS.Timeout | null>(null)

onMounted(async () => {
  history.pushState({}, '', props.url)
  show.value = true
  await nextTick()
  iframeRef.value?.contentWindow?.addEventListener('pushstate', updateCurrentUrl)
  window.addEventListener('popstate', updateIframeUrl)
})

onUnmounted(() => {
  history.pushState({}, '', 'https://www.bilibili.com')
  iframeRef.value?.contentWindow?.removeEventListener('pushstate', updateCurrentUrl)
  window.removeEventListener('popstate', updateIframeUrl)
})

function updateCurrentUrl() {
  if (iframeRef.value?.contentWindow) {
    try {
      currentUrl.value = iframeRef.value.contentWindow.location.href
      history.pushState({}, '', currentUrl.value)
    }
    catch (error) {
      console.error('Unable to access iframe URL:', error)
    }
  }
}

function updateIframeUrl() {
  if (isHomePage()) {
    handleClose()
    return
  }

  if (iframeRef.value?.contentWindow) {
    iframeRef.value.contentWindow.location.replace(location.href)
    // iframeRef.value.contentWindow.location.reload()
  }
}

function handleClose() {
  if (delayCloseTimer.value) {
    clearTimeout(delayCloseTimer.value)
  }
  show.value = false
  delayCloseTimer.value = setTimeout(() => {
    emit('close')
  }, 300)
}

function handleOpenInNewTab() {
  window.open(props.url, '_blank')
}

onKeyStroke('Escape', (e: KeyboardEvent) => {
  e.preventDefault()
  handleClose()
})

// const keys = useMagicKeys()
// const ctrlAltT = keys['Ctrl+Alt+T']

// watch(() => ctrlAltT, (value) => {
//   if (value) {
//     console.log('ctrlAltT', value)
//     handleOpenInNewTab()
//   }
// })
</script>

<template>
  <div
    pos="absolute top-0 left-0" of-hidden w-full h-full
    z-999999
  >
    <!-- Mask -->
    <Transition name="fade">
      <div
        v-if="show"
        pos="absolute bottom-0 left-0" w-full h-full bg="black opacity-60"
        @click="handleClose"
      />
    </Transition>

    <Transition name="fade">
      <div
        v-if="show"
        pos="relative top-0" flex="~ items-center justify-end gap-2"
        max-w="$bew-page-max-width" w-full h="$bew-top-bar-height"
        m-auto px-4
        pointer-events-none
      >
        <Button
          style="
            --b-button-color: var(--bew-elevated-solid);
            --b-button-color-hover: var(--bew-elevated-solid-hover);
          "
          pointer-events-auto
          @click="handleOpenInNewTab"
        >
          <template #left>
            <i i-mingcute:external-link-line />
          </template>
          {{ $t('iframe_drawer.open_in_new_tab') }}
          <!-- <div flex="~">
            <kbd>Ctrl</kbd><kbd>Alt</kbd><kbd>T</kbd>
          </div> -->
        </Button>
        <Button
          style="
            --b-button-color: var(--bew-elevated-solid);
            --b-button-color-hover: var(--bew-elevated-solid-hover);
          "
          pointer-events-auto
          @click="handleClose"
        >
          <template #left>
            <i i-mingcute:close-line />
          </template>
          {{ $t('iframe_drawer.close') }}
          <!-- <kbd>Esc</kbd> -->
        </Button>
      </div>
    </Transition>

    <!-- Iframe -->
    <Transition name="drawer">
      <div
        v-if="show"
        pos="absolute top-$bew-top-bar-height left-0" of-hidden bg="$bew-bg"
        rounded="t-$bew-radius" w-full h-full
      >
        <iframe
          ref="iframeRef"
          :src="url"
          frameborder="0"
          pointer-events-auto
          pos="absolute bottom-$bew-top-bar-height left-0"
          w-full h-full
        />
      </div>
    </Transition>
  </div>
</template>

<style lang="scss" scoped>
.drawer-enter-active,
.drawer-leave-active {
  transition: transform 0.3s;
}

.drawer-enter-from,
.drawer-leave-to {
  transform: translateY(100%);
}
</style>
