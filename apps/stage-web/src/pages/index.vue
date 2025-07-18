<script setup lang="ts">
import { WidgetStage } from '@proj-airi/stage-ui/components/scenes'
import { useLive2d } from '@proj-airi/stage-ui/stores'
import { breakpointsTailwind, useBreakpoints, useDark, useMouse } from '@vueuse/core'
import { storeToRefs } from 'pinia'
import { onMounted, ref, watch } from 'vue'

import Cross from '../components/Backgrounds/Cross.vue'
import Header from '../components/Layouts/Header.vue'
import InteractiveArea from '../components/Layouts/InteractiveArea.vue'
import MobileInteractiveArea from '../components/Layouts/MobileInteractiveArea.vue'
import AnimatedWave from '../components/Widgets/AnimatedWave.vue'

import { themeColorFromPropertyOf, useThemeColor } from '../composables/theme-color'

const dark = useDark()
const paused = ref(false)

function handleSettingsOpen(open: boolean) {
  paused.value = open
}

const positionCursor = useMouse()
const { scale, position, positionInPercentageString } = storeToRefs(useLive2d())
const breakpoints = useBreakpoints(breakpointsTailwind)
const isMobile = breakpoints.smaller('md')

const { updateThemeColor } = useThemeColor(themeColorFromPropertyOf('.widgets.top-widgets .colored-area', 'background-color'))
watch(dark, () => updateThemeColor(), { immediate: true })
onMounted(() => updateThemeColor())
</script>

<template>
  <Cross>
    <AnimatedWave
      class="widgets top-widgets"
      :fill-color="dark
        ? 'oklch(35% calc(var(--chromatic-chroma) * 0.6) var(--chromatic-hue))'
        : 'color-mix(in srgb, oklch(95% calc(var(--chromatic-chroma-50) * 0.5) var(--chromatic-hue)) 80%, oklch(100% 0 360))'"
    >
      <div relative flex="~ col" z-2 h-100dvh w-100vw of-hidden>
        <!-- header -->
        <div class="px-0 py-1 md:px-3 md:py-3" w-full gap-2>
          <Header class="flex" />
        </div>
        <!-- page -->
        <div relative flex="~ 1 row gap-y-0 gap-x-2 <md:col">
          <WidgetStage
            flex-1 min-w="1/2"
            :paused="paused"
            :focus-at="{
              x: positionCursor.x.value,
              y: positionCursor.y.value,
            }"
            :x-offset="`${isMobile ? position.x : position.x - 10}%`"
            :y-offset="positionInPercentageString.y"
            :scale="scale"
          />
          <InteractiveArea class="flex <md:hidden" absolute h="85dvh" right-4 flex-1 max-w="500px" min-w="30%" />
          <MobileInteractiveArea class="hidden <md:block" mx2 mb2 @settings-open="handleSettingsOpen" />
        </div>
      </div>
    </AnimatedWave>
  </Cross>
</template>

<route lang="yaml">
name: IndexScenePage
meta:
  layout: stage
  stageTransition:
    name: bubble-wave-out
</route>
