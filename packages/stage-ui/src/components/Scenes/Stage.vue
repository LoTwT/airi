<script setup lang="ts">
import type { DuckDBWasmDrizzleDatabase } from '@proj-airi/drizzle-duckdb-wasm'
import type { SpeechProviderWithExtraOptions } from '@xsai-ext/shared-providers'
import type { UnElevenLabsOptions } from 'unspeech'

import type { Emotion } from '../../constants/emotions'

import { drizzle } from '@proj-airi/drizzle-duckdb-wasm'
import { getImportUrlBundles } from '@proj-airi/drizzle-duckdb-wasm/bundles/import-url-browser'
// import { createTransformers } from '@xsai-transformers/embed'
// import embedWorkerURL from '@xsai-transformers/embed/worker?worker&url'
// import { embed } from '@xsai/embed'
import { generateSpeech } from '@xsai/generate-speech'
import { storeToRefs } from 'pinia'
import { onMounted, onUnmounted, ref } from 'vue'

import Live2DScene from './Live2D.vue'
import VRMScene from './VRM.vue'

import { useQueue } from '../../composables/queue'
import { useDelayMessageQueue, useEmotionsMessageQueue, useMessageContentQueue } from '../../composables/queues'
import { llmInferenceEndToken } from '../../constants'
import { EMOTION_EmotionMotionName_value, EMOTION_VRMExpressionName_value, EmotionThinkMotionName } from '../../constants/emotions'
import { useLive2d } from '../../stores'
import { useAudioContext, useSpeakingStore } from '../../stores/audio'
import { useChatStore } from '../../stores/chat'
import { useSpeechStore } from '../../stores/modules/speech'
import { useProvidersStore } from '../../stores/providers'
import { useSettings } from '../../stores/settings'

withDefaults(defineProps<{
  paused?: boolean
  focusAt: { x: number, y: number }
  xOffset?: number | string
  yOffset?: number | string
  scale?: number
}>(), { paused: false, scale: 1 })

const db = ref<DuckDBWasmDrizzleDatabase>()
// const transformersProvider = createTransformers({ embedWorkerURL })

const vrmViewerRef = ref<{ setExpression: (expression: string) => void }>()

const { stageView } = storeToRefs(useSettings())
const { mouthOpenSize } = storeToRefs(useSpeakingStore())
const { audioContext, calculateVolume } = useAudioContext()
const { onBeforeMessageComposed, onBeforeSend, onTokenLiteral, onTokenSpecial, onStreamEnd, onAssistantResponseEnd } = useChatStore()
const providersStore = useProvidersStore()

const audioAnalyser = ref<AnalyserNode>()
const nowSpeaking = ref(false)
const lipSyncStarted = ref(false)

const audioQueue = useQueue<{ audioBuffer: AudioBuffer, text: string }>({
  handlers: [
    (ctx) => {
      return new Promise((resolve) => {
        // Create an AudioBufferSourceNode
        const source = audioContext.createBufferSource()
        source.buffer = ctx.data.audioBuffer

        // Connect the source to the AudioContext's destination (the speakers)
        source.connect(audioContext.destination)
        // Connect the source to the analyzer
        source.connect(audioAnalyser.value!)

        // Start playing the audio
        nowSpeaking.value = true
        source.start(0)
        source.onended = () => {
          nowSpeaking.value = false
          resolve()
        }
      })
    },
  ],
})

const speechStore = useSpeechStore()
const { ssmlEnabled, activeSpeechProvider, activeSpeechModel, activeSpeechVoice, pitch } = storeToRefs(speechStore)

async function handleSpeechGeneration(ctx: { data: string }) {
  try {
    if (!activeSpeechProvider.value) {
      console.warn('No active speech provider configured')
      return
    }

    if (!activeSpeechVoice.value) {
      console.warn('No active speech voice configured')
      return
    }

    // TODO: UnElevenLabsOptions
    const provider = providersStore.getProviderInstance(activeSpeechProvider.value) as SpeechProviderWithExtraOptions<string, UnElevenLabsOptions>
    if (!provider) {
      console.error('Failed to initialize speech provider')
      return
    }

    const providerConfig = providersStore.getProviderConfig(activeSpeechProvider.value)

    const input = ssmlEnabled.value
      ? speechStore.generateSSML(ctx.data, activeSpeechVoice.value, { ...providerConfig, pitch: pitch.value })
      : ctx.data

    const res = await generateSpeech({
      ...provider.speech(activeSpeechModel.value, providerConfig),
      input,
      voice: activeSpeechVoice.value.id,
    })

    // Decode the ArrayBuffer into an AudioBuffer
    const audioBuffer = await audioContext.decodeAudioData(res)
    await audioQueue.add({ audioBuffer, text: ctx.data })
  }
  catch (error) {
    console.error('Speech generation failed:', error)
  }
}

