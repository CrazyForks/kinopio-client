<script setup>
import { reactive, computed, onMounted, onBeforeUnmount, onUpdated, onUnmounted, watch, ref, nextTick } from 'vue'
import { useStore } from 'vuex'

import utils from '@/utils.js'
import consts from '@/consts.js'
import fonts from '@/data/fonts.js'
import ItemConnectorButton from '@/components/ItemConnectorButton.vue'
import smartquotes from 'smartquotes'
import postMessage from '@/postMessage.js'

import randomColor from 'randomcolor'
import { colord, extend } from 'colord'
const store = useStore()

let unsubscribe

const borderWidth = 2

// locking
// long press to touch drag
const lockingPreDuration = 100 // ms
const lockingDuration = 100 // ms
let lockingAnimationTimer, lockingStartTime, shouldCancelLocking
let isMultiTouch
let initialTouchEvent = {}
let touchPosition = {}
let currentTouchPosition = {}

let observer

let prevSelectedBox

const boxElement = ref(null)

onMounted(() => {
  unsubscribe = store.subscribe((mutation, state) => {
    const { type, payload } = mutation
    if (type === 'updateRemoteCurrentConnection' || type === 'removeRemoteCurrentConnection') {
      updateRemoteConnections()
    } else if (type === 'isLoadingSpace') {
      updateCurrentConnections()
    } else if (type === 'triggerUpdateItemCurrentConnections') {
      const itemId = payload
      if (itemId !== props.box.id) { return }
      updateCurrentConnections()
    }
  })
  initViewportObserver()
  updateCurrentConnections()
})
onUpdated(() => {
  initViewportObserver()
})
onBeforeUnmount(() => {
  removeViewportObserver()
  unsubscribe()
})

const props = defineProps({
  box: Object
})
const state = reactive({
  isHover: false,
  isLocking: false,
  lockingPercent: 0,
  lockingAlpha: 0,
  isVisibleInViewport: false,
  shouldRenderParent: false,
  // connections
  currentConnections: [],
  isRemoteConnecting: false,
  remoteConnectionColor: ''
})

const spaceCounterZoomDecimal = computed(() => store.getters.spaceCounterZoomDecimal)
const canEditBox = computed(() => store.getters['currentUser/canEditBox'](props.box))
const currentUserIsSignedIn = computed(() => store.getters['currentUser/isSignedIn'])
const currentUserColor = computed(() => store.state.currentUser.color)
const name = computed(() => props.box.name)

// normalize

const normalizedBox = computed(() => {
  return normalizeBox(props.box)
})
const normalizeBox = (box) => {
  box = utils.clone(box)
  box.resizeWidth = box.resizeWidth || consts.minItemXY
  box.resizeHeight = box.resizeHeight || consts.minItemXY
  box.width = box.resizeWidth
  box.height = box.resizeHeight
  box.color = box.color || randomColor({ luminosity: 'light' })
  box.fill = box.fill || 'filled'
  return box
}
const normalizedName = computed(() => {
  // name without checkbox text
  let newName = name.value
  if (!newName) { return }
  const checkbox = utils.checkboxFromString(newName)
  if (checkbox) {
    newName = newName.replace(checkbox, '')
  }
  return newName.trim()
})
const nameSegments = computed(() => {
  const markdownSegments = utils.markdownSegments(normalizedName.value)
  return markdownSegments
})
const smartQuotes = (string) => {
  return smartquotes(string)
}

// should render

const updateShouldRenderParent = (value) => {
  state.shouldRenderParent = value
}
const shouldRender = computed(() => {
  return state.isVisibleInViewport || state.shouldRenderParent
})

// is visible in viewport

const initViewportObserver = async () => {
  removeViewportObserver()
  await nextTick()
  try {
    const callback = (entries, observer) => {
      entries.forEach((entry) => {
        if (entry.isIntersecting) {
          state.isVisibleInViewport = true
        } else {
          state.isVisibleInViewport = false
        }
      })
    }
    const target = boxElement.value
    if (!target) { return }
    observer = new IntersectionObserver(callback, { rootMargin: '50%' })
    observer.observe(target)
  } catch (error) {
    console.error('🚒 boxElement initViewportObserver', error)
  }
}
const removeViewportObserver = () => {
  const target = boxElement.value
  if (!observer) { return }
  observer.unobserve(target)
}

// styles

