<script setup>
import { reactive, computed, onMounted, watch, ref, nextTick } from 'vue'

import { useGlobalStore } from '@/stores/useGlobalStore'
import { useSpaceStore } from '@/stores/useSpaceStore'

import Loader from '@/components/Loader.vue'
import cache from '@/cache.js'
import utils from '@/utils.js'

const globalStore = useGlobalStore()
const spaceStore = useSpaceStore()

const props = defineProps({
  visible: Boolean
})

watch(() => props.visible, (value, prevValue) => {
  if (value) {
    updateSpaceIsCached()
  }
})

const state = reactive({
  spaceIsCached: false
})

const currentSpace = computed(() => spaceStore.getSpaceAllState)

const refreshBrowser = () => {
  window.location.reload()
}

const updateSpaceIsCached = async () => {
  const cachedSpace = await cache.space(currentSpace.value.id)
  state.spaceIsCached = utils.arrayHasItems(cachedSpace.cards)
}

// loading

const isLoadingSpace = computed(() => globalStore.isLoadingSpace)
const isLoadingOtherItems = computed(() => globalStore.isLoadingOtherItems)
const isJoiningSpace = computed(() => globalStore.isJoiningSpace)
const isConnectingToBroadcast = computed(() => globalStore.isConnectingToBroadcast)

// saving

const sendingQueue = computed(() => globalStore.sendingQueue)
const isSavingOperations = computed(() => Boolean(sendingQueue.value.length))
const pluralChanges = computed(() => {
  const condition = sendingQueue.value.length !== 1
  return utils.pluralize('change', condition)
})

// connected

const isConnected = computed(() => {
  const value = !isLoadingSpace.value && !isJoiningSpace.value && !isConnectingToBroadcast.value && !isSavingOperations.value
  return value
})
</script>

<template lang="pug">
dialog.space-status(v-if="visible" :open="visible" ref="dialog")
  section
    .row.title-row
      div(v-if="isConnected")
        .badge.success Connected
      div(v-else)
        Loader(:visible="true")
        span(v-if="isLoadingSpace || isLoadingOtherItems") Downloading
        span(v-else-if="isJoiningSpace || isConnectingToBroadcast") Connecting to Broadcast
        span(v-else-if="isSavingOperations") Syncing
      .button-wrap
        button.small-button(@click.left="refreshBrowser" title="Refresh browser")
          img.icon(src="@/assets/refresh.svg")

  section(v-if="isConnected")
    p Updates synced
  section(v-else)
    div(v-if="(isLoadingSpace || isLoadingOtherItems) && state.spaceIsCached")
      span.badge.info You can edit right now
      span and your changes will sync once connected
    div(v-else-if="isJoiningSpace || isConnectingToBroadcast")
      span.badge.info You can edit right now
      span {{' '}}
      span but cannot collaborate yet, your changes will sync once connected
    div(v-else-if="isSavingOperations")
      span Your changes are saving to the server
      section.subsection.operations-list
        span.badge.info
          Loader(:visible="true" :isSmall="true" :isStatic="true")
          span {{sendingQueue.length}} {{pluralChanges}} to sync
</template>

<style lang="stylus">
dialog.space-status
  width 200px
  @media(max-width 414px)
    left -60px
  .badge
    display inline-block

  .loader
    width 14px
    height 14px
    vertical-align -3px
    margin-right 6px
  .loader + span
    margin-left 2px

  .operations-list
    margin-top 10px
</style>
