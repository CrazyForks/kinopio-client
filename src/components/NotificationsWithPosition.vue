<script setup>
import { reactive, computed, onMounted, onBeforeUnmount, watch, ref, nextTick } from 'vue'

import { useGlobalStore } from '@/stores/useGlobalStore'

import utils from '@/utils.js'

const globalStore = useGlobalStore()

const props = defineProps({
  layer: String // app, space
})

const items = computed(() => {
  let itemsInLayer = globalStore.notificationsWithPosition.filter(item => item.layer === props.layer)
  itemsInLayer = utils.clone(itemsInLayer)
  itemsInLayer = itemsInLayer.map(item => {
    const isReadOnlyMessage = item.message === 'Space is Read Only'
    if (globalStore.currentSpaceIsUnavailableOffline && isReadOnlyMessage) {
      item.message = 'Space is Unavailable Offline'
      item.icon = 'offline'
    }
    return item
  })
  return itemsInLayer
})
const remove = () => {
  globalStore.removeNotificationWithPosition()
}
</script>

<template lang="pug">
aside.notifications-with-position
  .item.badge(
    v-for="item in items"
    v-bind:key="item.id"
    :data-notification-id="item.id"
    :style="{ left: item.position.x + 'px', top: item.position.y + 'px' }"
    :class="item.type"
    @animationend="remove"
  )
    img.icon.cancel(v-if="item.icon === 'cancel'" src="@/assets/add.svg")
    img.icon.add(v-if="item.icon === 'add'" src="@/assets/add.svg")
    img.icon.checkmark(v-if="item.icon === 'checkmark'" src="@/assets/checkmark.svg")
    img.icon.cut(v-if="item.icon === 'cut'" src="@/assets/cut.svg")
    img.icon.offline(v-if="item.icon === 'offline'" src="@/assets/offline.svg")
    span {{item.message}}
</template>

<style lang="stylus">
.notifications-with-position
  .item
    position absolute
    z-index var(--max-z)
    pointer-events none
    animation-name hideme
    animation-delay 1s
    animation-duration 0.1s
    animation-iteration-count 1
    animation-direction forward
    animation-fill-mode forwards
    animation-timing-function ease-out
  .checkmark
    vertical-align 2px
</style>
