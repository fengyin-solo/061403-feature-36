<template>
  <div class="game-container" :class="gameBgClass">
    <div v-if="isBlizzard" class="snow-overlay"></div>
    <div v-if="isFreezing" class="frost-overlay"></div>
    <div v-if="isHot" class="heat-overlay"></div>
    
    <header class="game-header">
      <h1 class="game-title">❄️ 雪地生存 ❄️</h1>
      <div class="header-actions">
        <button class="mute-btn" @click="toggleMute">
          {{ muted ? '🔇' : '🔊' }}
        </button>
        <SaveManager 
          :slots="saveSlots"
          @save="handleSave"
          @load="handleLoad"
          @delete="handleDelete"
        />
      </div>
    </header>

    <main class="game-main">
      <div class="top-section">
        <DayNightIndicator 
          :isDay="isDay" 
          :dayCount="dayCount"
          :isBlizzard="isBlizzard"
        />
      </div>

      <div class="middle-section">
        <div class="left-panel">
          <Thermometer 
            :temperature="temperature" 
            :isDanger="isDanger"
            :temperatureStage="temperatureStage"
          />
          <ResourcePanel 
            :heat="heat"
            :wood="wood"
            :food="food"
            :hide="hide"
            :tools="tools"
            :temperatureStage="temperatureStage"
          />
        </div>

        <div class="center-panel">
          <div class="campfire-wrapper" :class="`temp-${temperatureStage}`">
            <Campfire :heat="heat" :canvasSize="280" :temperatureStage="temperatureStage" />
          </div>
          <div class="heat-info">
            <div class="heat-label">🔥 热量值</div>
            <div class="heat-bar-container">
              <div 
                class="heat-bar" 
                :style="{ width: heat + '%', background: getHeatGradient() }"
              ></div>
            </div>
            <div class="heat-value">{{ Math.round(heat) }}/100</div>
          </div>
        </div>

        <div class="right-panel">
          <ActionPanel 
            :isNight="isNight"
            :gameOver="gameOver"
            :canFire="canMakeFire"
            :canCraft="wood >= 2 && hide >= 1"
            :huntRate="huntSuccessRate"
            :food="food"
            :temperatureStage="temperatureStage"
            @chop="handleChop"
            @hunt="handleHunt"
            @craft="handleCraft"
            @fire="handleFire"
            @eat="handleEat"
          />
        </div>
      </div>

      <div class="bottom-section">
        <LogPanel :logs="actionLog" :temperatureStage="temperatureStage" />
      </div>
    </main>

    <GameOver 
      v-if="gameOver"
      :reason="gameOverReason"
      :dayCount="dayCount"
      :temperature="temperature"
      :wood="wood"
      :tools="tools"
      @restart="handleRestart"
      @load="showSaveManager"
    />
  </div>
</template>

<script setup>
import { computed, watch } from 'vue'
import { useGame } from './composables/useGame'
import { useAudio } from './composables/useAudio'
import Campfire from './components/Campfire.vue'
import Thermometer from './components/Thermometer.vue'
import DayNightIndicator from './components/DayNightIndicator.vue'
import ResourcePanel from './components/ResourcePanel.vue'
import ActionPanel from './components/ActionPanel.vue'
import LogPanel from './components/LogPanel.vue'
import SaveManager from './components/SaveManager.vue'
import GameOver from './components/GameOver.vue'

const {
  temperature,
  heat,
  wood,
  food,
  hide,
  tools,
  isDay,
  isNight,
  dayCount,
  isBlizzard,
  gameOver,
  gameOverReason,
  actionLog,
  isDanger,
  canMakeFire,
  huntSuccessRate,
  temperatureStage,
  isFreezing,
  isCold,
  isCool,
  isWarm,
  isHot,
  chopWood,
  hunt,
  makeTools,
  makeFire,
  eatFood,
  saveGame,
  loadGame,
  getSaveSlots,
  deleteSave,
  restartGame
} = useGame()

const {
  muted,
  playChop,
  playHunt,
  playSuccess,
  playFire,
  playWarning,
  playDanger,
  playEat,
  playCraft,
  playBlizzard,
  toggleMute
} = useAudio()

const saveSlots = computed(() => getSaveSlots())