const ttsQueue = useQueue<string>({
  handlers: [
    handleSpeechGeneration,
  ],
})

ttsQueue.on('add', (content) => {
  // eslint-disable-next-line no-console
  console.debug('ttsQueue added', content)
})

const messageContentQueue = useMessageContentQueue(ttsQueue)

const { currentMotion } = storeToRefs(useLive2d())

const emotionsQueue = useQueue<Emotion>({
  handlers: [
    async (ctx) => {
      if (stageView.value === '3d') {
        const value = EMOTION_VRMExpressionName_value[ctx.data]
        if (!value)
          return

        await vrmViewerRef.value!.setExpression(value)
      }
      else if (stageView.value === '2d') {
        currentMotion.value = { group: EMOTION_EmotionMotionName_value[ctx.data] }
      }
    },
  ],
})

const emotionMessageContentQueue = useEmotionsMessageQueue(emotionsQueue)
emotionMessageContentQueue.onHandlerEvent('emotion', (emotion) => {
  // eslint-disable-next-line no-console
  console.debug('emotion detected', emotion)
})

const delaysQueue = useDelayMessageQueue()
delaysQueue.onHandlerEvent('delay', (delay) => {
  // eslint-disable-next-line no-console
  console.debug('delay detected', delay)
})

function getVolumeWithMinMaxNormalizeWithFrameUpdates() {
  requestAnimationFrame(getVolumeWithMinMaxNormalizeWithFrameUpdates)
  if (!nowSpeaking.value)
    return

  mouthOpenSize.value = calculateVolume(audioAnalyser.value!, 'linear')
}

function setupLipSync() {
  if (!lipSyncStarted.value) {
    getVolumeWithMinMaxNormalizeWithFrameUpdates()
    audioContext.resume()
    lipSyncStarted.value = true
  }
}

function setupAnalyser() {
  if (!audioAnalyser.value)
    audioAnalyser.value = audioContext.createAnalyser()
}

onBeforeMessageComposed(async () => {
  setupAnalyser()
  setupLipSync()
})

onBeforeSend(async () => {
  currentMotion.value = { group: EmotionThinkMotionName }
})

onTokenLiteral(async (literal) => {
  await messageContentQueue.add(literal)
})

onTokenSpecial(async (special) => {
  await delaysQueue.add(special)
  await emotionMessageContentQueue.add(special)
})

onStreamEnd(async () => {
  await delaysQueue.add(llmInferenceEndToken)
})

onAssistantResponseEnd(async (_message) => {
  // const res = await embed({
  //   ...transformersProvider.embed('Xenova/nomic-embed-text-v1'),
  //   input: message,
  // })

  // await db.value?.execute(`INSERT INTO memory_test (vec) VALUES (${JSON.stringify(res.embedding)});`)
})

onUnmounted(() => {
  lipSyncStarted.value = false
})

onMounted(async () => {
  db.value = drizzle({ connection: { bundles: getImportUrlBundles() } })
  await db.value.execute(`CREATE TABLE memory_test (vec FLOAT[768]);`)
})
</script>

<template>
  <div relative>
    <div h-full w-full>
      <Live2DScene
        v-if="stageView === '2d'"
        :focus-at="focusAt"
        :mouth-open-size="mouthOpenSize"
        min-w="50% <lg:full" min-h="100 sm:100" h-full w-full flex-1
        :paused="paused"
        :x-offset="xOffset"
        :y-offset="yOffset"
        :scale="scale"
      />
      <VRMScene
        v-else-if="stageView === '3d'"
        ref="vrmViewerRef"
        model="/assets/vrm/models/AvatarSample-B/AvatarSample_B.vrm"
        idle-animation="/assets/vrm/animations/idle_loop.vrma"
        min-w="50% <lg:full" min-h="100 sm:100" h-full w-full flex-1
        :paused="paused"
        @error="console.error"
      />
    </div>
  </div>
</template>
