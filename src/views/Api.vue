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
              .badge.button-badge(:style="{ background: item.color }") {{item.name}}
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
//   .cat
//     width 100px
</style>