const gameBgClass = computed(() => ({
  'day-bg': isDay.value && !isBlizzard.value,
  'night-bg': isNight.value && !isBlizzard.value,
  'blizzard-day-bg': isDay.value && isBlizzard.value,
  'blizzard-night-bg': isNight.value && isBlizzard.value,
  'temp-freezing': isFreezing.value,
  'temp-cold': isCold.value,
  'temp-cool': isCool.value,
  'temp-warm': isWarm.value,
  'temp-hot': isHot.value
}))

function getHeatGradient() {
  if (heat.value > 60) return 'linear-gradient(to right, #ff6600, #ffcc00)'
  if (heat.value > 30) return 'linear-gradient(to right, #ff9933, #ffcc00)'
  return 'linear-gradient(to right, #cc3300, #ff6600)'
}

function handleChop() {
  playChop()
  chopWood()
}

function handleHunt() {
  playHunt()
  const oldFood = food.value
  hunt()
  if (food.value > oldFood) {
    playSuccess()
  }
}

function handleCraft() {
  if (wood.value >= 2 && hide.value >= 1) {
    playCraft()
    makeTools()
    playSuccess()
  } else {
    playWarning()
  }
}

function handleFire() {
  if (canMakeFire.value) {
    playFire()
    makeFire()
    playSuccess()
  } else {
    playWarning()
  }
}

function handleEat() {
  if (food.value > 0) {
    playEat()
    eatFood()
  } else {
    playWarning()
  }
}

function handleSave(slotName) {
  saveGame(slotName)
}

function handleLoad(slotName) {
  loadGame(slotName)
}

function handleDelete(slotName) {
  deleteSave(slotName)
}

function handleRestart() {
  restartGame()
}

function showSaveManager() {
}

watch(isBlizzard, (newVal) => {
  if (newVal) {
    playBlizzard()
  }
})

watch(isDanger, (newVal) => {
  if (newVal && !muted.value) {
    playDanger()
  }
})
</script>

<style scoped>
.game-container {
  min-height: 100vh;
  padding: 20px;
  transition: background 1s ease;
  position: relative;
  overflow-x: hidden;
}

