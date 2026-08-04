<template>
  <main v-if="visible" class="radial-menu" :class="menuClasses" data-interface="radialMenu" @contextmenu.prevent>
    <section class="radial-menu__stage" :aria-label="title">
      <svg class="radial-menu__svg" viewBox="0 0 700 350" role="presentation">
        <defs>
          <radialGradient :id="centerGradientId" cx="50%" cy="50%" r="50%">
            <stop offset="0%" stop-color="#bd46c9" stop-opacity="0.96" />
            <stop offset="56%" stop-color="#7a0c86" stop-opacity="0.92" />
            <stop offset="100%" stop-color="#20002b" stop-opacity="0.98" />
          </radialGradient>

          <linearGradient :id="segmentGradientId" x1="0%" x2="100%" y1="0%" y2="100%">
            <stop offset="0%" stop-color="#e13dff" stop-opacity="0.96" />
            <stop offset="100%" stop-color="#7e008d" stop-opacity="0.96" />
          </linearGradient>

          <filter :id="glowFilterId" x="-46%" y="-46%" width="192%" height="192%">
            <feGaussianBlur stdDeviation="8" result="blur" />
            <feColorMatrix
              in="blur"
              result="glow"
              type="matrix"
              values="0 0 0 0 0.9 0 0 0 0 0.12 0 0 0 0 1 0 0 0 0.52 0"
            />
            <feMerge>
              <feMergeNode in="glow" />
              <feMergeNode in="SourceGraphic" />
            </feMerge>
          </filter>
        </defs>

        <g
          v-if="currentActionLayer"
          :key="`segments-${currentActionLayer.key}`"
          class="radial-menu__action-segments"
          :class="getActionSegmentsClass(currentActionLayer)"
        >
          <template v-for="(item, index) in currentActionLayer.items" :key="`${currentActionLayer.key}-${item.id}`">
            <path
              class="radial-menu__segment-base radial-menu__segment-base--outer"
              :class="{ 'radial-menu__segment-base--disabled': item.disabled }"
              :d="getSegmentPath(index, currentActionLayer.items.length, outerRing)"
              :style="getSequentialStyle(index, currentActionLayer.items.length)"
            />
            <path
              class="radial-menu__segment-highlight radial-menu__segment-highlight--outer"
              :class="getActionHighlightClass(currentActionLayer, item)"
              :d="getSegmentPath(index, currentActionLayer.items.length, outerRing)"
            />
            <path
              class="radial-menu__segment-hit"
              :class="{ 'radial-menu__segment-hit--disabled': item.disabled }"
              :d="getSegmentPath(index, currentActionLayer.items.length, outerRing)"
              @click="selectActionFromLayer(currentActionLayer, item)"
              @pointerdown="pressAction(currentActionLayer, item)"
              @pointerup="clearPressedAction"
              @pointercancel="clearPressedAction"
              @mouseenter="hoverAction(currentActionLayer, item)"
              @mouseleave="leaveAction(item)"
            />
          </template>
        </g>

        <g class="radial-menu__main-segments">
          <template v-for="(group, index) in groups" :key="group.id">
            <path
              class="radial-menu__segment-base radial-menu__segment-base--inner"
              :class="{ 'radial-menu__segment-base--disabled': group.disabled }"
              :d="getSegmentPath(index, groups.length, innerRing)"
              :style="getSequentialStyle(index, groups.length)"
            />
            <path
              class="radial-menu__segment-highlight radial-menu__segment-highlight--inner"
              :class="getMainHighlightClass(group)"
              :d="getSegmentPath(index, groups.length, innerRing)"
            />
            <path
              class="radial-menu__segment-hit"
              :class="{ 'radial-menu__segment-hit--disabled': group.disabled }"
              :d="getSegmentPath(index, groups.length, innerRing)"
              @click="selectGroup(group)"
              @mouseenter="hoveredGroupId = group.id"
              @mouseleave="hoveredGroupId = null"
            />
          </template>
        </g>

        <circle class="radial-menu__center-orb" :cx="stage.centerX" :cy="stage.centerY" :r="centerRadius" />
      </svg>

      <div
        v-for="layer in actionLayers"
        :key="`items-${layer.key}`"
        class="radial-menu__action-layer radial-menu__action-layer--items"
        :class="getActionLayerClass(layer)"
      >
        <button
          v-for="(item, index) in layer.items"
          :key="`action-${layer.key}-${item.id}`"
          class="radial-menu__item radial-menu__item--outer"
          :class="{
            'radial-menu__item--active': isActionLayerInteractive(layer) && pressedActionId === item.id,
            'radial-menu__item--hovered': isActionLayerInteractive(layer) && hoveredActionId === item.id && pressedActionId !== item.id,
            'radial-menu__item--disabled': item.disabled,
          }"
          :style="getItemStyle(index, layer.items.length, outerRing.labelRadius)"
          type="button"
          :aria-disabled="item.disabled ? 'true' : 'false'"
          :title="item.disabledReason || item.description || item.label"
          @click="selectActionFromLayer(layer, item)"
          @pointerdown="pressAction(layer, item)"
          @pointerup="clearPressedAction"
          @pointercancel="clearPressedAction"
          @mouseenter="hoverAction(layer, item)"
          @mouseleave="leaveAction(item)"
        >
          <span class="radial-menu__item-content">
            <component :is="iconComponent(item.icon)" class="radial-menu__icon" :size="25" :stroke-width="2.2" />
            <span class="radial-menu__label">{{ item.label }}</span>
          </span>
        </button>
      </div>

      <div class="radial-menu__main-items">
        <button
          v-for="(group, index) in groups"
          :key="`group-${group.id}`"
          class="radial-menu__item radial-menu__item--inner"
          :class="{
            'radial-menu__item--active': selectedGroupId === group.id,
            'radial-menu__item--hovered': hoveredGroupId === group.id && selectedGroupId !== group.id,
            'radial-menu__item--disabled': group.disabled,
          }"
          :style="getItemStyle(index, groups.length, innerRing.labelRadius)"
          type="button"
          :disabled="group.disabled"
          :title="group.disabledReason || group.description || group.label"
          @click="selectGroup(group)"
          @mouseenter="hoveredGroupId = group.id"
          @mouseleave="hoveredGroupId = null"
        >
          <span class="radial-menu__item-content">
            <component :is="iconComponent(group.icon)" class="radial-menu__icon" :size="25" :stroke-width="2.2" />
            <span class="radial-menu__label">{{ group.label }}</span>
          </span>
        </button>
      </div>

      <button class="radial-menu__center" type="button" @click="closeMenu">
        <span>{{ centerHint[0] }}</span>
        <span>{{ centerHint[1] }}</span>
      </button>
    </section>
  </main>

  <button v-else class="radial-menu__reopen" type="button" @click="() => openMenu()">Открыть меню</button>
