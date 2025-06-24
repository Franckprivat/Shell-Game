<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'

// Props (gardez si vous voulez passer un message depuis App.vue)
defineProps<{ msg?: string }>()

// État du jeu
const shells = ref([
  { hasPearl: false, revealed: false },
  { hasPearl: false, revealed: false },
  { hasPearl: false, revealed: false }
])

const gameState = ref<'initial' | 'showing' | 'playing' | 'revealed'>('initial')
const message = ref('Cliquez sur "Commencer" pour débuter la partie !')
const isLoading = ref(false)
const isShuffling = ref(false)
const wins = ref(0)
const losses = ref(0)
const streak = ref(0)
const correctShellIndex = ref(-1)

const messageClass = computed(() => {
  if (isLoading.value) return 'loading'
  if (message.value.includes('Bravo')) return 'success'
  if (message.value.includes('Dommage')) return 'error'
  return ''
})

// Charger les scores depuis le localStorage
onMounted(() => {
  const savedWins = localStorage.getItem('bonneteau-wins')
  const savedLosses = localStorage.getItem('bonneteau-losses')
  const savedStreak = localStorage.getItem('bonneteau-streak')
  
  if (savedWins) wins.value = parseInt(savedWins)
  if (savedLosses) losses.value = parseInt(savedLosses)
  if (savedStreak) streak.value = parseInt(savedStreak)
})

const saveScore = () => {
  localStorage.setItem('bonneteau-wins', wins.value.toString())
  localStorage.setItem('bonneteau-losses', losses.value.toString())
  localStorage.setItem('bonneteau-streak', streak.value.toString())
}

const resetShells = () => {
  shells.value.forEach(shell => {
    shell.hasPearl = false
    shell.revealed = false
  })
}

const getRandomPosition = async (): Promise<number> => {
  try {
    isLoading.value = true
    message.value = 'Récupération de la position aléatoire...'
    
    const response = await fetch('https://www.random.org/integers/?num=1&min=0&max=2&col=1&base=10&format=plain&rnd=new')
    
    if (!response.ok) {
      throw new Error('Erreur de réseau')
    }
    
    const randomNumber = await response.text()
    return parseInt(randomNumber.trim())
  } catch (error) {
    console.error('Erreur lors de la récupération du nombre aléatoire:', error)
    message.value = 'Erreur réseau, utilisation d\'un nombre local...'
    // Fallback vers un nombre aléatoire local
    return Math.floor(Math.random() * 3)
  } finally {
    isLoading.value = false
  }
}

const startNewGame = async () => {
  resetShells()
  gameState.value = 'showing' // Nouvel état pour montrer la perle
  
  // Obtenir la position aléatoire
  correctShellIndex.value = await getRandomPosition()
  shells.value[correctShellIndex.value].hasPearl = true
  
  // Montrer clairement où est la perle en soulevant le coquillage
  message.value = 'Regardez bien ! La perle est sous ce coquillage... 👀'
  shells.value[correctShellIndex.value].revealed = true
  
  setTimeout(() => {
    message.value = 'Mémorisez bien la position, les coquillages vont se mélanger !'
    setTimeout(() => {
      shells.value[correctShellIndex.value].revealed = false
      gameState.value = 'playing'
      isShuffling.value = true
      message.value = 'Les coquillages se mélangent... 🌊'
      
      setTimeout(() => {
        isShuffling.value = false
        message.value = 'À vous de jouer ! Cliquez sur un coquillage.'
      }, 2500)
    }, 1000)
  }, 2500)
}

const selectShell = (index: number) => {
  if (gameState.value !== 'playing' || isShuffling.value || isLoading.value) {
    return
  }

  gameState.value = 'revealed'
  shells.value[index].revealed = true

  if (shells.value[index].hasPearl) {
    wins.value++
    streak.value++
    message.value = `🎉 Bravo ! Vous avez trouvé la perle ! Série de ${streak.value}`
    
    // Révéler tous les coquillages après un court délai
    setTimeout(() => {
      shells.value.forEach(shell => shell.revealed = true)
    }, 1000)
  } else {
    losses.value++
    streak.value = 0
    message.value = '💔 Dommage ! La perle était ailleurs...'
    
    // Révéler le bon coquillage
    setTimeout(() => {
      shells.value.forEach(shell => shell.revealed = true)
    }, 1000)
  }
  
  saveScore()
}