const styles = computed(() => {
  const { x, y, z, resizeWidth, resizeHeight, background } = normalizedBox.value
  const width = resizeWidth
  const height = resizeHeight
  const styles = {
    left: x + 'px',
    top: y + 'px',
    zIndex: z ?? 1,
    width: width + 'px',
    height: height + 'px',
    border: `${borderWidth}px solid ${color.value}`
  }
  // dimensions set by currentBoxes/resize while resizing
  if (isResizing.value) {
    styles.width = normalizedBox.value.resizeWidth
    styles.height = normalizedBox.value.resizeHeight
  }
  if (hasFill.value && !background) {
    let fillColor = color.value
    fillColor = colord(fillColor).alpha(0.5).toRgbString()
    styles.backgroundColor = fillColor
  }
  return styles
})
const backgroundStyles = computed(() => {
  if (!props.box.background) { return }
  const newStyles = utils.clone(styles.value)
  delete newStyles.border
  delete newStyles.backgroundColor
  const isRetina = utils.urlIsRetina(props.box.background)
  if (isRetina) {
    newStyles.backgroundImage = `image-set("${props.box.background}" 2x)`
  } else {
    newStyles.backgroundImage = `url("${props.box.background}")`
  }
  if (props.box.backgroundIsStretch) {
    newStyles.backgroundSize = 'cover'
  }
  return newStyles
})
const userColor = computed(() => store.state.currentUser.color)
const color = computed(() => {
  const remoteColor = remoteBoxDetailsVisibleColor.value || remoteSelectedColor.value || remoteUserResizingBoxesColor.value || remoteBoxDraggingColor.value
  if (remoteColor) {
    return remoteColor
  } else if (currentBoxIsSelected.value) {
    return userColor.value
  } else {
    return normalizedBox.value.color
  }
})
const colorIsDark = computed(() => utils.colorIsDark(color.value))
const fill = computed(() => normalizedBox.value.fill)
const hasFill = computed(() => fill.value !== 'empty')
const infoClasses = computed(() => {
  const classList = []
  if (isPainting.value) {
    classList.push('unselectable')
  }
  if (colorIsDark.value) {
    classList.push('is-dark')
  }
  const fontId = props.box.headerFontId || 0
  classList.push(`header-font-${fontId}`)
  const fontSize = props.box.headerFontSize || 's'
  classList.push(`header-font-size-${fontSize}`)
  const font = fonts.find(item => item.id === fontId)
  const fontSizeModifier = font?.size || ''
  if (fontSizeModifier) {
    classList.push(`header-font-size-modifier-${fontSizeModifier}`)
  }
  return classList
})
const classes = computed(() => {
  return {
    hover: state.isHover,
    active: currentBoxIsBeingDragged.value,
    'is-resizing': isResizing.value,
    'is-selected': currentBoxIsSelected.value,
    'is-checked': isChecked.value || isInCheckedBox.value,
    filtered: isFiltered.value,
    transition: !store.state.currentBoxIsNew || !store.state.currentUserIsResizingBox
  }
})

// space filters

const filtersIsActive = computed(() => {
  return Boolean(store.getters['currentUser/totalItemFadingFiltersActive'])
})
const isFilteredByUnchecked = computed(() => {
  const filterUncheckedIsActive = store.state.currentUser.filterUnchecked
  if (!filterUncheckedIsActive) { return }
  return !isChecked.value && hasCheckbox.value
})
const isFilteredByBox = computed(() => {
  const boxIds = store.state.filteredBoxIds
  return boxIds.includes(props.box.id)
})
const isFiltered = computed(() => {
  if (!filtersIsActive.value) { return }
  const isInFilter = isFilteredByUnchecked.value || isFilteredByBox.value
  return !isInFilter
})

// resize

const resizeIsVisible = computed(() => {
  if (isLocked.value) { return }
  if (!canEditSpace.value) { return }
  return true
})
const startResizing = (event) => {
  if (!canEditSpace.value) { return }
  if (utils.isMultiTouch(event)) { return }
  store.dispatch('history/pause')
  store.dispatch('closeAllDialogs')
  store.commit('currentUserIsResizingBox', true)
  store.commit('preventMultipleSelectedActionsIsVisible', true)
  let boxIds = [props.box.id]
  const multipleBoxesSelectedIds = store.state.multipleBoxesSelectedIds
  if (multipleBoxesSelectedIds.length) {
    boxIds = multipleBoxesSelectedIds
  }
  store.commit('currentUserIsResizingBoxIds', boxIds)
  const updates = {
    userId: store.state.currentUser.id,
    boxIds
  }
  store.commit('broadcast/updateStore', { updates, type: 'updateRemoteUserResizingBoxes' })
  event.preventDefault() // allows resizing box without scrolling on mobile
}
const resizeColorClass = computed(() => {
  const colorClass = utils.colorClasses({ backgroundColorIsDark: colorIsDark.value })
  return [colorClass]
})