</template>

<script setup lang="ts">
import type { Component } from 'vue'
import { computed, onBeforeUnmount, onMounted, ref, watch } from 'vue'
import {
  Armchair,
  Badge,
  Banknote,
  BriefcaseBusiness,
  Building2,
  Car,
  Circle,
  CirclePower,
  DoorClosed,
  DoorOpen,
  Drama,
  FileText,
  FileUser,
  Footprints,
  Hand,
  HandCoins,
  Handshake,
  House,
  IdCard,
  KeyRound,
  LockOpen,
  PackageOpen,
  ScanSearch,
  Shield,
  UserRound,
  VenetianMask,
  Wallet,
  Wrench,
} from 'lucide-vue-next'

interface Point {
  x: number
  y: number
}

interface RingGeometry {
  innerRadius: number
  outerRadius: number
  labelRadius: number
}

interface RadialMenuAction {
  id: string
  label: string
  icon?: string
  description?: string
  disabled?: boolean
  disabledReason?: string
  hidden?: boolean
  closeOnSelect?: boolean
}

interface RadialMenuGroup {
  id: string
  label: string
  icon?: string
  description?: string
  disabled?: boolean
  disabledReason?: string
  hidden?: boolean
  items: RadialMenuAction[]
}

interface RadialMenuPayload {
  title?: string
  hint?: string
  visible?: boolean
  groups?: RadialMenuGroup[]
}

type ActionLayerMode = 'pop' | 'active' | 'switch-enter' | 'switch-leave'

interface ActionLayer {
  groupId: string
  items: RadialMenuAction[]
  key: string
  mode: ActionLayerMode
}

const stage = {
  width: 700,
  height: 350,
  centerX: 350,
  centerY: 350,
}

const arcStart = 180
const arcEnd = 360
const segmentOverlapDegrees = 0.5
const centerRadius = 100
const actionLayerStaggerMs = 54
const closeLayerStaggerMs = 27
const actionSwitchLeaveMs = 87
const actionSwitchEnterMs = 150
const actionSwitchMs = actionSwitchLeaveMs + actionSwitchEnterMs
const closeAnimationMs = 820
const closeAnimationWithoutActionsMs = 520
const centerGradientId = 'radialCenterGradient'
const segmentGradientId = 'radialSegmentGradient'
const glowFilterId = 'radialPurpleGlow'
const hoveredActionId = ref<string | null>(null)
const hoveredGroupId = ref<string | null>(null)
const payloadState = ref<RadialMenuPayload>({})
const selectedGroupId = ref<string | null>(null)
const pressedActionId = ref<string | null>(null)
const currentActionLayer = ref<ActionLayer | null>(null)
const leavingActionLayers = ref<ActionLayer[]>([])
const actionLeaveTimers: number[] = []
const closeTimers: number[] = []
const isClosed = ref(false)
const isClosing = ref(false)

const outerRing: RingGeometry = {
  innerRadius: 250,
  outerRadius: 350,
  labelRadius: 302,
}

