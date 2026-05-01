<script setup>
import { computed, onMounted, onUnmounted, ref } from 'vue'

const STORAGE_KEY = 'infusoria-fashion-save-v2'
const MAX = 100

const screen = ref('start')
const tab = ref('play')

const growth = ref(0)
const hunger = ref(62)
const fun = ref(62)
const energy = ref(68)
const sleepiness = ref(30)
const coins = ref(20)
const day = ref(1)
const totalActions = ref(0)
const message = ref('Инфузория поправила реснички и ждет твоего решения.')
const isSleepingNow = ref(false)
const sleepType = ref('')
const sleepEndsAt = ref(0)
const sleepRemainingSec = ref(0)

const sparklePulse = ref(false)
const clickedPet = ref(false)
const miniGameActive = ref(false)
const miniGameMode = ref('bubbles')
const miniGameTime = ref(15)
const miniGameScore = ref(0)
const miniGameBest = ref(0)
const miniGameStartFun = ref(0)
const miniGameBubbles = ref([])
const rhythmWindowOpen = ref(false)
const MEMORY_SYMBOLS = ['⭐', '🌙', '💖']
const memoryPattern = ref([])
const memoryProgress = ref(0)
const sleepStartEnergy = ref(0)
const sleepStartSleepiness = ref(0)
const FEED_COSTS = {
  algae: 6,
  vitamins: 9,
  dessert: 12,
}
let bubbleSpawnInterval = null
let gameTimerInterval = null
let rhythmBeatInterval = null
let sleepCountdownInterval = null
let bubbleId = 0

const stageIndex = computed(() => {
  if (growth.value >= 100) return 4
  if (growth.value >= 75) return 3
  if (growth.value >= 50) return 2
  if (growth.value >= 25) return 1
  return 0
})

const stageLabel = computed(
  () =>
    [
      'Маленькая инфузория',
      'Подросшая инфузория',
      'Намек на форму туфельки',
      'Почти туфелька',
      'Настоящая модная туфля',
    ][stageIndex.value]
)

const goodState = computed(
  () => hunger.value > 60 && fun.value > 60 && energy.value > 60 && sleepiness.value < 60
)

const isHungry = computed(() => hunger.value < 35)
const isSleepy = computed(() => energy.value < 35 || sleepiness.value > 70)
const isSad = computed(() => fun.value < 35 || hunger.value < 30)
const eyesClosed = computed(() => isSleepingNow.value)
const sleepRemainingText = computed(() => {
  const min = Math.floor(sleepRemainingSec.value / 60)
  const sec = sleepRemainingSec.value % 60
  return `${String(min).padStart(2, '0')}:${String(sec).padStart(2, '0')}`
})
const progressToShoe = computed(() => `${Math.round(growth.value)}%`)
const moodText = computed(() => {
  if (fun.value < 35) return 'Грустит'
  if (fun.value < 65) return 'Спокойная'
  return 'В восторге'
})
const miniGameTitle = computed(() => {
  if (miniGameMode.value === 'bubbles') return 'Лови пузырьки'
  if (miniGameMode.value === 'stars') return 'Созвездие памяти'
  return 'Ритм ресничек'
})

const growthHint = computed(() => {
  if (goodState.value) return 'Идеальный уход! Рост ускорен.'
  if (isHungry.value || isSleepy.value || isSad.value) return 'Ей тяжело: рост замедлен.'
  return 'Стабильный уход, но можно еще лучше.'
})

const petMoodClass = computed(() => ({
  sad: isSad.value,
  sleepy: isSleepy.value,
  glow: sparklePulse.value,
  transformed: stageIndex.value >= 4,
}))

const clamp = (v) => Math.max(0, Math.min(MAX, v))

const save = () => {
  localStorage.setItem(
    STORAGE_KEY,
    JSON.stringify({
      screen: screen.value,
      tab: tab.value,
      growth: growth.value,
      hunger: hunger.value,
      fun: fun.value,
      energy: energy.value,
      sleepiness: sleepiness.value,
      coins: coins.value,
      day: day.value,
      totalActions: totalActions.value,
      message: message.value,
      isSleepingNow: isSleepingNow.value,
      sleepType: sleepType.value,
      sleepEndsAt: sleepEndsAt.value,
      sleepRemainingSec: sleepRemainingSec.value,
    })
  )
}

const load = () => {
  const raw = localStorage.getItem(STORAGE_KEY)
  if (!raw) return

  try {
    const state = JSON.parse(raw)
    screen.value = state.screen === 'final' ? 'final' : state.screen === 'game' ? 'game' : 'start'
    tab.value = ['play', 'feed', 'sleep'].includes(state.tab) ? state.tab : 'play'
    growth.value = clamp(Number(state.growth ?? 0))
    hunger.value = clamp(Number(state.hunger ?? 62))
    fun.value = clamp(Number(state.fun ?? 62))
    energy.value = clamp(Number(state.energy ?? 68))
    sleepiness.value = clamp(Number(state.sleepiness ?? 30))
    coins.value = Math.max(0, Number(state.coins ?? 20))
    day.value = Math.max(1, Number(state.day ?? 1))
    totalActions.value = Math.max(0, Number(state.totalActions ?? 0))
    message.value = state.message || message.value
    isSleepingNow.value = Boolean(state.isSleepingNow)
    sleepType.value = state.sleepType || ''
    sleepEndsAt.value = Number(state.sleepEndsAt ?? 0)
    sleepRemainingSec.value = Math.max(0, Number(state.sleepRemainingSec ?? 0))
    if (growth.value >= 100) screen.value = 'final'
    if (isSleepingNow.value) {
      tab.value = 'sleep'
      ensureSleepTimer()
    }
  } catch {
    localStorage.removeItem(STORAGE_KEY)
  }
}