// shrink

const shrinkToDefaultBoxSize = () => {
  const updated = { id: props.box.id }
  updated.resizeWidth = consts.defaultBoxWidth
  updated.resizeHeight = consts.defaultBoxHeight
  store.dispatch('currentBoxes/update', updated)
}
const shrink = () => {
  prevSelectedBox = props.box
  const { cards, boxes } = containedItems()
  prevSelectedBox = null
  const items = cards.concat(boxes)
  if (!items.length) {
    shrinkToDefaultBoxSize()
    return
  }
  const rect = utils.boundaryRectFromItems(items)
  const padding = consts.spaceBetweenCards
  const paddingTop = 30 + padding
  const updated = { id: props.box.id }
  updated.x = rect.x - padding
  updated.y = rect.y - paddingTop
  updated.resizeWidth = rect.width + (padding * 2)
  updated.resizeHeight = rect.height + (padding + paddingTop)
  store.dispatch('currentBoxes/update', updated)
}

// locked to background

const isLocked = computed(() => props.box.isLocked)

// label

const infoStyles = computed(() => {
  const styles = {
    backgroundColor: color.value
  }
  if (isLocked.value) {
    styles.pointerEvents = 'none'
  }
  return styles
})

// interacting

const updateCurrentConnections = async () => {
  await nextTick()
  state.currentConnections = store.getters['currentConnections/byItemId'](props.box.id)
}
const isPainting = computed(() => store.state.currentUserIsPainting)
const canEditSpace = computed(() => store.getters['currentUser/canEditSpace']())
const currentBoxIsBeingDragged = computed(() => {
  const isDragging = store.state.currentUserIsDraggingBox
  const isCurrent = store.state.currentDraggingBoxId === props.box.id
  return isDragging && (isCurrent || currentBoxIsSelected.value)
})
const isResizing = computed(() => {
  const isResizing = store.state.currentUserIsResizingBox
  const isCurrent = store.state.currentUserIsResizingBoxIds.includes(props.box.id)
  return isResizing && isCurrent
})
const startBoxInfoInteraction = (event) => {
  if (event.target.closest('.connector')) {
    return
  }
  if (!currentBoxIsSelected.value) {
    store.dispatch('clearMultipleSelected')
  }
  store.commit('currentDraggingBoxId', '')
  store.dispatch('closeAllDialogs')
  store.commit('currentUserIsDraggingBox', true)
  store.commit('currentDraggingBoxId', props.box.id)
  store.dispatch('currentBoxes/incrementZ', props.box.id)
  const updates = {
    boxId: props.box.id,
    userId: store.state.currentUser.id
  }
  store.commit('broadcast/updateStore', { updates, type: 'addToRemoteBoxesDragging' })
  if (event.altKey) { return } // should not select contained items if alt/option key
  selectContainedCards()
  selectContainedBoxes()
}
const updateIsHover = (value) => {
  if (store.state.currentUserIsDraggingBox) { return }
  if (isPainting.value) { return }
  state.isHover = value
  if (value) {
    store.commit('currentUserIsHoveringOverBoxId', props.box.id)
    updateCurrentConnections()
  } else {
    store.commit('currentUserIsHoveringOverBoxId', '')
  }
}
const endBoxInfoInteraction = (event) => {
  if (isConnectingTo.value) { return }
  const isMeta = event.metaKey || event.ctrlKey
  const userId = store.state.currentUser.id
  store.dispatch('currentBoxes/afterMove')
  store.dispatch('currentCards/afterMove')
  if (store.state.currentUserIsPainting) { return }
  if (isMultiTouch) { return }
  if (store.state.currentUserIsPanningReady || store.state.currentUserIsPanning) { return }
  if (!canEditBox.value) { store.commit('triggerReadOnlyJiggle') }
  store.commit('broadcast/updateStore', { updates: { userId }, type: 'clearRemoteBoxesDragging' })
  store.dispatch('closeAllDialogs')
  if (isMeta) {
    store.dispatch('multipleBoxesSelectedIds', [])
  } else {
    store.dispatch('clearMultipleSelected')
  }
  if (store.state.preventDraggedBoxFromShowingDetails) { return }
  if (isMeta) { return }
  store.commit('boxDetailsIsVisibleForBoxId', props.box.id)
  event.stopPropagation() // prevent stopInteractions() from closing boxDetails
  store.commit('currentUserIsDraggingBox', false)
  store.commit('boxesWereDragged', false)
}
const currentBoxDetailsIsVisible = computed(() => {
  return props.box.id === store.state.boxDetailsIsVisibleForBoxId
})