const innerRing: RingGeometry = {
  innerRadius: 112,
  outerRadius: 225,
  labelRadius: 170,
}

const iconComponents: Record<string, Component> = {
  armchair: Armchair,
  badge: Badge,
  banknote: Banknote,
  briefcase: BriefcaseBusiness,
  building: Building2,
  car: Car,
  'circle-power': CirclePower,
  'door-closed': DoorClosed,
  'door-open': DoorOpen,
  drama: Drama,
  'file-text': FileText,
  'file-user': FileUser,
  footprints: Footprints,
  hand: Hand,
  'hand-coins': HandCoins,
  handshake: Handshake,
  house: House,
  'id-card': IdCard,
  key: KeyRound,
  'lock-open': LockOpen,
  'package-open': PackageOpen,
  'scan-search': ScanSearch,
  shield: Shield,
  user: UserRound,
  'venetian-mask': VenetianMask,
  wallet: Wallet,
  wrench: Wrench,
}

const defaultGroups: RadialMenuGroup[] = [
  {
    id: 'vehicle',
    label: 'Авто',
    icon: 'car',
    items: [
      { id: 'vehicle:door:toggle', label: 'Дверь', icon: 'door-open' },
      { id: 'vehicle:lock:toggle', label: 'Замок', icon: 'lock-open' },
      { id: 'vehicle:engine:toggle', label: 'Двигатель', icon: 'circle-power' },
      { id: 'vehicle:seat:0', label: 'Место 1', icon: 'armchair' },
      { id: 'vehicle:seat:1', label: 'Место 2', icon: 'armchair' },
      { id: 'vehicle:trunk:toggle', label: 'Багажник', icon: 'package-open' },
      { id: 'vehicle:repair', label: 'Ремонт', icon: 'wrench' },
    ],
  },
  {
    id: 'player',
    label: 'Игрок',
    icon: 'user',
    items: [
      { id: 'player:passport', label: 'Паспорт', icon: 'id-card' },
      { id: 'player:cash:1000', label: '$ 1000', icon: 'hand-coins' },
      { id: 'player:greet', label: 'Привет', icon: 'handshake' },
      { id: 'player:search', label: 'Обыскать', icon: 'scan-search' },
    ],
  },
  {
    id: 'documents',
    label: 'Документы',
    icon: 'id-card',
    items: [
      { id: 'docs:passport', label: 'Паспорт', icon: 'id-card' },
      { id: 'docs:vehicle', label: 'На авто', icon: 'file-text' },
      { id: 'docs:military', label: 'Военный', icon: 'file-user' },
    ],
  },
  {
    id: 'property',
    label: 'Имущество',
    icon: 'house',
    items: [
      { id: 'property:house:register', label: 'Дом', icon: 'house' },
      { id: 'property:apartment:register', label: 'Квартира', icon: 'building' },
      { id: 'property:business:sell', label: 'Бизнес', icon: 'briefcase' },
      { id: 'property:vehicle:sell', label: 'Авто', icon: 'car' },
    ],
  },
  {
    id: 'crime',
    label: 'Крайм',
    icon: 'venetian-mask',
    items: [
      { id: 'crime:invite', label: 'Группировка', icon: 'shield' },
      { id: 'crime:rob', label: 'Ограбить', icon: 'wallet' },
      { id: 'crime:ties', label: 'Стяжки', icon: 'badge' },
      { id: 'crime:hostage', label: 'Заложник', icon: 'user' },
      { id: 'crime:bag', label: 'Мешок', icon: 'package-open' },
    ],
  },
  {
    id: 'animations',
    label: 'Анимации',
    icon: 'drama',
    items: [
      { id: 'animation:greet', label: 'Привет', icon: 'handshake' },
      { id: 'animation:stop', label: 'Стоп', icon: 'hand' },
    ],
  },
]

const groups = computed(() => {
  const source = payloadState.value.groups?.length ? payloadState.value.groups : defaultGroups

  return source
    .filter((group) => !group.hidden)
    .map((group) => ({
      ...group,
      icon: group.icon || 'circle',
      items: group.items.filter((item) => !item.hidden),
    }))
    .filter((group) => group.items.length > 0)
})

const actionLayers = computed(() => [
  ...leavingActionLayers.value,
  ...(currentActionLayer.value ? [currentActionLayer.value] : []),
])
const visible = computed(() => !isClosed.value && payloadState.value.visible !== false && groups.value.length > 0)
const title = computed(() => payloadState.value.title || 'Радиальное меню')
const menuClasses = computed(() => ({
  'radial-menu--closing': isClosing.value,
  'radial-menu--has-actions': Boolean(currentActionLayer.value),
}))
const centerHint = computed(() => {
  const hint = payloadState.value.hint || 'Нажмите, чтобы закрыть'
  const parts = hint.split(/\s+/)

  if (parts.length <= 2) {
    return [hint, '']
  }

  return [parts.slice(0, 1).join(' '), parts.slice(1).join(' ')]
})

