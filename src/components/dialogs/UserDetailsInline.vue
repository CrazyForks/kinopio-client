<script setup>
import { reactive, computed, onMounted, onUnmounted, watch, ref, nextTick } from 'vue'

import UserDetailsInfo from '@/components/UserDetailsInfo.vue'
import UserDetailsActions from '@/components/UserDetailsActions.vue'
import utils from '@/utils.js'

const dialogElement = ref(null)

const props = defineProps({
  visible: Boolean,
  user: Object,
  showExploreSpaces: Boolean
})

</script>

<template lang="pug">
dialog.narrow.user-details.user-details-inline(v-if="props.visible" @keyup.stop :open="props.visible" @click.left.stop="closeDialogs" @keydown.stop ref="dialogElement" title="")
  UserDetailsInfo(:user="props.user" :showExploreSpaces="true" :showUserBadges="true")
  UserDetailsActions(:user="props.user" :showExploreSpaces="true")
</template>

<style lang="stylus">
.user-details
  cursor initial
  top calc(100% - 8px)
  left initial
  right 8px
  position absolute
</style>
