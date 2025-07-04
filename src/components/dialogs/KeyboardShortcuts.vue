<script setup>
import { reactive, computed, onMounted, onBeforeUnmount, watch, ref, nextTick } from 'vue'

import { useUserStore } from '@/stores/useUserStore'
import { useSpaceStore } from '@/stores/useSpaceStore'

import UserLabelInline from '@/components/UserLabelInline.vue'
import keyboardShortcutsCategories from '@/data/keyboardShortcutsCategories.js'
import postMessage from '@/postMessage.js'
import utils from '@/utils.js'

const userStore = useUserStore()
const spaceStore = useSpaceStore()

const dialogElement = ref(null)

const props = defineProps({
  visible: Boolean
})
const state = reactive({
  selectedCategory: 'All'
})

watch(() => props.visible, (value, prevValue) => {
  if (value) {
    state.selectedCategory = 'All'
  }
})

const updateDialogHeight = async () => {
  if (!props.visible) { return }
  await nextTick()
  const element = dialogElement.value
  state.dialogHeight = utils.elementHeight(element)
}

const categories = computed(() => keyboardShortcutsCategories)
const meta = computed(() => utils.metaKey())
const option = computed(() => utils.optionKey())
const currentUser = computed(() => userStore.getUserAllState)
const isMobile = computed(() => utils.isMobile())
const shouldUseLastConnectionType = computed(() => userStore.shouldUseLastConnectionType)
const lastOrNewConnectionTypeControlSetting = computed(() => {
  if (shouldUseLastConnectionType.value) {
    return 'New'
  } else {
    return 'Last'
  }
})

const updateSelectedCategory = (name) => {
  state.selectedCategory = name
}
const categoryButtonIsVisible = (name) => {
  return state.selectedCategory === name
}
const categoryIsVisible = (name) => {
  return state.selectedCategory === 'All' || state.selectedCategory === name
}
const categoryByName = (name) => {
  return categories.value.find(category => category.name === name)
}
const categoryColor = (name) => {
  const color = categoryByName(name).color
  return color
}
const closeDialogs = () => {
  // this.keyboardShortcutsCategoriesIsVisible = false
}

// [.] disable checkboxes

const checkboxStateNewSpace = computed({
  get () {
    return isChecked('newSpace')
  },
  set () {
    toggleChecked('newSpace')
  }
})
const isChecked = (value) => {
  const disabledKeyboardShortcuts = userStore.disabledKeyboardShortcuts
  const isEnabled = !disabledKeyboardShortcuts.includes(value)
  return isEnabled
}
const toggleChecked = (name) => {
  const prevValue = isChecked(name)
  if (prevValue) {
    userStore.addToDisabledKeyboardShortcuts(name)
  } else {
    userStore.removeFromDisabledKeyboardShortcuts(name)
  }
  postMessage.sendHaptics({ name: 'heavyImpact' })
  event.stopPropagation()
}
</script>

