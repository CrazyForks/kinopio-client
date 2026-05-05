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

const items = computed(() => [
  { name: 'All', link: '#all', color: 'khaki' },
  { name: 'Users', link: '#users', color: '#b9a8ff' },
  { name: 'Spaces', link: '#spaces', color: 'pink' },
  { name: 'Cards', link: '#cards', color: 'violet' },
  { name: 'Connections', link: '#connections', color: 'salmon' },
  { name: 'Boxes', link: '#boxes', color: 'lightskyblue' },
  { name: 'Lists', link: '#lists', color: '#f9cb77' },
  { name: 'Tags', link: '#tags', color: 'mediumaquamarine' },
  { name: 'Notifications', link: '#notifications', color: 'darkseagreen' },
  { name: 'Other', link: '#other', color: 'cadetblue' }
])
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
              .badge(:style="{ background: item.color }") {{item.name}}

        article.api
          //- img.cat(src="https://kinopio.club/help/assets/cat.png")
          ApiDocs

      FooterSitemap
  Footer
</template>

<style lang="stylus">
// article.api
//   .cat
//     width 100px
</style>
