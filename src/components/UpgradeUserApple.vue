<script setup>
import { reactive, computed, onMounted, defineProps, defineEmits, watch, ref, nextTick } from 'vue'
import { useStore } from 'vuex'

import User from '@/components/User.vue'
import Loader from '@/components/Loader.vue'
import postMessage from '@/postMessage.js'
import consts from '@/consts.js'
import utils from '@/utils.js'
const store = useStore()

onMounted(() => {
  window.addEventListener('message', handleSubscriptionSuccess) // iOS IAP subscription sheet transaction completes
})

const props = defineProps({
  visible: Boolean,
  price: Object // amount, applePriceId
})

const state = reactive({
  loading: {
    subscriptionIsBeingCreated: false
  },
  error: {
    unknownServerError: false
  }
})

const user = computed(() => store.state.currentUser)
const isUpgraded = computed(() => store.state.currentUser.isUpgraded)
const appleAppAccountToken = computed(() => store.state.currentUser.appleAppAccountToken)

const clearErrors = () => {
  state.error.unknownServerError = false
  state.error.subscriptionError = false
}

const subscribe = async () => {
  console.info('💰 appleAppAccountToken', appleAppAccountToken.value)
  clearErrors()
  if (state.loading.subscriptionIsBeingCreated) { return }
  state.loading.subscriptionIsBeingCreated = true
  try {
    if (!appleAppAccountToken.value) {
      throw { message: 'no user.appleAppAccountToken' }
    }
    const body = {
      name: 'createSubscription',
      value: {
        appleAppAccountToken: appleAppAccountToken.value,
        appleSubscriptionId: props.price.applePriceId
      }
    }
    console.info('🎡', body)
    postMessage.send(body)
  } catch (error) {
    console.error('🚒', error)
    state.error.unknownServerError = true
  }
}

const handleSubscriptionSuccess = (event) => {
  if (!consts.isSecureAppContext) { return }
  const data = event.data
  state.loading.subscriptionIsBeingCreated = false
  console.info('🎡 handleSubscriptionSuccess', data)
  if (data.name !== 'upgradedUser') { return }
  if (!data.isSuccess) {
    state.error.subscriptionError = true
    state.loading.subscriptionIsBeingCreated = false
    return
  }
  store.commit('currentUser/isUpgraded', true)
  store.commit('currentUser/appleSubscriptionIsActive', true)
  store.commit('notifyCardsCreatedIsOverLimit', false)
  if (!utils.dialogIsVisible()) {
    store.commit('addNotification', {
      message: 'Your account has been upgraded. Thank you for supporting independent, ad-free, sustainable software',
      type: 'success',
      isPersistentItem: true
    })
  }
}

</script>

<template lang="pug">
.upgrade-user-apple(v-if="visible")
  .row(v-if='!isUpgraded')
    button(@click.left="subscribe" :class="{active : state.loading.subscriptionIsBeingCreated}")
      User(:user="user" :isClickable="false" :hideYouLabel="true" :key="user.id" :isSmall="true")
      span Upgrade for ${{price.amount}}/{{price.period}}
      Loader(:visible="state.loading.subscriptionIsBeingCreated")
  .badge.danger(v-if="state.error.unknownServerError")
    span (シ_ _)シ Something went wrong, Please try again or contact support. Your transaction was not processed.
  .badge.danger(v-if='state.error.subscriptionError')
    span (シ_ _)シ Your subscription was not processed, Please try again or contact support.
  .badge.success(v-if="isUpgraded")
    span Your account has been upgraded. Thank you for supporting independent, ad-free, sustainable software

</template>

<style lang="stylus">
.upgrade-user-apple
  a
    color var(--primary)
</style>