// select

const multipleBoxesIsSelected = computed(() => Boolean(store.state.multipleBoxesSelectedIds.length))
const currentBoxIsSelected = computed(() => {
  const selected = store.state.multipleBoxesSelectedIds
  return selected.find(id => props.box.id === id)
})
const selectedBoxes = computed(() => store.getters['currentBoxes/isSelected'])
const containedItems = () => {
  const cards = []
  const boxes = []
  // cards
  selectableCards().forEach(card => {
    if (isItemInSelectedBoxes(card, 'card')) {
      cards.push(card)
    }
  })
  // boxes
  let selectableBoxes = store.getters['currentBoxes/all']
  selectableBoxes = utils.clone(selectableBoxes)
  selectableBoxes.forEach(box => {
    if (box.id === props.box.id) { return }
    box.width = box.resizeWidth
    box.height = box.resizeHeight
    if (isItemInSelectedBoxes(box)) {
      boxes.push(box)
    }
  })
  return { cards, boxes }
}
const selectContainedBoxes = () => {
  const boxes = containedItems().boxes
  boxes.forEach(box => {
    store.dispatch('addToMultipleBoxesSelected', box.id)
  })
}
const selectableCards = () => {
  store.dispatch('currentCards/updateCanBeSelectedSortedByY')
  return store.getters['currentCards/canBeSelectedSortedByY'].cards
}
const selectContainedCards = () => {
  const cards = containedItems().cards
  cards.forEach(card => {
    store.dispatch('addToMultipleCardsSelected', card.id)
  })
  if (!multipleBoxesIsSelected.value) {
    store.commit('preventMultipleSelectedActionsIsVisible', true)
  }
}
const isItemInSelectedBoxes = (item, type) => {
  if (type === 'card') {
    const canEditCard = store.getters['currentUser/canEditCard'](item)
    if (!canEditCard) { return }
  }
  if (item.isLocked) { return }
  let boxes = selectedBoxes.value
  if (prevSelectedBox) {
    boxes = [prevSelectedBox]
  }
  const isInside = boxes.find(box => {
    box = normalizeBox(box)
    const { x, y } = box
    const width = box.resizeWidth
    const height = box.resizeHeight
    // ┌─────────────────────────────────────┐
    // │ Box                                 │
    // │                                     │
    // │                                     │
    // │                                     │
    // │      x1 = x          x2 = x + w     │
    // │         ██───────────────██         │
    // │         │                 │         │
    // │         │      Item       │         │
    // │         │                 │         │
    // │         ██───────────────██         │
    // │      y1 = y          y2 = y + h     │
    // │                                     │
    // │                                     │
    // │                                     │
    // │                                     │
    // └─────────────────────────────────────┘
    const x1 = utils.isBetween({
      value: item.x,
      min: x,
      max: x + width
    })
    const x2 = utils.isBetween({
      value: item.x + item.width,
      min: x,
      max: x + width
    })
    const y1 = utils.isBetween({
      value: item.y,
      min: y,
      max: y + height
    })
    const y2 = utils.isBetween({
      value: item.y + item.height,
      min: y,
      max: y + height
    })
    return x1 && x2 && y1 && y2
  })
  return isInside
}

// Remote

const isRemoteSelected = computed(() => {
  const remoteBoxesSelected = store.state.remoteBoxesSelected
  const selectedBox = remoteBoxesSelected.find(box => box.boxId === props.box.id)
  return Boolean(selectedBox)
})
const isRemoteBoxDetailsVisible = computed(() => {
  const remoteBoxDetailsVisible = store.state.remoteBoxDetailsVisible
  const visibleBox = remoteBoxDetailsVisible.find(box => box.boxId === props.box.id)
  return Boolean(visibleBox)
})
const remoteBoxDetailsVisibleColor = computed(() => {
  const remoteBoxDetailsVisible = store.state.remoteBoxDetailsVisible
  const visibleBox = remoteBoxDetailsVisible.find(box => box.boxId === props.box.id)
  if (visibleBox) {
    const user = store.getters['currentSpace/userById'](visibleBox.userId)
    return user.color
  } else {
    return undefined
  }
})
const isRemoteBoxDragging = computed(() => {
  const remoteBoxesDragging = store.state.remoteBoxesDragging
  const isDragging = remoteBoxesDragging.find(box => box.boxId === props.box.id)
  return Boolean(isDragging)
})
const remoteSelectedColor = computed(() => {
  const remoteBoxesSelected = store.state.remoteBoxesSelected
  const selectedBox = remoteBoxesSelected.find(box => box.boxId === props.box.id)
  if (selectedBox) {
    const user = store.getters['currentSpace/userById'](selectedBox.userId)
    return user.color
  } else {
    return undefined
  }
})
const remoteUserResizingBoxesColor = computed(() => {
  const remoteUserResizingBoxes = store.state.remoteUserResizingBoxes
  if (!remoteUserResizingBoxes.length) { return }
  let user = remoteUserResizingBoxes.find(user => user.boxIds.includes(props.box.id))
  if (user) {
    user = store.getters['currentSpace/userById'](user.userId)
    return user.color
  } else {
    return undefined
  }
})
const remoteBoxDraggingColor = computed(() => {
  const remoteBoxesDragging = store.state.remoteBoxesDragging
  const draggingBox = remoteBoxesDragging.find(box => box.boxId === props.box.id)
  if (draggingBox) {
    const user = store.getters['currentSpace/userById'](draggingBox.userId)
    return user.color
  } else {
    return undefined
  }
})