.day-bg {
  background: linear-gradient(180deg, #87CEEB 0%, #B0E0E6 50%, #E0F7FA 100%);
}

.night-bg {
  background: linear-gradient(180deg, #0f0c29 0%, #302b63 50%, #24243e 100%);
}

.blizzard-day-bg {
  background: linear-gradient(180deg, #6b7b8c 0%, #8a9ba8 50%, #a8b5c0 100%);
}

.blizzard-night-bg {
  background: linear-gradient(180deg, #1a1a2e 0%, #16213e 50%, #0f3460 100%);
}

.temp-freezing.day-bg {
  background: linear-gradient(180deg, #4a90a4 0%, #6bb5c9 30%, #a8d8e8 60%, #d4f1f9 100%),
              linear-gradient(180deg, rgba(100, 200, 255, 0.3) 0%, rgba(150, 220, 255, 0.2) 100%);
  background-blend-mode: overlay;
}

.temp-freezing.night-bg {
  background: linear-gradient(180deg, #0a1628 0%, #1a2a4a 50%, #2a3a5a 100%),
              linear-gradient(180deg, rgba(50, 150, 200, 0.4) 0%, rgba(80, 180, 220, 0.3) 100%);
  background-blend-mode: overlay;
}

.temp-freezing.blizzard-day-bg {
  background: linear-gradient(180deg, #4a5a6a 0%, #6a7a8a 50%, #8a9aaa 100%),
              linear-gradient(180deg, rgba(100, 200, 255, 0.5) 0%, rgba(150, 220, 255, 0.4) 100%);
  background-blend-mode: overlay;
}

.temp-freezing.blizzard-night-bg {
  background: linear-gradient(180deg, #0a1020 0%, #101828 50%, #182038 100%),
              linear-gradient(180deg, rgba(50, 150, 200, 0.5) 0%, rgba(80, 180, 220, 0.4) 100%);
  background-blend-mode: overlay;
}

.temp-cold.day-bg {
  background: linear-gradient(180deg, #6ab0c4 0%, #8ac8d8 50%, #b8e0ec 100%);
}

.temp-cold.night-bg {
  background: linear-gradient(180deg, #0c1a2e 0%, #1c2a4e 50%, #2c3a5e 100%);
}

.temp-cold.blizzard-day-bg {
  background: linear-gradient(180deg, #5a6a7a 0%, #7a8a9a 50%, #9aaaba 100%);
}

.temp-cold.blizzard-night-bg {
  background: linear-gradient(180deg, #121a2e 0%, #1a2438 50%, #222c42 100%);
}

.temp-cool.day-bg {
  background: linear-gradient(180deg, #7bc0d4 0%, #9ad4e4 40%, #c0e0ec 70%, #d4ebf2 100%);
}

.temp-cool.night-bg {
  background: linear-gradient(180deg, #141f3a 0%, #243050 50%, #344060 100%);
}

.temp-cool.blizzard-day-bg {
  background: linear-gradient(180deg, #6a7a8a 0%, #8a9aaa 50%, #aababa 100%);
}

.temp-cool.blizzard-night-bg {
  background: linear-gradient(180deg, #181f3a 0%, #202848 50%, #283058 100%);
}

.temp-warm.day-bg {
  background: linear-gradient(180deg, #87ceeb 0%, #b8e0f0 30%, #ffe4b5 70%, #ffd700 100%);
}

.temp-warm.night-bg {
  background: linear-gradient(180deg, #1a1020 0%, #2a1a3a 40%, #4a2a2a 70%, #6a3a1a 100%);
}

.temp-warm.blizzard-day-bg {
  background: linear-gradient(180deg, #7a8a8a 0%, #9aaaa0 50%, #c0c0a0 100%);
}

.temp-warm.blizzard-night-bg {
  background: linear-gradient(180deg, #1a1a2a 0%, #2a2a3a 50%, #4a3a2a 100%);
}

.temp-hot.day-bg {
  background: linear-gradient(180deg, #ff7f50 0%, #ffa07a 30%, #ffd700 60%, #ffff00 100%);
}

.temp-hot.night-bg {
  background: linear-gradient(180deg, #2a1010 0%, #4a2010 40%, #6a3010 70%, #8a4020 100%);
}

.temp-hot.blizzard-day-bg {
  background: linear-gradient(180deg, #8a6a5a 0%, #aa8a6a 50%, #caaa8a 100%);
}

.temp-hot.blizzard-night-bg {
  background: linear-gradient(180deg, #2a1810 0%, #4a2818 50%, #6a3820 100%);
}

.snow-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  pointer-events: none;
  z-index: 1;
  background-image: 
    radial-gradient(circle at 20% 80%, rgba(255,255,255,0.1) 0%, transparent 50%),
    radial-gradient(circle at 80% 20%, rgba(255,255,255,0.1) 0%, transparent 50%),
    radial-gradient(circle at 40% 40%, rgba(255,255,255,0.05) 0%, transparent 40%);
  animation: snowfall 3s linear infinite;
}

@keyframes snowfall {
  0% { background-position: 0 0, 0 0, 0 0; }
  100% { background-position: 50px 100px, -50px 100px, 30px 100px; }
}

.frost-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  pointer-events: none;
  z-index: 1;
  background-image: 
    radial-gradient(ellipse at 10% 90%, rgba(200, 230, 255, 0.15) 0%, transparent 40%),
    radial-gradient(ellipse at 90% 10%, rgba(200, 230, 255, 0.15) 0%, transparent 40%),
    radial-gradient(ellipse at 50% 50%, rgba(150, 200, 255, 0.1) 0%, transparent 60%);
  animation: frostPulse 3s ease-in-out infinite;
}

@keyframes frostPulse {
  0%, 100% { opacity: 0.6; }
  50% { opacity: 1; }
}

.heat-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  pointer-events: none;
  z-index: 1;
  background-image: 
    radial-gradient(ellipse at 50% 30%, rgba(255, 150, 50, 0.2) 0%, transparent 50%),
    radial-gradient(ellipse at 20% 80%, rgba(255, 100, 0, 0.1) 0%, transparent 40%),
    radial-gradient(ellipse at 80% 70%, rgba(255, 120, 30, 0.1) 0%, transparent 40%);
  animation: heatPulse 2s ease-in-out infinite;
}

@keyframes heatPulse {
  0%, 100% { opacity: 0.5; transform: scale(1); }
  50% { opacity: 0.8; transform: scale(1.02); }
}

.game-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
  position: relative;
  z-index: 10;
}

.game-title {
  color: white;
  font-size: 32px;
  text-shadow: 2px 2px 8px rgba(0, 0, 0, 0.5);
  margin: 0;
}

.header-actions {
  display: flex;
  gap: 10px;
  align-items: center;
}

.mute-btn {
  padding: 10px 15px;
  background: rgba(255, 255, 255, 0.2);
  border: 2px solid rgba(255, 255, 255, 0.3);
  border-radius: 10px;
  font-size: 18px;
  cursor: pointer;
  transition: all 0.2s;
}

.mute-btn:hover {
  background: rgba(255, 255, 255, 0.3);
  transform: scale(1.05);
}

.game-main {
  display: flex;
  flex-direction: column;
  gap: 20px;
  max-width: 1400px;
  margin: 0 auto;
  position: relative;
  z-index: 10;
}

.top-section {
  display: flex;
  justify-content: center;
}

.middle-section {
  display: grid;
  grid-template-columns: 280px 1fr 320px;
  gap: 20px;
  align-items: start;
}

.left-panel {
  display: flex;
  flex-direction: column;
  gap: 20px;
}

.center-panel {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 20px;
}

.campfire-wrapper {
  padding: 20px;
  background: rgba(0, 0, 0, 0.2);
  border-radius: 20px;
  border: 2px solid rgba(255, 255, 255, 0.1);
  transition: all 0.5s ease;
}

.campfire-wrapper.temp-freezing {
  background: linear-gradient(135deg, rgba(0, 50, 80, 0.4), rgba(0, 30, 60, 0.3));
  border-color: rgba(100, 200, 255, 0.3);
  box-shadow: 0 0 30px rgba(100, 200, 255, 0.2), inset 0 0 30px rgba(0, 50, 80, 0.3);
}

.campfire-wrapper.temp-cold {
  background: linear-gradient(135deg, rgba(0, 40, 70, 0.35), rgba(0, 25, 50, 0.25));
  border-color: rgba(80, 180, 220, 0.25);
  box-shadow: 0 0 25px rgba(80, 180, 220, 0.15);
}

.campfire-wrapper.temp-cool {
  background: linear-gradient(135deg, rgba(20, 50, 80, 0.3), rgba(30, 50, 70, 0.2));
  border-color: rgba(100, 180, 200, 0.25);
  box-shadow: 0 0 25px rgba(100, 180, 200, 0.18);
}

.campfire-wrapper.temp-warm {
  background: linear-gradient(135deg, rgba(50, 20, 0, 0.3), rgba(80, 40, 0, 0.2));
  border-color: rgba(255, 150, 50, 0.25);
  box-shadow: 0 0 25px rgba(255, 150, 50, 0.2);
}

.campfire-wrapper.temp-hot {
  background: linear-gradient(135deg, rgba(80, 30, 0, 0.4), rgba(120, 50, 0, 0.3));
  border-color: rgba(255, 100, 50, 0.4);
  box-shadow: 0 0 40px rgba(255, 100, 50, 0.4), inset 0 0 30px rgba(100, 30, 0, 0.3);
}

.heat-info {
  width: 100%;
  max-width: 300px;
  text-align: center;
}

.heat-label {
  color: white;
  font-size: 16px;
  font-weight: bold;
  margin-bottom: 10px;
  text-shadow: 1px 1px 3px rgba(0, 0, 0, 0.5);
}

.heat-bar-container {
  height: 20px;
  background: rgba(0, 0, 0, 0.3);
  border-radius: 10px;
  overflow: hidden;
  border: 2px solid rgba(255, 255, 255, 0.2);
}

.heat-bar {
  height: 100%;
  border-radius: 8px;
  transition: width 0.5s ease;
}

.heat-value {
  color: white;
  font-size: 18px;
  font-weight: bold;
  margin-top: 8px;
  text-shadow: 1px 1px 3px rgba(0, 0, 0, 0.5);
}

.right-panel {
  display: flex;
  flex-direction: column;
}

.bottom-section {
  display: grid;
  grid-template-columns: 1fr;
}

@media (max-width: 1200px) {
  .middle-section {
    grid-template-columns: 1fr;
  }
  
  .left-panel,
  .right-panel {
    flex-direction: row;
    flex-wrap: wrap;
    justify-content: center;
  }
  
  .left-panel > *,
  .right-panel > * {
    flex: 1;
    min-width: 280px;
  }
}

@media (max-width: 600px) {
  .game-container {
    padding: 10px;
  }
  
  .game-title {
    font-size: 24px;
  }
  
  .game-header {
    flex-direction: column;
    gap: 10px;
  }
  
  .left-panel,
  .right-panel {
    flex-direction: column;
  }
}
</style>