const startGame = () => {
  if (growth.value === 0) {
    hunger.value = 62
    fun.value = 62
    energy.value = 68
    sleepiness.value = 30
    day.value = 1
    totalActions.value = 0
    message.value = 'Новая жизнь началась. Сегодня она мечтает стать туфелькой.'
  }
  screen.value = 'game'
  save()
}

const resetGame = () => {
  screen.value = 'start'
  tab.value = 'play'
  growth.value = 0
  hunger.value = 62
  fun.value = 62
  energy.value = 68
  sleepiness.value = 30
  coins.value = 20
  day.value = 1
  totalActions.value = 0
  isSleepingNow.value = false
  sleepType.value = ''
  sleepEndsAt.value = 0
  sleepRemainingSec.value = 0
  message.value = 'Сброс выполнен. Вода чистая, сцена готова.'
  if (sleepCountdownInterval) {
    clearInterval(sleepCountdownInterval)
    sleepCountdownInterval = null
  }
  save()
}

const passTime = (base = 1) => {
  hunger.value = clamp(hunger.value - 3 * base)
  fun.value = clamp(fun.value - 2 * base)
  energy.value = clamp(energy.value - 3 * base)
  sleepiness.value = clamp(sleepiness.value + 4 * base)
  day.value += 1
}

const applyGrowth = (actionBonus = 0.8) => {
  let gain = 0.8 + actionBonus
  if (goodState.value) gain += 1.1
  if (isHungry.value) gain -= 0.7
  if (fun.value < 30) gain -= 0.5
  if (isSleepy.value) gain -= 0.8
  gain = Math.max(0.25, gain)
  growth.value = clamp(growth.value + gain)

  sparklePulse.value = true
  setTimeout(() => {
    sparklePulse.value = false
  }, 450)

  if (growth.value >= 100) {
    growth.value = 100
    screen.value = 'final'
    message.value = 'ТАДАМ! Инфузория превратилась в настоящую туфлю.'
  }
}

const doAction = (fn) => {
  totalActions.value += 1
  fn()
  save()
}

const maybeApplyGrowth = (canGrow, bonus) => {
  if (!canGrow) return false
  applyGrowth(bonus)
  return true
}

const trySpendCoins = (cost, foodLabel) => {
  if (coins.value < cost) {
    message.value = `Не хватает монет на "${foodLabel}". Нужно ${cost}, у тебя ${coins.value}.`
    save()
    return false
  }
  coins.value -= cost
  return true
}

const blockIfSleeping = (actionLabel) => {
  if (!isSleepingNow.value) return false
  message.value = `Инфузория спит (${sleepRemainingText.value}). Сейчас нельзя ${actionLabel}.`
  save()
  return true
}

const clearSleepTimer = () => {
  if (sleepCountdownInterval) {
    clearInterval(sleepCountdownInterval)
    sleepCountdownInterval = null
  }
}

const completeSleep = () => {
  const wasLong = sleepType.value === 'long'
  const canGrowFromSleep = sleepStartEnergy.value < 100 || sleepStartSleepiness.value > 0
  isSleepingNow.value = false
  const finishedType = sleepType.value
  sleepType.value = ''
  sleepEndsAt.value = 0
  sleepRemainingSec.value = 0

  hunger.value = clamp(hunger.value - (wasLong ? 9 : 5))
  fun.value = clamp(fun.value - (wasLong ? 4 : 2))
  energy.value = clamp(energy.value + (wasLong ? 36 : 24))
  sleepiness.value = clamp(sleepiness.value - (wasLong ? 44 : 28))
  day.value += wasLong ? 2 : 1
  maybeApplyGrowth(canGrowFromSleep, wasLong ? 0.9 : 0.6)
  message.value =
    finishedType === 'long'
      ? canGrowFromSleep
        ? 'Долгий сон завершен. Инфузория выспалась и сияет.'
        : 'Долгий сон завершен. Энергия уже была полной, рост не изменился.'
      : canGrowFromSleep
        ? 'Короткий сон завершен. Она бодра и готова к приключениям.'
        : 'Короткий сон завершен. Энергия уже была полной, рост не изменился.'
  save()
}

const updateSleepRemaining = () => {
  if (!isSleepingNow.value) {
    clearSleepTimer()
    return
  }
  const seconds = Math.max(0, Math.ceil((sleepEndsAt.value - Date.now()) / 1000))
  sleepRemainingSec.value = seconds
  if (seconds <= 0) {
    clearSleepTimer()
    completeSleep()
  } else {
    save()
  }
}

const ensureSleepTimer = () => {
  if (!isSleepingNow.value) return
  updateSleepRemaining()
  if (sleepCountdownInterval) return
  sleepCountdownInterval = setInterval(updateSleepRemaining, 1000)
}