// touch locking

const lockingFrameStyle = computed(() => {
  const initialPadding = 65 // matches initialLockCircleRadius in paintSelect
  const initialBorderRadius = 50
  const padding = initialPadding * state.lockingPercent
  const borderRadius = Math.max((state.lockingPercent * initialBorderRadius), 5) + 'px'
  const size = `calc(100% + ${padding}px)`
  const position = -(padding / 2) + 'px'
  return {
    width: size,
    height: size,
    left: position,
    top: position,
    background: userColor.value,
    opacity: state.lockingAlpha,
    borderRadius
  }
})
const cancelLocking = () => {
  shouldCancelLocking = true
}
const cancelLockingAnimationFrame = () => {
  state.isLocking = false
  state.lockingPercent = 0
  state.lockingAlpha = 0
  shouldCancelLocking = false
}
const startLocking = (event) => {
  console.info('startLocking', event)
  updateTouchPosition(event)
  updateCurrentTouchPosition(event)
  state.isLocking = true
  shouldCancelLocking = false
  setTimeout(() => {
    if (!lockingAnimationTimer) {
      lockingAnimationTimer = window.requestAnimationFrame(lockingAnimationFrame)
    }
  }, lockingPreDuration)
}
const lockingAnimationFrame = (timestamp) => {
  if (!lockingStartTime) {
    lockingStartTime = timestamp
  }
  const elaspedTime = timestamp - lockingStartTime
  const percentComplete = (elaspedTime / lockingDuration) // between 0 and 1
  if (!utils.cursorsAreClose(touchPosition, currentTouchPosition)) {
    notifyPressAndHoldToDrag()
    cancelLockingAnimationFrame()
  }
  if (shouldCancelLocking) {
    cancelLockingAnimationFrame()
  }
  if (state.isLocking && percentComplete <= 1) {
    const percentRemaining = Math.abs(percentComplete - 1)
    state.lockingPercent = percentRemaining
    const alpha = utils.easeOut(percentComplete, elaspedTime, lockingDuration)
    state.lockingAlpha = alpha
    window.requestAnimationFrame(lockingAnimationFrame)
  } else if (state.isLocking && percentComplete > 1) {
    console.info('🔒🐢 box lockingAnimationFrame locked')
    lockingAnimationTimer = undefined
    lockingStartTime = undefined
    state.isLocking = false
    startBoxInfoInteraction(initialTouchEvent)
  } else {
    window.cancelAnimationFrame(lockingAnimationTimer)
    lockingAnimationTimer = undefined
    lockingStartTime = undefined
    cancelLockingAnimationFrame()
  }
}
const notifyPressAndHoldToDrag = () => {
  const hasNotified = store.state.hasNotifiedPressAndHoldToDrag
  if (!hasNotified) {
    store.commit('addNotification', { message: 'Press and hold to drag', icon: 'press-and-hold' })
  }
  store.commit('hasNotifiedPressAndHoldToDrag', true)
}
const updateTouchPosition = (event) => {
  initialTouchEvent = event
  isMultiTouch = false
  if (utils.isMultiTouch(event)) {
    isMultiTouch = true
    return
  }
  touchPosition = utils.cursorPositionInViewport(event)
}
const updateCurrentTouchPosition = (event) => {
  currentTouchPosition = utils.cursorPositionInViewport(event)
  if (currentBoxIsBeingDragged.value || isResizing.value) {
    event.preventDefault() // allows dragging boxes without scrolling
  }
}
const touchIsNearTouchPosition = (event) => {
  const currentPosition = utils.cursorPositionInViewport(event)
  const touchBlur = 12
  const isTouchX = utils.isBetween({
    value: currentPosition.x,
    min: touchPosition.x - touchBlur,
    max: touchPosition.x + touchBlur
  })
  const isTouchY = utils.isBetween({
    value: currentPosition.y,
    min: touchPosition.y - touchBlur,
    max: touchPosition.y + touchBlur
  })
  if (isTouchX && isTouchY) {
    return true
  }
}
const endBoxInfoInteractionTouch = (event) => {
  cancelLocking()
  if (touchIsNearTouchPosition(event)) {
    endBoxInfoInteraction(event)
  }
}