<template lang="pug">
dialog.keyboard-shortcuts(v-if="visible" :open="visible" @click.left.stop ref="dialogElement" @click="closeDialogs" :style="{'max-height': state.dialogHeight + 'px'}")
  section
    .row
      p Keyboard Shortcuts
      .badge.keyboard-shortcut ?
    .categories
      template(v-for="category in categories" :key="category.name")
        .badge.secondary.button-badge(:class="{ active: categoryButtonIsVisible(category.name) }" :style="{ 'background-color': category.color }" @click="updateSelectedCategory(category.name)") {{category.name}}
  section
    //- General
    template(v-if="categoryIsVisible('General')")
      .section-title
        .badge.info(:style="{ 'background-color': categoryColor('General') }") General
      article
        .row
          .badge.title
            //- [.]
            .checkbox-wrap(title="Uncheck to disable N shortcut")
              label(:class="{active: checkboxStateNewSpace }")
                input(name="checkbox" type="checkbox" v-model="checkboxStateNewSpace")
            img.icon(src="@/assets/add.svg" :class="{'is-disabled': !isChecked('newSpace')}")
            span(:class="{'is-disabled': !isChecked('newSpace')}") New Space
          .badge.keyboard-shortcut(:class="{'is-disabled': !isChecked('newSpace')}") N
      article
        .row
          .badge.title
            img.icon.dark(src="@/assets/dark.svg")
            span Toggle Dark Theme
          .badge.keyboard-shortcut T
      article
        .row
          .badge.title
            img.icon.cancel(src="@/assets/add.svg")
            span Close Dialogs
          .badge.keyboard-shortcut Escape

    //- General
    template(v-if="categoryIsVisible('Toolbar')")
      .section-title
        .badge.info(:style="{ 'background-color': categoryColor('Toolbar') }") Toolbar
      article
        .row
          .badge.title
            img.icon.box-icon(src="@/assets/box.svg")
            span Box Mode
          .badge.keyboard-shortcut B
        .row
          .badge.title
            img.icon.pencil(src="@/assets/pencil.svg")
            span Drawing Mode
          .badge.keyboard-shortcut D
        .row
          .badge.title
            img.icon.brush-size(src="@/assets/brush-size-l.svg")
            span Cycle Brush Size
          .badge.keyboard-shortcut S
        .row
          .badge.title
            img.icon.eraser(src="@/assets/eraser.svg")
            span Toggle Eraser
          .badge.keyboard-shortcut E

    //- Navigate
    template(v-if="categoryIsVisible('Navigate')")
      .section-title
        .badge.info(:style="{ 'background-color': categoryColor('Navigate') }") Navigate
      article
        .row
          .badge.title
            img.icon.presentation(src="@/assets/presentation.svg")
            span Presentation Mode
          .badge.keyboard-shortcut P
      article
        .row
          .badge.title
            img.icon.hand(src="@/assets/hand.svg")
            span Drag to Pan
          .badge.keyboard-shortcut Space/Right-Click Drag
      article
        .row
          .badge.title
            img.icon.magnifying-glass(src="@/assets/magnifying-glass.svg")
            span Zoom In or Out
          .badge.keyboard-shortcut {{meta}}-+/-, {{meta}}-Scroll
      article
        .row
          .badge.title
            img.icon.magnifying-glass(src="@/assets/magnifying-glass-negative.svg")
            span Toggle Max Zoom Out
          .badge.keyboard-shortcut Z
      article
        .row
          .badge.title
            img.icon.minimap(src="@/assets/minimap.svg")
            span Toggle Minimap
          .badge.keyboard-shortcut M

    //- Edit
    template(v-if="categoryIsVisible('Edit')")
      .section-title
        .badge.info(:style="{ 'background-color': categoryColor('Edit') }") Edit
      article
        .row.multiple-items
          .badge.title
            img.icon(src="@/assets/add.svg")
            span Add Card
          .badge.keyboard-shortcut Enter
      article
        .row
          .badge.title
            img.icon(src="@/assets/add.svg")
            span Add Child Card
          .badge.keyboard-shortcut Shift-Enter
        p Subsequent&nbsp;
          span.badge.keyboard-shortcut Enter
          span adds siblings
      article
        .row
          .badge.title
            img.icon(src="@/assets/line-break.svg")
            span Line break in card
          .badge.keyboard-shortcut Ctrl-Enter
      article
        .row
          .badge.title
            img.icon(src="@/assets/constrain-axis.svg")
            span Snap Card or Box to Grid
          .badge.keyboard-shortcut Shift-Drag Drag or Resize Item

      //- article
      //-   .row
      //-     .badge.title Focus Nearest Card
      //-     .badge.keyboard-shortcut Arrow(→↑←↓)
      article
        .row
          .badge.title
            img.icon(src="@/assets/lock.svg")
            span Toggle Lock Cards
          .badge.keyboard-shortcut {{meta}}-Shift-L
      article
        .row
          .badge.title
            img.icon(src="@/assets/box.svg")
            span Surround Selected Cards with Box
          .badge.keyboard-shortcut B
      article
        .row
          .badge.title
            img.icon(src="@/assets/undo.svg")
            span Undo/Redo
          .badge.keyboard-shortcut {{meta}}-Z/{{meta}}-Shift-Z

    //- Select
    template(v-if="categoryIsVisible('Select')")
      .section-title
        .badge.info(:style="{ 'background-color': categoryColor('Select') }") Select
      article
        .row
          .badge.title
            img.icon(src="@/assets/box-select.svg")
            span Box Select
          .badge.keyboard-shortcut Shift-Drag
      article
        .row
          .badge.title
            img.icon(src="@/assets/brush.svg")
            span Select All Cards
          .badge.keyboard-shortcut {{meta}}-A
      article
        .row
          .badge.title
            img.icon(src="@/assets/brush-y.svg")
            span Select All Cards Below Cursor
          .badge.keyboard-shortcut {{meta}}-Shift-A
      article
        .row
          .badge.title
            img.icon(src="@/assets/brush.svg")
            span Select All Connected Cards
          .badge.keyboard-shortcut {{meta}}-Click Card
      article
        .row
          .badge.title
            img.icon(src="@/assets/brush.svg")
            span Select All Cards Inside Box
          .badge.keyboard-shortcut {{meta}}-Click Box
      article
        .row
          .badge.title
            img.icon.box-icon(src="@/assets/box.svg")
            span Move Box Without Moving Cards
          .badge.keyboard-shortcut {{option}}-Drag on Box
      article
        .row
          .badge.title
            img.icon.cut(src="@/assets/cut.svg")
            span Copy/Cut/Paste Selected Cards
          .badge.keyboard-shortcut {{meta}}-C/{{meta}}-X/{{meta}}-V
        p You can copy and paste cards between spaces
      article
        .row
          .badge.title
            img.icon(src="@/assets/remove.svg")
            span Remove Selected
          .badge.keyboard-shortcut Delete

    //- Filter
    template(v-if="categoryIsVisible('Filter')")
      .section-title
        .badge.info(:style="{ 'background-color': categoryColor('Filter') }") Filter
      article
        .row
          .badge.title
            UserLabelInline(:user="currentUser" :shouldHideName="true")
            span Toggle Card User Filter
          .badge.keyboard-shortcut 1
      article
        .row
          .badge.title
            img.icon.time(src="@/assets/time.svg")
            span Toggle Card Date Filter
          .badge.keyboard-shortcut 2
      article
        .row
          .badge.title
            img.icon.time(src="@/assets/unchecked.svg")
            span Toggle Cards Unchecked Filter
          .badge.keyboard-shortcut 3
      article
        .row
          .badge.title
            img.icon.time(src="@/assets/comment.svg")
            span Toggle Hide Comment Cards
          .badge.keyboard-shortcut 4

    //- Connect
    template(v-if="categoryIsVisible('Connect')")
      .section-title
        .badge.info(:style="{ 'background-color': categoryColor('Connect') }") Connect
      article
        .row
          .badge.title
            img.icon.connector-icon(src="@/assets/connector-open.svg")
            span Use {{lastOrNewConnectionTypeControlSetting}} Connection Type
          .badge.keyboard-shortcut Shift-Click on
            img.icon.connector-icon(src="@/assets/connector-open.svg")
        p
          span.badge.keyboard-shortcut Shift-Drag
          span card connector or
          span.badge.keyboard-shortcut Shift-Click
          span 'Connect' button to use {{lastOrNewConnectionTypeControlSetting}} connection type

    //- Search and Jump
    template(v-if="categoryIsVisible('Search and Jump')")
      .section-title
        .badge.info(:style="{ 'background-color': categoryColor('Search and Jump') }") Search and Jump
      article
        .row
          .badge.title
            img.icon(src="@/assets/search.svg")
            span Search/Jump-to Spaces
          .badge.keyboard-shortcut {{meta}}–K
      article
        .row
          .badge.title
            img.icon(src="@/assets/search.svg")
            span Search/Jump-to Cards in Current Space
          .badge.keyboard-shortcut {{meta}}–F
      article
        .row
          .badge.title
            img.icon(src="@/assets/search.svg")
            span Search/Jump-to Cards in All Spaces
          .badge.keyboard-shortcut {{meta}}–Shift–F