const startSleep = (type, durationSec) => {
  if (isSleepingNow.value) return
  clearMiniGameIntervals()
  miniGameActive.value = false
  miniGameBubbles.value = []
  tab.value = 'sleep'
  totalActions.value += 1
  isSleepingNow.value = true
  sleepStartEnergy.value = energy.value
  sleepStartSleepiness.value = sleepiness.value
  sleepType.value = type
  sleepEndsAt.value = Date.now() + durationSec * 1000
  sleepRemainingSec.value = durationSec
  message.value =
    type === 'long'
      ? 'Инфузория ушла в долгий сон на 10 минут. Не будим.'
      : 'Инфузория уснула на 3 минуты. Тихо-тихо.'
  ensureSleepTimer()
  save()
}

const clearMiniGameIntervals = () => {
  if (bubbleSpawnInterval) {
    clearInterval(bubbleSpawnInterval)
    bubbleSpawnInterval = null
  }
  if (gameTimerInterval) {
    clearInterval(gameTimerInterval)
    gameTimerInterval = null
  }
  if (rhythmBeatInterval) {
    clearInterval(rhythmBeatInterval)
    rhythmBeatInterval = null
  }
}

const spawnMiniBubble = () => {
  const id = bubbleId++
  const left = Math.round(8 + Math.random() * 82)
  const top = Math.round(10 + Math.random() * 72)
  const size = Math.round(22 + Math.random() * 22)
  miniGameBubbles.value.push({ id, left, top, size })
}

const selectMiniGame = (mode) => {
  if (isSleepingNow.value) return
  if (miniGameActive.value) return
  miniGameMode.value = mode
}

const generateMemoryPattern = () => {
  const length = 3 + Math.floor(Math.random() * 2)
  memoryPattern.value = Array.from({ length }, () => {
    const index = Math.floor(Math.random() * MEMORY_SYMBOLS.length)
    return MEMORY_SYMBOLS[index]
  })
  memoryProgress.value = 0
}

const finishMiniGame = () => {
  clearMiniGameIntervals()
  miniGameActive.value = false
  miniGameBubbles.value = []

  doAction(() => {
    passTime(1)
    const score = miniGameScore.value
    const canGrowFromPlay = miniGameStartFun.value < 100
    const normalizedScore = Math.max(0, score)
    const earnedCoins = Math.max(1, Math.floor(normalizedScore / 2))
    coins.value += earnedCoins
    if (normalizedScore >= 16) {
      fun.value = clamp(fun.value + 24)
      energy.value = clamp(energy.value - 8)
      hunger.value = clamp(hunger.value - 6)
      const grew = maybeApplyGrowth(canGrowFromPlay, 1.25)
      message.value = grew
        ? `${miniGameTitle.value}: топ-результат ${normalizedScore}! +${earnedCoins} монет.`
        : `${miniGameTitle.value}: топ-результат ${normalizedScore}! +${earnedCoins} монет, но веселье уже было 100%.`
    } else if (normalizedScore >= 10) {
      fun.value = clamp(fun.value + 16)
      energy.value = clamp(energy.value - 6)
      hunger.value = clamp(hunger.value - 4)
      const grew = maybeApplyGrowth(canGrowFromPlay, 0.95)
      message.value = grew
        ? `${miniGameTitle.value}: отличный заход (${normalizedScore})! +${earnedCoins} монет.`
        : `${miniGameTitle.value}: отличный заход (${normalizedScore})! +${earnedCoins} монет, но веселье уже было 100%.`
    } else {
      fun.value = clamp(fun.value + 7)
      energy.value = clamp(energy.value - 5)
      sleepiness.value = clamp(sleepiness.value + 3)
      const grew = maybeApplyGrowth(canGrowFromPlay, 0.55)
      message.value = grew
        ? `${miniGameTitle.value}: результат ${normalizedScore}. +${earnedCoins} монет, тренируемся дальше.`
        : `${miniGameTitle.value}: результат ${normalizedScore}. +${earnedCoins} монет, но веселье уже было 100%.`
    }
  })
}

const popMiniBubble = (id) => {
  if (miniGameMode.value !== 'bubbles') return
  const index = miniGameBubbles.value.findIndex((bubble) => bubble.id === id)
  if (index === -1 || !miniGameActive.value) return
  miniGameBubbles.value.splice(index, 1)
  miniGameScore.value += 1
  fun.value = clamp(fun.value + 1.2)
  if (miniGameScore.value > miniGameBest.value) {
    miniGameBest.value = miniGameScore.value
  }
  energy.value = clamp(energy.value - 0.35)
}

const tapMemorySymbol = (symbol) => {
  if (!miniGameActive.value || miniGameMode.value !== 'stars') return
  const expected = memoryPattern.value[memoryProgress.value]
  if (!expected) return

  if (symbol === expected) {
    miniGameScore.value += 1
    fun.value = clamp(fun.value + 0.9)
    memoryProgress.value += 1
    if (memoryProgress.value >= memoryPattern.value.length) {
      miniGameScore.value += 2
      energy.value = clamp(energy.value - 0.8)
      generateMemoryPattern()
    }
  } else {
    miniGameScore.value = Math.max(0, miniGameScore.value - 1)
    fun.value = clamp(fun.value - 0.8)
    energy.value = clamp(energy.value - 0.6)
    generateMemoryPattern()
  }
  if (miniGameScore.value > miniGameBest.value) miniGameBest.value = miniGameScore.value
}