const resetScore = () => {
  wins.value = 0
  losses.value = 0
  streak.value = 0
  saveScore()
  message.value = 'Scores remis à zéro !'
}
</script>

<template>
  <div class="game-container">
    <h1>🐚 Jeu du Bonneteau 🐚</h1>
    <p class="subtitle">Trouvez la perle cachée sous le coquillage !</p>
    
    <div class="score">
      <div class="score-item">
        <div>Victoires: {{ wins }}</div>
      </div>
      <div class="score-item">
        <div>Défaites: {{ losses }}</div>
      </div>
      <div class="score-item">
        <div>Série: {{ streak }}</div>
      </div>
    </div>

    <div class="message" :class="messageClass">{{ message }}</div>

    <div class="game-area">
      <div class="shells-container">
        <div 
          v-for="(shell, index) in shells" 
          :key="index"
          class="shell"
          :class="{
            'shuffling': isShuffling,
            'revealed': (gameState === 'showing' || gameState === 'revealed') && shell.revealed,
            'correct': gameState === 'revealed' && shell.hasPearl && shell.revealed,
            'wrong': gameState === 'revealed' && !shell.hasPearl && shell.revealed
          }"
          @click="selectShell(index)"
          :style="{ animationDelay: index * 0.1 + 's' }"
        >
          <div class="pearl" v-if="shell.hasPearl"></div>
        </div>
      </div>
    </div>

    <div class="controls">
      <button 
        class="btn primary" 
        @click="startNewGame"
        :disabled="isLoading || isShuffling"
      >
        {{ gameState === 'initial' ? 'Commencer' : 'Nouvelle Partie' }}
      </button>
      
      <button 
        class="btn" 
        @click="resetScore"
        :disabled="isLoading || isShuffling"
      >
        Remettre à zéro
      </button>
    </div>

    <div class="instructions">
      <strong>Comment jouer :</strong><br>
      1. Cliquez sur "Commencer" pour placer la perle<br>
      2. Les coquillages vont se mélanger<br>
      3. Cliquez sur le coquillage où vous pensez que se trouve la perle<br>
      4. Tentez de faire la plus longue série possible !
    </div>
  </div>
</template>

<style scoped>
.game-container {
  text-align: center;
  background: rgba(255, 255, 255, 0.1);
  backdrop-filter: blur(10px);
  border-radius: 20px;
  padding: 40px;
  box-shadow: 0 20px 40px rgba(0, 0, 0, 0.3);
  max-width: 600px;
  width: 90%;
  margin: 0 auto;
  color: white;
}

h1 {
  font-size: 2.5rem;
  margin-bottom: 10px;
  text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.3);
}

.subtitle {
  font-size: 1.1rem;
  margin-bottom: 30px;
  opacity: 0.9;
}

.game-area {
  margin: 40px 0;
}

.shells-container {
  display: flex;
  justify-content: center;
  gap: 20px;
  margin: 40px 0;
  perspective: 1000px;
}