// connections

const isConnectingTo = computed(() => {
  const connectingToId = store.state.currentConnectionSuccess.id
  const isConnecting = connectingToId === props.box.id
  if (isConnecting) {
    postMessage.sendHaptics({ name: 'softImpact' })
  }
  return isConnecting
})
const isConnectingFrom = computed(() => {
  return store.state.currentConnectionStartItemIds.includes(props.box.id)
})
const connectedConnectionTypes = computed(() => store.getters['currentConnections/typesByItemId'](props.box.id))
const connectorIsVisible = computed(() => {
  const spaceIsOpen = store.state.currentSpace.privacy === 'open' && currentUserIsSignedIn.value
  let isVisible
  if (isLocked.value) { return }
  if (state.isRemoteConnecting) {
    isVisible = true
  } else if (spaceIsOpen || canEditBox.value || connectedConnectionTypes.value.length) {
    isVisible = true
  }
  return isVisible
})
const connectorIsHiddenByOpacity = computed(() => {
  if (utils.isMobile()) { return }
  const isPresentationMode = store.state.isPresentationMode
  const isNotHovering = !state.isHover
  const isNotConnected = !isConnectingFrom.value && !isConnectingTo.value && !state.currentConnections.length
  return isPresentationMode && isNotHovering && isNotConnected
})

const updateRemoteConnections = () => {
  const connection = store.state.remoteCurrentConnections.find(remoteConnection => {
    const isConnectedToStart = remoteConnection.startItemId === props.box.id
    const isConnectedToEnd = remoteConnection.endItemId === props.box.id
    return isConnectedToStart || isConnectedToEnd
  })
  if (connection) {
    state.isRemoteConnecting = true
    state.remoteConnectionColor = connection.color
  } else {
    state.isRemoteConnecting = false
  }
}

// checkbox

const isChecked = computed(() => utils.nameIsChecked(name.value))
const hasCheckbox = computed(() => {
  return utils.checkboxFromString(name.value)
})
const checkboxState = computed({
  get () {
    return isChecked.value
  }
})
const toggleBoxChecked = () => {
  if (store.state.preventDraggedBoxFromShowingDetails) { return }
  if (!canEditBox.value) { return }
  const value = !isChecked.value
  store.dispatch('closeAllDialogs')
  store.dispatch('currentBoxes/toggleChecked', { boxId: props.box.id, value })
  postMessage.sendHaptics({ name: 'heavyImpact' })
  cancelLocking()
  store.commit('currentUserIsDraggingBox', false)
  const userId = store.state.currentUser.id
  store.commit('broadcast/updateStore', { updates: { userId }, type: 'clearRemoteBoxesDragging' })
  event.stopPropagation()
  store.commit('preventMultipleSelectedActionsIsVisible', false)
  store.dispatch('clearMultipleSelected')
  store.commit('currentDraggingBoxId', '')
  store.dispatch('multipleBoxesSelectedIds', [])
}
const containingBoxes = computed(() => {
  if (!state.isVisibleInViewport) { return }
  if (store.state.currentUserIsDraggingBox) { return }
  if (currentBoxIsBeingDragged.value) { return }
  if (currentBoxIsSelected.value) { return }
  if (isResizing.value) { return }
  if (store.state.boxDetailsIsVisibleForBoxId) { return }
  let boxes = store.getters['currentBoxes/all']
  boxes = utils.clone(boxes)
  boxes = boxes.filter(box => {
    const currentBox = utils.clone(props.box)
    const isInsideBox = utils.isRectACompletelyInsideRectB(currentBox, box)
    const boxArea = box.resizeWidth * box.resizeHeight
    const currentBoxArea = currentBox.resizeWidth * currentBox.resizeHeight
    const boxIsParent = boxArea > currentBoxArea
    return isInsideBox && boxIsParent
  })
  return boxes
})
const isInCheckedBox = computed(() => {
  if (!containingBoxes.value) { return }
  const checkedBox = containingBoxes.value.find(box => utils.nameIsChecked(box.name))
  return Boolean(checkedBox)
})

