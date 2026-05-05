<script setup>
import { reactive, computed, onMounted, onUnmounted, onBeforeUnmount, watch, ref, nextTick } from 'vue'

import { useGlobalStore } from '@/stores/useGlobalStore'
import { useThemeStore } from '@/stores/useThemeStore'
import { useUserStore } from '@/stores/useUserStore'

import AboutJsonLd from '@/components/page/about/AboutJsonLd.vue'
import Header from '@/components/page/Header.vue'
import Wordmark from '@/components/page/Wordmark.vue'
import FooterSitemap from '@/components/page/FooterSitemap.vue'
import Footer from '@/components/page/Footer.vue'
import consts from '@/consts.js'

import ApiDocs from '@/data/apiDocs.md'

const globalStore = useGlobalStore()
const themeStore = useThemeStore()
const userStore = useUserStore()

const appsButtonElement = ref(null)
let unsubscribes

const state = reactive({
  currentItems: []
})

let tableWrapObserver

const observeTableWraps = () => {
  tableWrapObserver = new IntersectionObserver((entries) => {
    entries.forEach(entry => {
      const sectionClass = Array.from(entry.target.classList).find(c => c !== 'table-wrap')
      if (!sectionClass) return
      const link = `#${sectionClass}`
      if (entry.isIntersecting) {
        if (!state.currentItems.includes(link)) {
          state.currentItems.push(link)
        }
      } else {
        state.currentItems = state.currentItems.filter(item => item !== link)
      }
    })
  }, { threshold: 0.01 })
  document.querySelectorAll('.table-wrap').forEach(el => tableWrapObserver.observe(el))
}

if (!consts.isStaticPrerenderingPage) {
  window.globalStore = useGlobalStore()
  window.themeStore = useThemeStore()
  if (consts.isDevelopment()) {
    window.userStore = useUserStore()
  }
}

onMounted(() => {
  if (!consts.isStaticPrerenderingPage) {
    window.matchMedia('(prefers-color-scheme: dark)').addEventListener('change', logSystemThemeChange)
    window.matchMedia('(prefers-color-scheme: dark)').addEventListener('change', updateSystemTheme)
    themeStore.restoreTheme()
  }
  if (consts.isDevelopment()) {
    document.title = '[DEV] Kinopio API Docs'
  } else {
    document.title = 'Kinopio API Docs'
  }
  nextTick(() => observeTableWraps())
})

onBeforeUnmount(() => {
  tableWrapObserver?.disconnect()
})

const closeAllDialogs = () => {
  globalStore.closeAllDialogs()
}

// theme

const isThemeDark = computed(() => themeStore.getIsThemeDark)
const logSystemThemeChange = (event) => {
  const themeIsSystem = userStore.themeIsSystem
  console.warn('🌓 logSystemThemeChange', window.matchMedia('(prefers-color-scheme: dark)'), event, { themeIsSystem })
}
const updateSystemTheme = () => {
  themeStore.updateSystemTheme()
}

// toc

const items = computed(() => {
  const sections = [
    { name: 'All', color: 'khaki' },
    { name: 'Users', color: '#b9a8ff' },
    { name: 'Spaces', color: 'pink' },
    { name: 'Cards', color: 'violet' },
    { name: 'Connections', color: 'salmon' },
    { name: 'Boxes', color: 'lightskyblue' },
    { name: 'Lists', color: '#f9cb77' },
    { name: 'Tags', color: 'mediumaquamarine' },
    { name: 'Notifications', color: 'darkseagreen' },
    { name: 'Other', color: 'cadetblue' }
  ]
  return sections.map(item => {
    item.link = `#${item.name.toLowerCase()}`
    return item
  })
})
</script>

<template lang="pug">
AboutJsonLd
.page(:class="{ 'is-dark-theme': isThemeDark }")
  Header(:isDocumentPage="true")
  main.page(@click="closeAllDialogs")
    .page-wrap
      section.intro
        Wordmark
        h2 API Documentation
        ul.api-contents
          li(v-for="item in items")
            a(:href="item.link")
              .badge.button-badge(:style="{ background: item.color }" :class="{ active: state.currentItems.includes(item.link) }") {{item.name}}
        article.api
          //- img.cat(src="https://kinopio.club/help/assets/cat.png")
          ApiDocs

      FooterSitemap
  Footer
</template>

<style lang="stylus">
ul.api-contents
  padding 0
  position sticky
  top 45px
  z-index 10
  li
    list-style none
    display inline-block
    a
      text-decoration none

article.api
  .anchor
    padding-top 100px // offset anchor link
    margin-top -100px
    display block

  h2
    font-size 18px
  h3
    font-size 16px
  h2,
  h3
    &.badge
      display inline-block
      padding 4px 5px
      padding-bottom 2px
      font-family sans-serif
      font-weight bold
      margin-bottom 0

    // TODO dark themes and programatic var colors
    // .all,
    // .users,
    // .spaces,
    // .cards,
    // .connections,
    // .connection-types,
    // .boxes,
    // .lists,
    // .tags,
    // .notifications,
    // .other
    //   color var(--primary)
    //   padding 2px 5px
    // .all
    //   background khaki
    // .users
    //   background #b9a8ff
    // .spaces
    //   background pink
    // .cards
    //   background violet
    // .connections
    //   background salmon
    // .boxes
    //   background lightskyblue
    // .lists
    //   background #f9cb77
    // .tags
    //   background mediumaquamarine
    // .notifications
    //   background darkseagreen
    // .other
    //   background cadetblue

  .badge,
  code
    color var(--primary)
    // padding 2px 5px
    &.all
      background khaki
    &.users
      background #b9a8ff
    &.spaces
      background pink
    &.cards
      background violet
    &.connections
      background salmon
    &.boxes
      background lightskyblue
    &.lists
      background #f9cb77
    &.tags
      background mediumaquamarine
    &.notifications
      background darkseagreen
    &.other
      background cadetblue
  a.badge
    text-decoration none
    color var(--primary)

  .table-wrap
    table
      background var(--primary-background)
      border-width 2px
      border-style solid
      &.all
        border-color khaki
      &.users
        border-color #b9a8ff
      &.spaces
        border-color pink
      &.cards
        border-color violet
      &.connections
        border-color salmon
      &.boxes
        border-color lightskyblue
      &.lists
        border-color #f9cb77
      &.tags
        border-color mediumaquamarine
      &.notifications
        border-color darkseagreen
      &.other
        border-color cadetblue

</style>