watch(groups, (nextGroups) => {
  if (!selectedGroupId.value) {
    return
  }

  const nextGroup = nextGroups.find((group) => group.id === selectedGroupId.value)

  if (!nextGroup) {
    resetActionLayers()
    return
  }

  if (currentActionLayer.value) {
    currentActionLayer.value = {
      ...currentActionLayer.value,
      items: nextGroup.items,
    }
  }
})

function parsePayload(payload?: RadialMenuPayload | string | null): RadialMenuPayload {
  if (!payload) {
    return {}
  }

  if (typeof payload === 'object') {
    return payload
  }

  try {
    return JSON.parse(payload) as RadialMenuPayload
  } catch {
    return {}
  }
}

function openMenu(payload?: RadialMenuPayload | string | null) {
  closeTimers.splice(0).forEach((timer) => window.clearTimeout(timer))
  isClosed.value = false
  isClosing.value = false
  payloadState.value = payload ? parsePayload(payload) : {}
  resetActionLayers()
  hoveredGroupId.value = null
}

function resetActionLayers() {
  actionLeaveTimers.splice(0).forEach((timer) => window.clearTimeout(timer))
  leavingActionLayers.value = []
  currentActionLayer.value = null
  selectedGroupId.value = null
  hoveredActionId.value = null
  pressedActionId.value = null
}

function iconComponent(name?: string) {
  return iconComponents[name || ''] || Circle
}

function getSequentialStyle(index: number, total: number) {
  return {
    '--delay': `${index * actionLayerStaggerMs}ms`,
    '--reverse-delay': `${Math.max(total - 1 - index, 0) * actionLayerStaggerMs}ms`,
    '--close-reverse-delay': `${Math.max(total - 1 - index, 0) * closeLayerStaggerMs}ms`,
  }
}

function getActionLayerClass(layer: ActionLayer) {
  return {
    'radial-menu__action-layer--pop': layer.mode === 'pop',
    'radial-menu__action-layer--switch-enter': layer.mode === 'switch-enter',
    'radial-menu__action-layer--switch-leave': layer.mode === 'switch-leave',
  }
}

function getActionSegmentsClass(layer: ActionLayer) {
  return {
    'radial-menu__action-segments--pop': layer.mode === 'pop',
  }
}

function getMainHighlightClass(group: RadialMenuGroup) {
  const active = selectedGroupId.value === group.id

  return {
    'radial-menu__segment-highlight--active': active,
    'radial-menu__segment-highlight--hovered': !active && hoveredGroupId.value === group.id,
  }
}

function getActionHighlightClass(layer: ActionLayer, item: RadialMenuAction) {
  const interactive = isActionLayerInteractive(layer)
  const active = interactive && pressedActionId.value === item.id
  const hovered = interactive && !active && hoveredActionId.value === item.id

  return {
    'radial-menu__segment-highlight--active': active,
    'radial-menu__segment-highlight--hovered': hovered,
  }
}

function isActionLayerInteractive(layer: ActionLayer) {
  return (
    currentActionLayer.value?.key === layer.key &&
    layer.mode !== 'switch-leave' &&
    layer.mode !== 'switch-enter' &&
    !isClosing.value
  )
}

function getSegmentPath(index: number, total: number, geometry: RingGeometry) {
  if (total <= 0) {
    return ''
  }

  const step = (arcEnd - arcStart) / total
  const startAngle = arcStart + index * step - segmentOverlapDegrees
  const endAngle = arcStart + (index + 1) * step + segmentOverlapDegrees
  const outerStart = polarToCartesian(startAngle, geometry.outerRadius)
  const outerEnd = polarToCartesian(endAngle, geometry.outerRadius)
  const innerEnd = polarToCartesian(endAngle, geometry.innerRadius)
  const innerStart = polarToCartesian(startAngle, geometry.innerRadius)
  const largeArcFlag = endAngle - startAngle > 180 ? 1 : 0

  return [
    `M ${outerStart.x} ${outerStart.y}`,
    `A ${geometry.outerRadius} ${geometry.outerRadius} 0 ${largeArcFlag} 1 ${outerEnd.x} ${outerEnd.y}`,
    `L ${innerEnd.x} ${innerEnd.y}`,
    `A ${geometry.innerRadius} ${geometry.innerRadius} 0 ${largeArcFlag} 0 ${innerStart.x} ${innerStart.y}`,
    'Z',
  ].join(' ')
}

function getItemStyle(index: number, total: number, radius: number) {
  const point = getItemPosition(index, total, radius)
  const shift = getCenterShift(point, 28)

  return {
    '--delay': `${index * actionLayerStaggerMs}ms`,
    '--reverse-delay': `${Math.max(total - 1 - index, 0) * actionLayerStaggerMs}ms`,
    '--close-reverse-delay': `${Math.max(total - 1 - index, 0) * closeLayerStaggerMs}ms`,
    '--shift-x': `${shift.x}px`,
    '--shift-y': `${shift.y}px`,
    left: `${(point.x / stage.width) * 100}%`,
    top: `${(point.y / stage.height) * 100}%`,
  }
}