const rhythmTap = () => {
  if (!miniGameActive.value || miniGameMode.value !== 'rhythm') return
  if (rhythmWindowOpen.value) {
    miniGameScore.value += 2
    fun.value = clamp(fun.value + 1.5)
  } else {
    miniGameScore.value = Math.max(0, miniGameScore.value - 1)
    fun.value = clamp(fun.value - 0.6)
  }
  energy.value = clamp(energy.value - 0.4)
  if (miniGameScore.value > miniGameBest.value) miniGameBest.value = miniGameScore.value
}

const startMiniGame = () => {
  if (blockIfSleeping('играть')) return
  tab.value = 'play'
  miniGameActive.value = true
  miniGameTime.value = 15
  miniGameScore.value = 0
  miniGameStartFun.value = fun.value
  miniGameBubbles.value = []
  rhythmWindowOpen.value = false
  memoryPattern.value = []
  memoryProgress.value = 0
  message.value = `Мини-игра "${miniGameTitle.value}" началась!`

  clearMiniGameIntervals()
  if (miniGameMode.value === 'rhythm') {
    rhythmBeatInterval = setInterval(() => {
      rhythmWindowOpen.value = !rhythmWindowOpen.value
    }, 700)
  } else if (miniGameMode.value === 'stars') {
    generateMemoryPattern()
  } else {
    for (let i = 0; i < 5; i += 1) spawnMiniBubble()
    bubbleSpawnInterval = setInterval(() => {
      if (miniGameBubbles.value.length < 9) spawnMiniBubble()
    }, 550)
  }

  gameTimerInterval = setInterval(() => {
    miniGameTime.value -= 1
    if (miniGameTime.value <= 0) {
      finishMiniGame()
    }
  }, 1000)
}

const setTab = (nextTab) => {
  if (isSleepingNow.value && nextTab !== 'sleep') {
    message.value = `Сейчас сон (${sleepRemainingText.value}). Вкладки Играть и Кормить заблокированы.`
    save()
    return
  }
  tab.value = nextTab
}

const feedAlgae = () => {
  if (blockIfSleeping('кормить')) return
  doAction(() => {
    if (!trySpendCoins(FEED_COSTS.algae, 'Микроводоросли')) return
    const canGrowFromFeed = hunger.value < 100
    tab.value = 'feed'
    passTime(1)
    hunger.value = clamp(hunger.value + 24)
    energy.value = clamp(energy.value + 6)
    sleepiness.value = clamp(sleepiness.value + 2)
    const grew = maybeApplyGrowth(canGrowFromFeed, 0.95)
    message.value = grew
      ? `Микроводоросли поданы (-${FEED_COSTS.algae} монет). Она одобрительно кивает мембраной.`
      : `Микроводоросли поданы (-${FEED_COSTS.algae} монет), но сытость уже была 100%, рост не изменился.`
  })
}

const vitaminDrop = () => {
  if (blockIfSleeping('кормить')) return
  doAction(() => {
    if (!trySpendCoins(FEED_COSTS.vitamins, 'Капля витаминов')) return
    const canGrowFromFeed = hunger.value < 100
    tab.value = 'feed'
    passTime(1)
    hunger.value = clamp(hunger.value + 14)
    energy.value = clamp(energy.value + 14)
    fun.value = clamp(fun.value + 4)
    sleepiness.value = clamp(sleepiness.value - 5)
    const grew = maybeApplyGrowth(canGrowFromFeed, 0.75)
    message.value = grew
      ? `Капля витаминов (-${FEED_COSTS.vitamins} монет): бодрость +100 к харизме.`
      : `Капля витаминов (-${FEED_COSTS.vitamins} монет), но сытость уже была 100%, рост не изменился.`
  })
}

const bubbleDessert = () => {
  if (blockIfSleeping('кормить')) return
  doAction(() => {
    if (!trySpendCoins(FEED_COSTS.dessert, 'Десерт-пузырек')) return
    const canGrowFromFeed = hunger.value < 100
    tab.value = 'feed'
    passTime(1)
    hunger.value = clamp(hunger.value + 18)
    fun.value = clamp(fun.value + 10)
    energy.value = clamp(energy.value - 1)
    sleepiness.value = clamp(sleepiness.value + 4)
    const grew = maybeApplyGrowth(canGrowFromFeed, 0.85)
    message.value = grew
      ? `Десерт-пузырек (-${FEED_COSTS.dessert} монет)! Сладко, странно, романтично.`
      : `Десерт-пузырек (-${FEED_COSTS.dessert} монет), но сытость уже была 100%, рост не изменился.`
  })
}

const shortSleep = () => {
  startSleep('short', 3 * 60)
}

const longSleep = () => {
  startSleep('long', 10 * 60)
}

const petClickReaction = () => {
  clickedPet.value = true
  setTimeout(() => {
    clickedPet.value = false
  }, 250)

  if (isSleepingNow.value) {
    message.value = `Тсс... она спит. Осталось ${sleepRemainingText.value}.`
  } else if (isHungry.value) {
    message.value = 'Она грустит и шепчет: "Можно что-нибудь вкусное, пожалуйста?"'
  } else if (isSleepy.value) {
    message.value = 'Она зевает: "Я стильная, но очень сонная..."'
  } else if (stageIndex.value < 4) {
    message.value = 'Инфузория улыбается: "Я расту! Скоро стану туфелькой!"'
  } else {
    message.value = 'Туфелька сияет: "Спасибо, стилист моей судьбы!"'
  }
  save()
}

