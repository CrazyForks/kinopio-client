<script setup>
import { reactive, computed, onMounted, watch, ref, nextTick } from 'vue'
import { useStore } from 'vuex'

import DiscountRow from '@/components/DiscountRow.vue'
import UserLabelInline from '@/components/UserLabelInline.vue'
import CardsCreatedProgress from '@/components/CardsCreatedProgress.vue'
import AboutMe from '@/components/AboutMe.vue'
import UpgradeFAQ from '@/components/dialogs/UpgradeFAQ.vue'
import AboutGroups from '@/components/dialogs/AboutGroups.vue'
import consts from '@/consts.js'
import utils from '@/utils.js'
const store = useStore()

const props = defineProps({
  visible: Boolean
})
const dialog = ref(null)

onMounted(() => {
  window.addEventListener('resize', updateDialogHeight)
  store.subscribe((mutation, state) => {
    if (mutation.type === 'triggerCloseChildDialogs') {
      closeChildDialogs()
    }
  })
})

const state = reactive({
  dialogHeight: null,
  upgradeFAQIsVisible: false,
  aboutGroupsIsVisible: false
})

watch(() => props.visible, (value, prevValue) => {
  if (value) {
    closeChildDialogs()
    updateDialogHeight()
    store.commit('shouldExplicitlyHideFooter', true)
  } else {
    store.commit('shouldExplicitlyHideFooter', false)
  }
})

const updateDialogHeight = async () => {
  if (!props.visible) { return }
  await nextTick()
  state.dialogHeight = utils.elementHeight(dialog.value)
}

const isSecureAppContextIOS = computed(() => consts.isSecureAppContextIOS)
const studentDiscountIsAvailable = computed(() => store.state.currentUser.studentDiscountIsAvailable)
const monthlyPrice = computed(() => consts.price('month').amount)
const yearlyPrice = computed(() => {
  const isStudentDiscount = store.state.currentUser.studentDiscountIsAvailable
  return consts.price('year', isStudentDiscount).amount
})
const lifePrice = computed(() => consts.price('life').amount)

// child dialogs

const toggleUpgradeFAQIsVisible = () => {
  const value = !state.upgradeFAQIsVisible
  closeDialogs()
  state.upgradeFAQIsVisible = value
}
const toggleAboutGroupsIsVisible = () => {
  const value = !state.aboutGroupsIsVisible
  closeDialogs()
  state.aboutGroupsIsVisible = value
}
const closeDialogs = () => {
  store.commit('triggerCloseChildDialogs')
}
const closeChildDialogs = () => {
  state.upgradeFAQIsVisible = false
  state.aboutGroupsIsVisible = false
}

// free cards from space member

const spaceCreatorIsUpgraded = computed(() => store.getters['currentSpace/spaceCreatorIsUpgraded'])
const spaceUser = computed(() => store.state.currentSpace.users[0])

</script>

<template lang="pug">
dialog.pricing(v-if="visible" :open="visible" @click.left.stop="closeDialogs" ref="dialog" :style="{'max-height': state.dialogHeight + 'px'}")
  section
    .row.title-row
      //- price
      template(v-if="isSecureAppContextIOS")
        p Kinopio is free for 100 cards, afterwards it's ${{monthlyPrice}}/month or ${{yearlyPrice}}/year
      template(v-else)
        p Kinopio is free for 100 cards, afterwards it's ${{monthlyPrice}}/month, ${{yearlyPrice}}/year, or ${{lifePrice}}/life
      .button-wrap
        button.small-button(@click.stop="toggleUpgradeFAQIsVisible" :class="{active: state.upgradeFAQIsVisible}")
          span ?
          UpgradeFAQ(:visible="state.upgradeFAQIsVisible")
    p.badge.success(v-if="studentDiscountIsAvailable") Your account qualifies for a student discount
    DiscountRow
    table
      tbody
        tr.table-header
          td
            span Free
          td
            span.badge.success Upgraded
        tr
          td 100 cards
          td Unlimited cards
        tr
          td 5mb file upload size limit
          td No upload limit
        tr
          td Can only join Groups
          td
            .row
              span Can create Groups
              button.small-button(@click.stop="toggleAboutGroupsIsVisible" :class="{ active: state.aboutGroupsIsVisible }" title="About Groups")
                span ?
                AboutGroups(:visible="state.aboutGroupsIsVisible")
    CardsCreatedProgress
    //- free cards from space member
    section.subsection(v-if="spaceCreatorIsUpgraded")
      p
        UserLabelInline(:user="spaceUser")
        span is upgraded, so cards you create in this space won't increase your free card count
  section
    AboutMe
</template>

<style lang="stylus">
dialog.pricing
  overflow auto
  left initial
  right 8px
  max-height calc(100vh - 25px)
  p
    user-select text
  .subsection
    margin-top 10px
  table
    td
      max-width 120px
  @media(max-height 650px)
    top -60px !important
  .title-row
    align-items flex-start
    .button-wrap,
    .small-button
      margin-top 0

</style>