function getItemPosition(index: number, total: number, radius: number) {
  const step = (arcEnd - arcStart) / Math.max(total, 1)
  const angle = arcStart + index * step + step / 2

  return polarToCartesian(angle, radius)
}

function polarToCartesian(angleInDegrees: number, radius: number): Point {
  const angle = (angleInDegrees * Math.PI) / 180

  return {
    x: stage.centerX + radius * Math.cos(angle),
    y: stage.centerY + radius * Math.sin(angle),
  }
}

function getCenterShift(point: Point, distance: number): Point {
  const dx = stage.centerX - point.x
  const dy = stage.centerY - point.y
  const length = Math.hypot(dx, dy) || 1

  return {
    x: (dx / length) * distance,
    y: (dy / length) * distance,
  }
}

function createActionLayer(group: RadialMenuGroup, mode: ActionLayerMode): ActionLayer {
  return {
    groupId: group.id,
    items: group.items,
    key: `${group.id}-${mode}-${Date.now()}-${Math.round(Math.random() * 100000)}`,
    mode,
  }
}

function selectGroup(group: RadialMenuGroup) {
  if (isClosing.value || group.disabled || selectedGroupId.value === group.id) {
    return
  }

  actionLeaveTimers.splice(0).forEach((timer) => window.clearTimeout(timer))
  const previousLayer = currentActionLayer.value

  if (previousLayer) {
    const leavingLayer = {
      ...previousLayer,
      mode: 'switch-leave' as const,
    }

    leavingActionLayers.value = [leavingLayer]

    const timer = window.setTimeout(() => {
      leavingActionLayers.value = leavingActionLayers.value.filter((layer) => layer.key !== leavingLayer.key)
    }, actionSwitchLeaveMs + 40)

    actionLeaveTimers.push(timer)
  } else {
    leavingActionLayers.value = []
  }

  const nextLayer = createActionLayer(group, previousLayer ? 'switch-enter' : 'pop')

  currentActionLayer.value = nextLayer
  selectedGroupId.value = group.id
  hoveredActionId.value = null
  pressedActionId.value = null

  if (previousLayer) {
    const nextKey = nextLayer.key
    const timer = window.setTimeout(() => {
      currentActionLayer.value =
        currentActionLayer.value?.key === nextKey
          ? { ...currentActionLayer.value, mode: 'active' }
          : currentActionLayer.value
    }, actionSwitchMs + 40)

    actionLeaveTimers.push(timer)
  }
}

function pressAction(layer: ActionLayer, action: RadialMenuAction) {
  if (isClosing.value || !isActionLayerInteractive(layer)) {
    return
  }

  pressedActionId.value = action.id
}

function clearPressedAction() {
  pressedActionId.value = null
}

function hoverAction(layer: ActionLayer, action: RadialMenuAction) {
  if (isClosing.value || !isActionLayerInteractive(layer)) {
    return
  }

  hoveredActionId.value = action.id
}

function leaveAction(action: RadialMenuAction) {
  if (hoveredActionId.value === action.id) {
    hoveredActionId.value = null
  }

  if (pressedActionId.value === action.id) {
    pressedActionId.value = null
  }
}

function selectActionFromLayer(layer: ActionLayer, action: RadialMenuAction) {
  if (isClosing.value || !isActionLayerInteractive(layer)) {
    return
  }

  selectAction(action)
}

function selectAction(action: RadialMenuAction) {
  if (isClosing.value || action.disabled) {
    return
  }

  pressedActionId.value = action.id
  window.setTimeout(() => {
    if (pressedActionId.value === action.id) {
      pressedActionId.value = null
    }
  }, 160)

  if (action.closeOnSelect !== false) {
    window.setTimeout(closeMenu, 120)
  }
}

function closeMenu() {
  if (isClosing.value) {
    return
  }

  actionLeaveTimers.splice(0).forEach((timer) => window.clearTimeout(timer))
  leavingActionLayers.value = []
  hoveredActionId.value = null
  hoveredGroupId.value = null
  pressedActionId.value = null
  isClosing.value = true
  const closeDelay = currentActionLayer.value ? closeAnimationMs : closeAnimationWithoutActionsMs

  const timer = window.setTimeout(() => {
    isClosed.value = true
    isClosing.value = false
    resetActionLayers()
  }, closeDelay)

  closeTimers.push(timer)
}

function handleKeydown(event: KeyboardEvent) {
  if (event.key === 'Escape' || event.key.toLowerCase() === 'b') {
    closeMenu()
  }
}

onMounted(() => {
  window.addEventListener('keydown', handleKeydown)
})