onMounted(load)
onUnmounted(() => {
  clearMiniGameIntervals()
  clearSleepTimer()
})
</script>

<template>
  <main class="app">
    <section v-if="screen === 'start'" class="card">
      <p class="eyebrow">Mini PWA Pet-Care</p>
      <h1>Инфузория-туфелька</h1>
      <p class="lead">
        Ухаживай за питомцем, как в уютной игре про говорящего друга: играй, корми, укладывай
        спать и вырасти ее в настоящую модную туфлю.
      </p>
      <div class="chips">
        <span>Милота</span><span>Абсурд</span><span>Романтика</span>
      </div>
      <button class="btn primary" @click="startGame">Начать уход</button>
    </section>

    <section v-else-if="screen === 'game'" class="card game">
      <header class="top">
        <div>
          <p class="eyebrow">День {{ day }}</p>
          <h2>{{ stageLabel }}</h2>
        </div>
        <div class="top-right">
          <div class="coin-balance">Монеты: {{ coins }}</div>
          <button class="btn ghost" @click="resetGame">Сброс</button>
        </div>
      </header>

      <div class="pet-card">
        <div class="bubbles" aria-hidden="true">
          <span v-for="n in 8" :key="n" class="bubble" :style="{ '--i': n }"></span>
        </div>

        <div
          class="pet"
          :class="[petMoodClass, `stage-${stageIndex + 1}`, { clicked: clickedPet }]"
          @click="petClickReaction"
        >
          <span class="eye left" :class="{ closed: eyesClosed }"></span>
          <span class="eye right" :class="{ closed: eyesClosed }"></span>
          <span class="mouth"></span>
        </div>
      </div>

      <div class="meter-block">
        <div class="line"><span>Прогресс до туфли</span><strong>{{ progressToShoe }}</strong></div>
        <div class="bar pink"><i :style="{ width: `${growth}%` }"></i></div>
        <small class="hint">{{ growthHint }}</small>
      </div>

      <div class="stats">
        <div class="line"><span>Сытость</span><strong>{{ Math.round(hunger) }}%</strong></div>
        <div class="bar food"><i :style="{ width: `${hunger}%` }"></i></div>

        <div class="line"><span>Веселье</span><strong>{{ Math.round(fun) }}%</strong></div>
        <div class="bar fun"><i :style="{ width: `${fun}%` }"></i></div>
        <div class="line"><span>Настроение</span><strong>{{ moodText }}</strong></div>

        <div class="line"><span>Энергия</span><strong>{{ Math.round(energy) }}%</strong></div>
        <div class="bar energy"><i :style="{ width: `${energy}%` }"></i></div>

        <div class="line"><span>Сонливость</span><strong>{{ Math.round(sleepiness) }}%</strong></div>
        <div class="bar sleep"><i :style="{ width: `${sleepiness}%` }"></i></div>
      </div>

      <p class="msg">{{ message }}</p>

      <nav class="tabs">
        <button
          class="tab"
          :class="{ active: tab === 'play' }"
          :disabled="isSleepingNow"
          @click="setTab('play')"
        >
          Играть
        </button>
        <button
          class="tab"
          :class="{ active: tab === 'feed' }"
          :disabled="isSleepingNow"
          @click="setTab('feed')"
        >
          Кормить
        </button>
        <button class="tab" :class="{ active: tab === 'sleep' }" @click="setTab('sleep')">Спать</button>
      </nav>

      <div v-if="tab === 'play'" class="actions">
        <div class="mini-game-panel">
          <div class="game-picker">
            <button class="mini-tab" :class="{ active: miniGameMode === 'bubbles' }" @click="selectMiniGame('bubbles')">
              Лови пузырьки
            </button>
            <button class="mini-tab" :class="{ active: miniGameMode === 'stars' }" @click="selectMiniGame('stars')">
              Созвездие памяти
            </button>
            <button class="mini-tab" :class="{ active: miniGameMode === 'rhythm' }" @click="selectMiniGame('rhythm')">
              Ритм ресничек
            </button>
          </div>
          <div class="mini-game-head">
            <strong>Мини-игра: {{ miniGameTitle }}</strong>
            <span>Время: {{ miniGameTime }}с | Счет: {{ miniGameScore }} | Рекорд: {{ miniGameBest }}</span>
          </div>
          <div class="mini-game-field" :class="{ active: miniGameActive }">
            <button
              v-if="miniGameMode === 'rhythm' && miniGameActive"
              class="rhythm-btn"
              :class="{ open: rhythmWindowOpen }"
              @click="rhythmTap"
            >
              {{ rhythmWindowOpen ? 'ТЫКАЙ СЕЙЧАС!' : 'Жди ритм...' }}
            </button>
            <div v-else-if="miniGameMode === 'stars' && miniGameActive" class="memory-panel">
              <div class="memory-pattern">
                <span
                  v-for="(symbol, index) in memoryPattern"
                  :key="`${symbol}-${index}`"
                  class="memory-symbol"
                  :class="{ done: index < memoryProgress, current: index === memoryProgress }"
                >
                  {{ symbol }}
                </span>
              </div>
              <p class="memory-text">Нажми символы в правильном порядке</p>
              <div class="memory-options">
                <button
                  v-for="symbol in MEMORY_SYMBOLS"
                  :key="symbol"
                  class="memory-option"
                  @click="tapMemorySymbol(symbol)"
                >
                  {{ symbol }}
                </button>
              </div>
            </div>
            <button
              v-else
              v-for="bubble in miniGameBubbles"
              :key="bubble.id"
              class="mini-bubble-btn"
              :style="{ left: `${bubble.left}%`, top: `${bubble.top}%`, width: `${bubble.size}px`, height: `${bubble.size}px` }"
              @click="popMiniBubble(bubble.id)"
              aria-label="Лопнуть пузырек"
            ></button>
            <div v-if="!miniGameActive" class="mini-game-overlay">
              <button class="btn secondary" @click="startMiniGame">Запустить мини-игру</button>
            </div>
          </div>
        </div>
      </div>

      <div v-else-if="tab === 'feed'" class="actions">
        <button class="btn secondary" :disabled="coins < FEED_COSTS.algae" @click="feedAlgae">
          Микроводоросли ({{ FEED_COSTS.algae }} 🪙)
        </button>
        <button class="btn secondary" :disabled="coins < FEED_COSTS.vitamins" @click="vitaminDrop">
          Капля витаминов ({{ FEED_COSTS.vitamins }} 🪙)
        </button>
        <button class="btn secondary" :disabled="coins < FEED_COSTS.dessert" @click="bubbleDessert">
          Десерт-пузырек ({{ FEED_COSTS.dessert }} 🪙)
        </button>
      </div>

      <div v-else class="actions">
        <div v-if="isSleepingNow" class="sleeping-note">
          Сейчас идет {{ sleepType === 'long' ? 'долгий' : 'короткий' }} сон.
          Осталось: {{ sleepRemainingText }}
        </div>
        <button class="btn secondary" :disabled="isSleepingNow" @click="shortSleep">Короткий сон (3:00)</button>
        <button class="btn secondary" :disabled="isSleepingNow" @click="longSleep">Долгий сон (10:00)</button>
      </div>
    </section>

    <section v-else class="card final">
      <p class="eyebrow">Финал</p>
      <h2>Она стала настоящей туфлей</h2>
      <p class="lead">
        Путь от крошечной инфузории до гламурной туфельки пройден. Ты официально главный стилист
        микромира.
      </p>
      <div class="final-shoe-wrap">
        <div class="final-shoe"></div>
      </div>
      <p class="hearts">❤ ❤ ❤</p>
      <button class="btn primary" @click="resetGame">Начать заново</button>
    </section>
  </main>