</template>

<style lang="stylus">
.keyboard-shortcuts
  user-select text
  overflow auto
  max-height calc(100vh - 60px)
  span
    color var(--primary)
  .title
    padding-left 0
  .badge
    display inline-block
    color var(--primary)

  .badge.info
    img
      margin-left 6px

  article
    position static
    margin-bottom 10px
    padding-bottom 10px
    border-bottom 1px solid var(--primary-border)
    &:last-child
      margin-bottom 0
      padding-bottom 0
      border-bottom 0
  .row
    display flex
    justify-content space-between
  .multiple-items
    .badge
      margin-right 0
    .divider
      padding-left 0
      margin-right 6px
  .badge.title + .badge.info
    margin-right 0
  .connector-icon
    width 11px
  video
    margin-top 10px
  .user
    margin-right 5px !important
  .magnifying-glass
    vertical-align -2px
  .keyboard-shortcut
    margin 0
    .icon
      margin-left 3px
  .keyboard-shortcut + span
    margin-left 6px
  .section-title
    margin-bottom 10px
    .badge
      color var(--primary-on-light-background)

  .hand
    vertical-align middle
  .pencil
    vertical-align -2px
  .brush-size,
  .eraser
    vertical-align -1px
  .categories
    margin-top -6px
    .button-badge
      color var(--primary)
    .button-badge + .button-badge
      margin-top 6px
      color var(--primary-on-light-background)

  .inbox-icon
    margin 0

  .icon.minimap,
  .icon.presentation
    width 12px
    vertical-align -1px

  .checkbox-wrap
    display inline-block
    label
      padding 0
      padding-left 5px
    margin-right 6px

  .is-disabled
    opacity 0.5
</style>