onBeforeUnmount(() => {
  actionLeaveTimers.splice(0).forEach((timer) => window.clearTimeout(timer))
  closeTimers.splice(0).forEach((timer) => window.clearTimeout(timer))
  window.removeEventListener('keydown', handleKeydown)
})
</script>

<style scoped>
:global(*) {
  box-sizing: border-box;
}

:global(html),
:global(body),
:global(#app) {
  min-width: 100%;
  min-height: 100%;
  margin: 0;
}

:global(body) {
  overflow: hidden;
  background: #ffffff;
}

.radial-menu {
  position: fixed;
  inset: 0;
  z-index: 40;
  overflow: hidden;
  color: #ffffff;
  font-family: "SF Pro Display", "SF Pro Text", "Segoe UI", Arial, sans-serif;
  pointer-events: auto;
}

.radial-menu__reopen {
  position: fixed;
  left: 50%;
  bottom: 28px;
  z-index: 20;
  min-width: 142px;
  height: 40px;
  transform: translateX(-50%);
  border: 1px solid rgba(224, 61, 255, 0.72);
  border-radius: 6px;
  background: rgba(14, 5, 19, 0.94);
  color: #ffffff;
  cursor: pointer;
  font-family: "SF Pro Display", "SF Pro Text", "Segoe UI", Arial, sans-serif;
  font-size: 13px;
  font-weight: 600;
  letter-spacing: 0;
  transition:
    background 180ms ease,
    transform 180ms ease;
}

.radial-menu__reopen:hover {
  background: rgba(98, 16, 110, 0.96);
  transform: translateX(-50%) translateY(-1px);
}

.radial-menu__stage {
  position: fixed;
  left: 50%;
  bottom: 0;
  width: min(700px, calc(100vw - 24px), calc((100vh - 4px) * 2));
  aspect-ratio: 700 / 350;
  overflow: visible;
  transform: translateX(-50%);
  pointer-events: auto;
}

.radial-menu__svg,
.radial-menu__action-layer--items,
.radial-menu__main-items {
  position: absolute;
  inset: 0;
  width: 100%;
  height: 100%;
}

.radial-menu__action-layer--items,
.radial-menu__main-items {
  pointer-events: none;
}

.radial-menu__svg {
  overflow: visible;
}

.radial-menu__action-layer {
  transform-origin: 50% 100%;
}

.radial-menu__segment-base,
.radial-menu__segment-highlight,
.radial-menu__segment-hit {
  transform-box: view-box;
  transform-origin: 50% 100%;
  vector-effect: non-scaling-stroke;
}

.radial-menu--closing {
  pointer-events: none;
}

.radial-menu__action-layer--switch-leave {
  pointer-events: none;
}

.radial-menu__segment-base {
  fill: rgba(3, 6, 10, 0.995);
  stroke: rgba(3, 6, 10, 0.995);
  stroke-width: 2;
  stroke-linejoin: round;
}

.radial-menu__segment-base--inner {
  fill: rgba(8, 7, 14, 0.995);
  stroke: rgba(8, 7, 14, 0.995);
}

.radial-menu__segment-base--disabled {
  opacity: 0.6;
}