.shell {
  width: 100px;
  height: 100px;
  background: linear-gradient(145deg, #ff9a9e 0%, #fecfef 50%, #fecfef 100%);
  border-radius: 50% 50% 50% 50% / 60% 60% 40% 40%;
  cursor: pointer;
  transition: all 0.3s ease;
  position: relative;
  box-shadow: 0 10px 20px rgba(0, 0, 0, 0.2);
  transform-style: preserve-3d;
}

.shell:hover {
  transform: translateY(-5px) rotateX(10deg);
  box-shadow: 0 15px 30px rgba(0, 0, 0, 0.3);
}

.shell.shuffling {
  animation: shuffle 0.8s ease-in-out;
}

.shell.revealed {
  transform: translateY(-20px) rotateX(35deg);
  background: linear-gradient(145deg, rgba(255, 154, 158, 0.7) 0%, rgba(254, 207, 239, 0.7) 50%, rgba(254, 207, 239, 0.7) 100%);
}

.shell.correct {
  background: linear-gradient(145deg, #4facfe 0%, #00f2fe 100%);
  animation: celebrate 0.6s ease-in-out;
}

.shell.wrong {
  background: linear-gradient(145deg, #ff6b6b 0%, #ee5a52 100%);
  animation: shake 0.5s ease-in-out;
}

.pearl {
  position: absolute;
  top: 80%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 25px;
  height: 25px;
  background: radial-gradient(circle at 30% 30%, #ffffff, #f0f0f0, #e0e0e0);
  border-radius: 50%;
  box-shadow: 0 0 15px rgba(255, 255, 255, 0.8), 0 0 25px rgba(255, 255, 255, 0.4);
  opacity: 0;
  transition: opacity 0.3s ease;
  animation: sparkle 1.5s ease-in-out infinite;
  z-index: 10;
}

.shell.revealed .pearl {
  opacity: 1;
}

.controls {
  margin: 30px 0;
}

.btn {
  background: linear-gradient(45deg, #ff6b6b, #ee5a52);
  color: white;
  border: none;
  padding: 15px 30px;
  font-size: 1.1rem;
  border-radius: 50px;
  cursor: pointer;
  transition: all 0.3s ease;
  box-shadow: 0 5px 15px rgba(0, 0, 0, 0.2);
  margin: 0 10px;
}

.btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 8px 20px rgba(0, 0, 0, 0.3);
}

.btn:disabled {
  background: #666;
  cursor: not-allowed;
  transform: none;
}

.btn.primary {
  background: linear-gradient(45deg, #4facfe, #00f2fe);
}

.score {
  display: flex;
  justify-content: center;
  gap: 40px;
  margin: 20px 0;
  font-size: 1.2rem;
}

.score-item {
  background: rgba(255, 255, 255, 0.2);
  padding: 10px 20px;
  border-radius: 15px;
  backdrop-filter: blur(5px);
}

.message {
  font-size: 1.3rem;
  margin: 20px 0;
  min-height: 30px;
  font-weight: bold;
  text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.3);
}

.loading {
  color: #ffd700;
}

.success {
  color: #00ff88;
}

.error {
  color: #ff6b6b;
}

@keyframes shuffle {
  0%, 100% { transform: translateX(0); }
  25% { transform: translateX(-30px) rotateZ(-10deg); }
  75% { transform: translateX(30px) rotateZ(10deg); }
}

@keyframes celebrate {
  0%, 100% { transform: scale(1); }
  50% { transform: scale(1.1); }
}

@keyframes shake {
  0%, 100% { transform: translateX(0); }
  25% { transform: translateX(-5px); }
  75% { transform: translateX(5px); }
}

@keyframes sparkle {
  0%, 100% { 
    box-shadow: 0 0 15px rgba(255, 255, 255, 0.8), 0 0 25px rgba(255, 255, 255, 0.4);
    transform: translate(-50%, -50%) scale(1);
  }
  50% { 
    box-shadow: 0 0 20px rgba(255, 255, 255, 1), 0 0 35px rgba(255, 255, 255, 0.6);
    transform: translate(-50%, -50%) scale(1.1);
  }
}

.instructions {
  background: rgba(255, 255, 255, 0.1);
  border-radius: 15px;
  padding: 20px;
  margin: 20px 0;
  font-size: 0.9rem;
  line-height: 1.5;
}

@media (max-width: 600px) {
  .shells-container {
    gap: 10px;
  }
  
  .shell {
    width: 80px;
    height: 80px;
  }
  
  h1 {
    font-size: 2rem;
  }
  
  .score {
    flex-direction: column;
    gap: 10px;
  }
}
</style>