</template>

<style scoped>
:root {
  color-scheme: light;
}
*,
*::before,
*::after {
  box-sizing: border-box;
}

.app {
  min-height: 100svh;
  padding: 14px;
  display: grid;
  place-items: center;
  background:
    radial-gradient(circle at 8% 10%, #ffe7f5 0%, transparent 24%),
    radial-gradient(circle at 88% 84%, #e7f7ff 0%, transparent 26%),
    linear-gradient(180deg, #fff7fc 0%, #f8fbff 100%);
  font-family: 'Trebuchet MS', 'Segoe UI', system-ui, sans-serif;
  color: #503f5f;
}

.card {
  width: min(560px, 100%);
  background: #fff;
  border: 2px solid #ffe5f2;
  border-radius: 24px;
  box-shadow: 0 14px 34px rgba(196, 114, 177, 0.2);
  padding: 16px;
  display: grid;
  gap: 12px;
}

.eyebrow {
  margin: 0;
  font-size: 11px;
  font-weight: 800;
  letter-spacing: 0.08em;
  text-transform: uppercase;
  color: #ff5cab;
}
h1,
h2,
p {
  margin: 0;
}
h1 {
  font-size: clamp(24px, 4vw, 34px);
}
h2 {
  font-size: clamp(19px, 3.8vw, 26px);
}
.lead {
  color: #736184;
}

.chips {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
}
.chips span {
  font-size: 12px;
  padding: 5px 10px;
  border-radius: 999px;
  background: #ffe4f3;
}

.btn {
  border: 0;
  border-radius: 14px;
  padding: 11px 12px;
  font-weight: 700;
  cursor: pointer;
}
.btn.primary {
  color: #fff;
  background: linear-gradient(135deg, #ff77bc, #b289ff);
}
.btn.secondary {
  color: #fff;
  background: linear-gradient(135deg, #ff9acd, #8db6ff);
}
.btn.ghost {
  background: #f7f0ff;
  color: #7657a4;
  border: 1px solid #ead8ff;
}
.btn:disabled {
  opacity: 0.55;
  cursor: not-allowed;
  filter: grayscale(0.2);
}

.top {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 8px;
}
.top-right {
  display: grid;
  gap: 8px;
  justify-items: end;
}
.coin-balance {
  font-size: 12px;
  font-weight: 800;
  color: #7a5b90;
  background: #fff5cf;
  border: 1px solid #f5de8f;
  border-radius: 999px;
  padding: 4px 10px;
}
.pet-card {
  position: relative;
  height: 220px;
  border-radius: 20px;
  background: linear-gradient(180deg, #fff2fc 0%, #ebf7ff 100%);
  overflow: hidden;
  display: grid;
  place-items: center;
}

.bubbles {
  position: absolute;
  inset: 0;
}
.bubble {
  position: absolute;
  bottom: -20px;
  left: calc(10% * var(--i));
  width: calc(8px + (var(--i) * 1px));
  height: calc(8px + (var(--i) * 1px));
  border-radius: 50%;
  background: rgba(255, 255, 255, 0.65);
  animation: bubbleUp calc(4s + var(--i) * 0.4s) linear infinite;
  animation-delay: calc(var(--i) * -0.35s);
}

.pet {
  position: relative;
  background: radial-gradient(circle at 28% 28%, #fff 0, #ffd7eb 36%, #ff7fbe 100%);
  border-radius: 58% 42% 58% 42% / 51% 57% 43% 49%;
  box-shadow: 0 10px 18px rgba(220, 95, 162, 0.35);
  animation: bob 2.8s ease-in-out infinite;
  transition: transform 0.2s ease, filter 0.2s ease;
}
.pet.clicked {
  transform: scale(0.95);
}
.pet.glow {
  filter: drop-shadow(0 0 8px rgba(255, 131, 196, 0.7));
}

.stage-1 {
  width: 88px;
  height: 66px;
}
.stage-2 {
  width: 108px;
  height: 82px;
}
.stage-3 {
  width: 126px;
  height: 90px;
}
.stage-4 {
  width: 146px;
  height: 94px;
  border-radius: 70% 30% 58% 42% / 58% 40% 60% 42%;
  transform: rotate(-8deg);
}
.stage-5,
.pet.transformed {
  width: 170px;
  height: 96px;
  border-radius: 72% 28% 56% 44% / 60% 38% 62% 40%;
  transform: rotate(-9deg);
}
.stage-4::before,
.stage-5::before,
.pet.transformed::before {
  content: '';
  position: absolute;
  right: 4px;
  bottom: -20px;
  width: 28px;
  height: 44px;
  border-radius: 42% 42% 80% 80%;
  background: linear-gradient(180deg, #ff9dcd 0%, #b88bff 100%);
}
.stage-4::after,
.stage-5::after,
.pet.transformed::after {
  content: '';
  position: absolute;
  left: 12px;
  top: 10px;
  width: 62px;
  height: 24px;
  border-radius: 90% 10% 60% 20%;
  background: rgba(255, 255, 255, 0.42);
}

.eye {
  position: absolute;
  top: 34%;
  width: 10px;
  height: 10px;
  border-radius: 999px;
  background: #5f4a6e;
  animation: blink 4s infinite;
}
.eye.left {
  left: 32%;
}
.eye.right {
  right: 32%;
}
.eye.closed {
  height: 3px;
  animation: none;
}
.mouth {
  position: absolute;
  left: 50%;
  bottom: 26%;
  width: 24px;
  height: 10px;
  border-bottom: 3px solid #785887;
  border-radius: 0 0 40px 40px;
  transform: translateX(-50%);
}
.pet.sad .mouth {
  border-bottom: 0;
  border-top: 3px solid #785887;
  border-radius: 40px 40px 0 0;
  bottom: 23%;
}
.pet.sleepy {
  filter: saturate(0.85);
}

.meter-block,
.stats {
  display: grid;
  gap: 6px;
}
.line {
  display: flex;
  justify-content: space-between;
  font-size: 14px;
  color: #735f84;
}
.bar {
  height: 10px;
  border-radius: 999px;
  background: #f0e8f7;
  overflow: hidden;
}
.bar i {
  display: block;
  height: 100%;
  border-radius: inherit;
  transition: width 0.3s ease;
}
.bar.pink i {
  background: linear-gradient(90deg, #ff73bc, #b489ff);
}
.bar.food i {
  background: linear-gradient(90deg, #ffa9cb, #ffc58e);
}
.bar.fun i {
  background: linear-gradient(90deg, #ff8ec7, #f9a869);
}
.bar.energy i {
  background: linear-gradient(90deg, #82b5ff, #9ce4ff);
}
.bar.sleep i {
  background: linear-gradient(90deg, #9a97ff, #c0b5ff);
}
.hint {
  color: #8c739f;
  font-size: 12px;
}

.msg {
  background: #fff5fb;
  border: 1px dashed #f7bfdc;
  border-radius: 14px;
  padding: 10px;
  color: #6f5e80;
}

.tabs {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 8px;
}
.tab {
  border: 1px solid #efd9ec;
  border-radius: 12px;
  background: #fff8fc;
  color: #7a648d;
  font-weight: 700;
  padding: 10px 6px;
}
.tab.active {
  color: #fff;
  border-color: transparent;
  background: linear-gradient(135deg, #ff8ec5, #b79aff);
}
.tab:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}
.actions {
  display: grid;
  gap: 8px;
}
.sleeping-note {
  background: #eef3ff;
  border: 1px dashed #b8c8ef;
  border-radius: 12px;
  padding: 10px;
  font-size: 13px;
  color: #5d6792;
}
.mini-game-panel {
  display: grid;
  gap: 8px;
  background: #fff6fc;
  border: 1px solid #f6d9ee;
  border-radius: 14px;
  padding: 10px;
}
.game-picker {
  display: grid;
  grid-template-columns: 1fr;
  gap: 6px;
}
.mini-tab {
  border: 1px solid #efd4eb;
  border-radius: 10px;
  background: #fff;
  color: #7a648d;
  font-size: 12px;
  font-weight: 700;
  padding: 8px 10px;
  cursor: pointer;
}
.mini-tab.active {
  color: #fff;
  border-color: transparent;
  background: linear-gradient(135deg, #ff8ec5, #b79aff);
}
.mini-game-head {
  display: grid;
  gap: 2px;
  color: #735f84;
}
.mini-game-head strong {
  font-size: 14px;
}
.mini-game-head span {
  font-size: 12px;
}
.mini-game-field {
  position: relative;
  height: 170px;
  border-radius: 12px;
  overflow: hidden;
  background: linear-gradient(180deg, #ffeefa 0%, #e5f3ff 100%);
  border: 1px dashed #efc9e8;
}
.mini-game-field.active {
  box-shadow: inset 0 0 20px rgba(169, 205, 255, 0.3);
}
.mini-game-overlay {
  position: absolute;
  inset: 0;
  display: grid;
  place-items: center;
  background: rgba(255, 255, 255, 0.45);
  z-index: 2;
}
.mini-bubble-btn {
  position: absolute;
  transform: translate(-50%, -50%);
  border-radius: 50%;
  border: 0;
  cursor: pointer;
  background: radial-gradient(circle at 35% 30%, #ffffff 0%, #b7e6ff 38%, #86c6ff 100%);
  box-shadow: 0 4px 10px rgba(94, 162, 229, 0.35);
  animation: bubblePopTarget 0.9s ease-in-out infinite alternate;
}
.rhythm-btn {
  position: absolute;
  inset: 22% 14%;
  border-radius: 16px;
  border: 2px solid #e3d3f4;
  background: #f6efff;
  color: #7a5ca0;
  font-size: 18px;
  font-weight: 800;
  cursor: pointer;
  transition: all 0.15s ease;
}
.rhythm-btn.open {
  background: linear-gradient(135deg, #8ff0c9, #6fd9ff);
  border-color: #5ecdb9;
  color: #21445c;
  transform: scale(1.04);
}
.memory-panel {
  position: absolute;
  inset: 0;
  display: grid;
  align-content: center;
  gap: 10px;
  padding: 14px;
}
.memory-pattern {
  display: flex;
  justify-content: center;
  gap: 8px;
}
.memory-symbol {
  width: 40px;
  height: 40px;
  border-radius: 10px;
  background: #f4ebff;
  display: grid;
  place-items: center;
  font-size: 20px;
  border: 1px solid #decdf7;
}
.memory-symbol.current {
  border-color: #72d8c4;
  box-shadow: 0 0 0 2px rgba(114, 216, 196, 0.25);
}
.memory-symbol.done {
  background: #dff9f1;
}
.memory-text {
  text-align: center;
  font-size: 13px;
  color: #725c85;
}
.memory-options {
  display: flex;
  justify-content: center;
  gap: 8px;
}
.memory-option {
  width: 46px;
  height: 46px;
  border-radius: 12px;
  border: 1px solid #e3d2f4;
  background: #fff;
  font-size: 21px;
  cursor: pointer;
}

.final .final-shoe-wrap {
  height: 180px;
  border-radius: 18px;
  display: grid;
  place-items: center;
  background: linear-gradient(180deg, #fff1fb, #edf8ff);
}
.final-shoe {
  width: 210px;
  height: 104px;
  border-radius: 72% 28% 56% 44% / 60% 38% 62% 40%;
  background: radial-gradient(circle at 24% 24%, #fff 0, #ffd6eb 35%, #ff76b8 100%);
  box-shadow: 0 10px 18px rgba(216, 96, 166, 0.34);
  transform: rotate(-9deg);
  position: relative;
  animation: transformGlow 1.8s ease-in-out infinite;
}
.final-shoe::before {
  content: '';
  position: absolute;
  right: 8px;
  bottom: -24px;
  width: 34px;
  height: 48px;
  border-radius: 42% 42% 80% 80%;
  background: linear-gradient(180deg, #ff95cc, #bb8bff);
}
.final-shoe::after {
  content: '';
  position: absolute;
  left: 16px;
  top: 10px;
  width: 66px;
  height: 24px;
  border-radius: 90% 10% 60% 20%;
  background: rgba(255, 255, 255, 0.45);
}
.hearts {
  text-align: center;
  color: #ff5cad;
  font-size: 28px;
  letter-spacing: 6px;
}

@keyframes bubbleUp {
  0% {
    transform: translateY(0) scale(0.7);
    opacity: 0.2;
  }
  20% {
    opacity: 0.7;
  }
  100% {
    transform: translateY(-250px) scale(1.2);
    opacity: 0;
  }
}
@keyframes bubblePopTarget {
  0% {
    transform: translate(-50%, -50%) scale(0.9);
  }
  100% {
    transform: translate(-50%, -50%) scale(1.08);
  }
}
@keyframes bob {
  0%,
  100% {
    translate: 0 0;
  }
  50% {
    translate: 0 -7px;
  }
}
@keyframes blink {
  0%,
  45%,
  100% {
    transform: scaleY(1);
  }
  48% {
    transform: scaleY(0.15);
  }
  52% {
    transform: scaleY(1);
  }
}
@keyframes transformGlow {
  0%,
  100% {
    filter: drop-shadow(0 0 0 rgba(255, 150, 206, 0.2));
  }
  50% {
    filter: drop-shadow(0 0 10px rgba(255, 124, 192, 0.75));
  }
}

@media (max-width: 420px) {
  .app {
    padding: 10px;
  }
  .card {
    border-radius: 18px;
    padding: 12px;
  }
}
</style>