.radial-menu__segment-highlight {
  fill: url(#radialSegmentGradient);
  opacity: 0;
  filter: none;
  pointer-events: none;
  transition: opacity 420ms cubic-bezier(0.16, 1, 0.3, 1);
}

.radial-menu__segment-highlight--hovered {
  opacity: 0.5;
  filter: url(#radialPurpleGlow);
}

.radial-menu__segment-highlight--active {
  opacity: 1;
  filter: url(#radialPurpleGlow);
}

.radial-menu__segment-hit {
  fill: transparent;
  cursor: pointer;
  pointer-events: all;
}

.radial-menu__segment-hit--disabled {
  cursor: default;
}

.radial-menu__center-orb {
  fill: url(#radialCenterGradient);
  filter: drop-shadow(0 0 28px rgba(222, 47, 255, 0.36));
  transform-box: fill-box;
  transform-origin: center;
  animation: radial-center-pop 430ms cubic-bezier(0.16, 1, 0.3, 1) both;
}

.radial-menu__main-segments .radial-menu__segment-base {
  animation: radial-segment-pop 480ms cubic-bezier(0.16, 1, 0.3, 1) both;
  animation-delay: calc(260ms + var(--delay, 0ms));
}

.radial-menu__action-segments--pop .radial-menu__segment-base {
  animation: radial-segment-pop 480ms cubic-bezier(0.16, 1, 0.3, 1) both;
  animation-delay: var(--delay, 0ms);
}

.radial-menu__item {
  position: absolute;
  display: flex;
  transform: translate3d(-50%, -50%, 0);
  flex-direction: column;
  align-items: center;
  justify-content: center;
  border: 0;
  overflow: hidden;
  background: transparent;
  color: rgba(255, 255, 255, 0.74);
  text-align: center;
  cursor: pointer;
  outline: none;
  backface-visibility: visible;
  pointer-events: auto;
  transition:
    color 420ms cubic-bezier(0.16, 1, 0.3, 1),
    opacity 420ms cubic-bezier(0.16, 1, 0.3, 1),
    transform 420ms cubic-bezier(0.16, 1, 0.3, 1),
    filter 420ms cubic-bezier(0.16, 1, 0.3, 1);
}

.radial-menu__item-content {
  display: flex;
  width: 100%;
  height: 100%;
  contain: layout paint style;
  transform: translate3d(0, 0, 0);
  transform-origin: 50% 50%;
  transform-style: preserve-3d;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 3px;
  backface-visibility: visible;
  will-change: transform, opacity;
}

.radial-menu__item--inner {
  width: 86px;
  height: 54px;
}

.radial-menu__item--outer {
  width: 96px;
  height: 58px;
}

.radial-menu__main-items .radial-menu__item {
  animation: radial-item-pop 420ms cubic-bezier(0.16, 1, 0.3, 1) both;
  animation-delay: calc(340ms + var(--delay, 0ms));
}

.radial-menu__action-layer--pop .radial-menu__item {
  animation: radial-item-pop 420ms cubic-bezier(0.16, 1, 0.3, 1) both;
  animation-delay: calc(80ms + var(--delay, 0ms));
}

.radial-menu__action-layer--switch-enter .radial-menu__item {
  animation: radial-action-item-enter-center 150ms cubic-bezier(0.16, 1, 0.3, 1) both;
  animation-delay: 87ms;
}

.radial-menu__action-layer--switch-leave .radial-menu__item {
  animation: radial-action-item-leave-center 87ms cubic-bezier(0.45, 0, 0.9, 0.48) both;
}

.radial-menu__item--hovered {
  color: rgba(255, 255, 255, 0.9);
  filter: none;
  transform: translate3d(-50%, -50%, 0) scale(1.02);
}

.radial-menu__item--active {
  color: #ffffff;
  filter: none;
  transform: translate3d(-50%, -50%, 0) scale(1.045);
}

.radial-menu__action-layer--items {
  contain: layout paint style;
  isolation: isolate;
  perspective: 760px;
  transform: translate3d(0, 0, 0);
  transform-style: preserve-3d;
  will-change: transform;
}

.radial-menu__item--disabled {
  cursor: default;
  opacity: 0.58;
}

.radial-menu__icon {
  flex: 0 0 auto;
  width: 25px;
  height: 25px;
  color: currentColor;
  filter: drop-shadow(0 0 5px rgba(255, 255, 255, 0.12));
}

.radial-menu__label {
  display: -webkit-box;
  max-width: 100%;
  min-height: 14px;
  overflow: hidden;
  color: currentColor;
  font-size: 12px;
  font-weight: 400;
  line-height: 1.12;
  letter-spacing: 0;
  overflow-wrap: break-word;
  text-shadow: 0 1px 11px rgba(0, 0, 0, 0.76);
  -webkit-box-orient: vertical;
  -webkit-line-clamp: 2;
}

.radial-menu__center {
  position: absolute;
  left: 50%;
  top: 85.714%;
  display: grid;
  width: 28.571%;
  height: 28.571%;
  transform: translate(-50%, -50%);
  place-items: center;
  align-content: center;
  border: 0;
  background: transparent;
  color: rgba(255, 255, 255, 0.72);
  cursor: pointer;
  font-size: 14px;
  font-weight: 600;
  line-height: 1.12;
  letter-spacing: 0;
  text-align: center;
  animation: radial-center-text 300ms ease both;
  animation-delay: 180ms;
  transition:
    color 420ms cubic-bezier(0.16, 1, 0.3, 1),
    transform 420ms cubic-bezier(0.16, 1, 0.3, 1);
}

.radial-menu__center:hover {
  color: rgba(255, 255, 255, 0.9);
  transform: translate(-50%, -50%) scale(1.025);
}

.radial-menu__center:active {
  transform: translate(-50%, -50%) scale(0.98);
}

.radial-menu--closing .radial-menu__segment-hit {
  pointer-events: none;
}

.radial-menu--closing .radial-menu__action-layer--items .radial-menu__item {
  animation: radial-action-item-leave-center 87ms cubic-bezier(0.45, 0, 0.9, 0.48) both;
  animation-delay: var(--close-reverse-delay, 0ms);
}

.radial-menu--closing .radial-menu__action-segments .radial-menu__segment-base {
  animation: radial-segment-fold 150ms cubic-bezier(0.45, 0, 0.9, 0.48) both;
  animation-delay: var(--close-reverse-delay, 0ms);
}

.radial-menu--closing .radial-menu__action-segments .radial-menu__segment-highlight {
  transition: none;
  animation: radial-highlight-hide 80ms ease both;
}

.radial-menu--closing .radial-menu__main-items .radial-menu__item {
  animation: radial-item-fold 150ms cubic-bezier(0.45, 0, 0.9, 0.48) both;
  animation-delay: var(--close-reverse-delay, 0ms);
}

.radial-menu--closing.radial-menu--has-actions .radial-menu__main-items .radial-menu__item {
  animation-delay: calc(320ms + var(--close-reverse-delay, 0ms));
}

.radial-menu--closing .radial-menu__main-segments .radial-menu__segment-base {
  animation: radial-segment-fold 150ms cubic-bezier(0.45, 0, 0.9, 0.48) both;
  animation-delay: var(--close-reverse-delay, 0ms);
}

.radial-menu--closing.radial-menu--has-actions .radial-menu__main-segments .radial-menu__segment-base {
  animation-delay: calc(320ms + var(--close-reverse-delay, 0ms));
}

.radial-menu--closing .radial-menu__main-segments .radial-menu__segment-highlight {
  transition: none;
  animation: radial-highlight-hide 90ms ease both;
  animation-delay: 60ms;
}

.radial-menu--closing.radial-menu--has-actions .radial-menu__main-segments .radial-menu__segment-highlight {
  animation-delay: 380ms;
}

.radial-menu--closing .radial-menu__center-orb {
  animation: radial-center-fold 150ms cubic-bezier(0.45, 0, 0.9, 0.48) both;
  animation-delay: 320ms;
}

.radial-menu--closing.radial-menu--has-actions .radial-menu__center-orb {
  animation-delay: 640ms;
}

.radial-menu--closing .radial-menu__center {
  animation: radial-center-text-fold 110ms ease both;
  animation-delay: 310ms;
}

.radial-menu--closing.radial-menu--has-actions .radial-menu__center {
  animation-delay: 630ms;
}

@keyframes radial-center-pop {
  0% {
    opacity: 0;
    transform: scale(0);
  }

  78% {
    opacity: 1;
    transform: scale(1.045);
  }

  100% {
    opacity: 1;
    transform: scale(1);
  }
}

@keyframes radial-center-fold {
  0% {
    opacity: 1;
    transform: scale(1);
  }

  100% {
    opacity: 0;
    transform: scale(0);
  }
}

@keyframes radial-center-text {
  from {
    opacity: 0;
    transform: translate3d(-50%, -50%, 0) scale(0.92);
  }

  to {
    opacity: 1;
    transform: translate3d(-50%, -50%, 0) scale(1);
  }
}

@keyframes radial-center-text-fold {
  from {
    opacity: 1;
    transform: translate3d(-50%, -50%, 0) scale(1);
  }

  to {
    opacity: 0;
    transform: translate3d(-50%, -50%, 0) scale(0.88);
  }
}

@keyframes radial-segment-pop {
  0% {
    opacity: 0;
    transform: scale(0);
  }

  72% {
    opacity: 1;
    transform: scale(1.035);
  }

  100% {
    opacity: 1;
    transform: scale(1);
  }
}

@keyframes radial-segment-fold {
  0% {
    opacity: 1;
    transform: scale(1);
  }

  100% {
    opacity: 0;
    transform: scale(0);
  }
}

@keyframes radial-highlight-hide {
  to {
    opacity: 0;
  }
}

@keyframes radial-item-pop {
  0% {
    opacity: 0;
    transform: translate3d(-50%, -50%, 0) scale(0);
  }

  72% {
    opacity: 1;
    transform: translate3d(-50%, -50%, 0) scale(1.08);
  }

  100% {
    opacity: 1;
    transform: translate3d(-50%, -50%, 0) scale(1);
  }
}

@keyframes radial-item-fold {
  0% {
    opacity: 1;
    transform: translate3d(-50%, -50%, 0) scale(1);
  }

  100% {
    opacity: 0;
    transform: translate3d(
      calc(-50% + var(--shift-x, 0px)),
      calc(-50% + var(--shift-y, 24px)),
      0
    ) scale(0.72);
  }
}

@keyframes radial-action-item-enter-center {
  0% {
    opacity: 0;
    transform: translate3d(
      calc(-50% + var(--shift-x, 0px)),
      calc(-50% + var(--shift-y, 24px)),
      0
    ) scale(0.92);
  }

  100% {
    opacity: 1;
    transform: translate3d(-50%, -50%, 0) scale(1);
  }
}

@keyframes radial-action-item-leave-center {
  0% {
    opacity: 1;
    transform: translate3d(-50%, -50%, 0) scale(1);
  }

  100% {
    opacity: 0;
    transform: translate3d(
      calc(-50% + var(--shift-x, 0px)),
      calc(-50% + var(--shift-y, 24px)),
      0
    ) scale(0.92);
  }
}

@media (max-width: 720px) {
  .radial-menu__item--inner {
    width: 74px;
    height: 50px;
  }

  .radial-menu__item--outer {
    width: 82px;
    height: 52px;
  }

  .radial-menu__icon {
    width: 21px;
    height: 21px;
  }

  .radial-menu__label,
  .radial-menu__center {
    font-size: 11px;
  }
}
</style>