// box focus

const isFocusing = computed(() => props.box.id === store.state.focusOnBoxId)
const focusColor = computed(() => {
  if (isFocusing.value) {
    return currentUserColor.value
  } else {
    return null
  }
})
</script>

<template lang="pug">
.box(
  :key="box.id"
  :data-box-id="box.id"
  :data-x="normalizedBox.x"
  :data-y="normalizedBox.y"
  :data-resize-width="normalizedBox.resizeWidth"
  :data-resize-height="normalizedBox.resizeHeight"
  :data-info-width="normalizedBox.infoWidth"
  :data-info-height="normalizedBox.infoHeight"
  :data-background="normalizedBox.background"
  :data-is-locked="isLocked"
  :data-is-visible-in-viewport="state.isVisibleInViewport"
  :data-should-render="shouldRender"

  :style="styles"
  :class="classes"
  ref="boxElement"
)
  .focusing-frame(v-if="isFocusing" :style="{backgroundColor: currentUserColor}")
  teleport(to="#box-backgrounds")
    .box-background(v-if="box.background && state.isVisibleInViewport" :data-box-id="box.id" :style="backgroundStyles")
  //- name
  .box-info(
    v-if="shouldRender"
    :data-box-id="box.id"
    :data-is-visible-in-viewport="state.isVisibleInViewport"
    :style="infoStyles"
    :class="infoClasses"
    tabindex="0"

    @mouseover="updateIsHover(true)"
    @mouseleave="updateIsHover(false)"
    @mousedown.left="startBoxInfoInteraction"

    @mouseup.left="endBoxInfoInteraction"
    @keyup.stop.enter="endBoxInfoInteraction"

    @touchstart="startLocking"
    @touchmove="updateCurrentTouchPosition"
    @touchend="endBoxInfoInteractionTouch"
  )
    .locking-frame(v-if="state.isLocking" :style="lockingFrameStyle")
    //- [·]
    .checkbox-wrap(v-if="hasCheckbox" @mouseup.left="toggleBoxChecked" @touchend.prevent="toggleBoxChecked")
      label(:class="{active: isChecked, disabled: !canEditBox}")
        input(name="checkbox" type="checkbox" v-model="checkboxState")
    .name-wrap(:class="{'is-checked': isChecked}")
      //- simplified nameSegments
      template(v-for="segment in nameSegments")
        template(v-if="segment.type === 'text'")
          span {{smartQuotes(segment.content)}}
        template(v-else-if="segment.type === 'bold'")
          strong {{smartQuotes(segment.content)}}
        template(v-else-if="segment.type === 'h1'")
          h1 {{smartQuotes(segment.content)}}
        template(v-else-if="segment.type === 'h2'")
          h2 {{smartQuotes(segment.content)}}
        template(v-else-if="segment.type === 'h3'")
          h3 {{smartQuotes(segment.content)}}
        template(v-else-if="segment.type === 'h4'")
          h4 {{segment.content}}
        template(v-else-if="segment.type === 'emphasis'")
          em {{smartQuotes(segment.content)}}
        template(v-else-if="segment.type === 'strikethrough'")
          del {{smartQuotes(segment.content)}}
      .selected-user-avatar(v-if="isRemoteSelected || isRemoteBoxDetailsVisible" :style="{backgroundColor: remoteSelectedColor || remoteBoxDetailsVisibleColor}")
        img(src="@/assets/anon-avatar.svg")

    ItemConnectorButton(
      :visible="connectorIsVisible"
      :isHiddenByOpacity="connectorIsHiddenByOpacity"
      :box="box"
      :itemConnections="state.currentConnections"
      :isConnectingTo="isConnectingTo"
      :isConnectingFrom="isConnectingFrom"
      :isVisibleInViewport="state.isVisibleInViewport"
      :isRemoteConnecting="state.isRemoteConnecting"
      :remoteConnectionColor="state.remoteConnectionColor"
      :currentBackgroundColor="color"
      :backgroundIsTransparent="true"
      :parentDetailsIsVisible="currentBoxDetailsIsVisible"
      @shouldRenderParent="updateShouldRenderParent"
    )

  //- resize
  .bottom-button-wrap(v-if="resizeIsVisible" :class="{unselectable: isPainting}")
    .inline-button-wrap(
        @pointerover="updateIsHover(true)"
        @pointerleave="updateIsHover(false)"
        @mousedown.left="startResizing"
        @touchstart="startResizing"
        @dblclick="shrink"
      )
      button.inline-button(
        tabindex="-1"
      )
        img.resize-icon.icon(src="@/assets/resize-corner.svg" :class="resizeColorClass")
</template>

<style lang="stylus">
.box
  --min-box-size 70px
  --ease-out-circ cubic-bezier(0, 0.55, 0.45, 1) // https://easings.net/#easeOutCirc
  position absolute
  border-radius var(--entity-radius)
  min-height var(--min-box-size)
  min-width var(--min-box-size)
  // pointer-events none
  // animate box expand and shrink
  &.transition
    transition width 0.2s var(--ease-out-circ),
      height 0.2s var(--ease-out-circ),
      left 0.2s var(--ease-out-circ),
      top 0.2s var(--ease-out-circ)
  &.hover
    box-shadow var(--hover-shadow)
  &.active
    box-shadow var(--active-shadow)
    transition none
    z-index 1
  &.is-resizing
    box-shadow var(--active-shadow)
    transition none
  &.is-selected
    transition none
  &.is-checked
    opacity var(--is-checked-opacity)
  .box-info
    --header-font var(--header-font-0)
    z-index 1
    border-radius 4px
    display flex
    align-items center
    &.header-font-1
      --header-font var(--header-font-1)
    &.header-font-2
      --header-font var(--header-font-2)
    &.header-font-3
      --header-font var(--header-font-3)
    &.header-font-4
      --header-font var(--header-font-4)
    &.header-font-5
      --header-font var(--header-font-5)
    &.header-font-6
      --header-font var(--header-font-6)
    &.header-font-7
      --header-font var(--header-font-7)
    &.header-font-8
      --header-font var(--header-font-8)
    &.header-font-9
      --header-font var(--header-font-9)
    &.header-font-size-modifier-s
      h1
        font-size 18px
      h2
        font-size 16px
    &.header-font-size-m
      h1
        font-size 44px
      h2
        font-size 36px
      h3
        font-size 24px
    &.header-font-size-l
      h1
        font-size 66px
      h2
        font-size 52px
      h3
        font-size 36px
    pointer-events all
    position absolute
    cursor pointer
    border-bottom-right-radius var(--entity-radius)
    word-break break-word
    color var(--primary-on-light-background)
    &:hover
      box-shadow var(--hover-shadow)
    &:active
      box-shadow var(--active-shadow)
    &.is-dark
      color var(--primary-on-dark-background)

  .checkbox-wrap
    padding-left 8px
    padding-top 5px
    padding-bottom 6px
    display inline-block
    label
      pointer-events none
      width 20px
      height 16px
      display flex
      align-items center
      padding-left 4px
      padding-right 4px
      input
        margin 0
        margin-top -1px
        width 10px
        height 10px
        background-size contain
    &:hover
      label
        box-shadow 3px 3px 0 var(--heavy-shadow)
        background-color var(--secondary-hover-background)
        input
          background-color var(--secondary-hover-background)
      label.active
        box-shadow var(--active-inset-shadow)
        background-color var(--secondary-active-background)
        input
          background-color var(--secondary-active-background)
    &:active
      label
        box-shadow none
        color var(--primary)
        background-color var(--secondary-active-background)
      input
        background-color var(--secondary-active-background)

  .name-wrap
    padding 6px 8px
    padding-top 4px
    // padding-right 4px
    display inline-block
    &.is-checked
      text-decoration line-through
  h1
    font-family var(--header-font)
    font-size 20px
    font-weight bold
    display inline-block
  h2
    font-family var(--header-font)
    font-weight normal
    font-size 20px
    display inline-block
  h1,
  h2,
  h3,
  h4
    margin 0
    margin-top 2px
    vertical-align sub // to align with checkboxes

  // resize
  .bottom-button-wrap
    .inline-button-wrap
      cursor nwse-resize
      button
        cursor nwse-resize
      .resize-icon
        top 0
        left 0

  .locking-frame
    position absolute
    z-index -1
    pointer-events none

  // same as Card.vue
  .selected-user-avatar
    padding 0 3px
    border-radius var(--small-entity-radius)
    position absolute
    top -5px
    left -5px
    pointer-events none
    z-index 1
    img
      width 10px
      height 10px

  .connector
    padding 8px
    padding-top 4px
    padding-bottom 3px
    padding-left 0
    align-self right
    cursor cell
    right 0
    pointer-events all
    position relative
    display inline-block
    button
      z-index 1
  .connector-glow
    left -9px
    top -5px
  .connected-colors
    left 1px
    top 5px
</style>
