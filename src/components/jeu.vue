<template>
  <main class="main-wrapper">
    <article class="game-layout">
      <section class="game-arena-container">
        <div class="stars"></div>
        <div class="stars2"></div>
        <div class="ground-limit"></div>

        <section v-if="!gameStarted && !gameOver" class="arena-modal">
          <h1>🚀 Clavier Galactique</h1>
          <p>
            Utilise ton clavier physique pour détruire les lettres avant
            qu'elles ne touchent le sol !
          </p>
          <button class="btn-start" @click="startGame">Décoller</button>
        </section>

        <section v-if="gameOver" class="arena-modal">
          <h1>💥 Mission Échouée</h1>
          <p class="final-stats">
            Score Final : <strong>{{ score }}</strong>
          </p>
          <p class="final-stats">
            Meilleur Combo : <strong>{{ maxStreak }}</strong>
          </p>
          <button class="btn-restart" @click="resetGame">Réessayer</button>
        </section>

        <section class="game-area" aria-label="Zone de jeu">
          <div
            v-for="a in asteroids"
            :key="a.id"
            class="asteroid"
            :class="{ destroying: a.destroying }"
            :style="{ left: a.x + '%', top: a.y + '%' }"
            role="img"
            :aria-label="`Astéroïde lettre ${a.letter}`"
          >
            {{ a.letter }}
          </div>
        </section>

        <transition name="shake">
          <aside v-if="showMissIndicator" class="miss-indicator" role="alert">
            ❌ Raté !
          </aside>
        </transition>
      </section>

      <aside class="side-panel" aria-label="Tableau de bord">
        <h2>Tableau de Bord</h2>

        <section class="panel-stat score-box">
          <p class="label">SCORE</p>
          <p class="value">{{ score }}</p>
        </section>

        <section
          class="panel-stat combo-box"
          :class="{ hot: streak >= 5, mega: streak >= 10 }"
        >
          <p class="label">COMBO 🔥</p>
          <p class="value">{{ streak }}</p>
          <p v-if="streak >= 5" class="combo-text">{{ getComboText() }}</p>
        </section>

        <section class="panel-stat lives-box">
          <p class="label">BOUCLIERS</p>
          <ul class="lives-icons">
            <li v-for="i in maxLives" :key="i" class="life-icon">
              {{ i <= lives ? "🛡️" : "💥" }}
            </li>
          </ul>
        </section>
      </aside>
    </article>
  </main>
</template>

<script setup>
import { ref, onMounted, onUnmounted, computed } from "vue";

// --- État du Jeu ---
const asteroids = ref([]);
const score = ref(0);
const lives = ref(3);
const maxLives = 3;
const streak = ref(0);
const maxStreak = ref(0);
const difficulty = ref(1);
const gameStarted = ref(false);
const gameOver = ref(false);
const destroyed = ref(0);
const totalKeyPresses = ref(0);
const showMissIndicator = ref(false);

// --- Configuration ---
const letters = "ABCDEFGHIJKLMNOPQRSTUVWXYZ".split("");
const GROUND_LIMIT_Y = 88;
const BASE_SPAWN_INTERVAL = 2000;
const MIN_SPAWN_INTERVAL = 800;
const DIFFICULTY_INTERVAL = 15000;

let id = 0;
let lastSpawn = 0;
let startTime = 0;
let raf = null;
let missTimeout = null;

// --- Computed ---
const accuracy = computed(() => {
  if (totalKeyPresses.value === 0) return 100;
  return Math.round((destroyed.value / totalKeyPresses.value) * 100);
});

// --- Logique ---

const spawnAsteroid = () => {
  asteroids.value.push({
    id: id++,
    letter: letters[Math.floor(Math.random() * letters.length)],
    x: Math.random() * 80 + 10,
    y: -15,
    speed: 0.5 + difficulty.value * 0.1,
    destroying: false,
  });
};

const destroyAsteroid = (letter) => {
  totalKeyPresses.value++;

  const targets = asteroids.value
    .filter((a) => a.letter === letter && !a.destroying)
    .sort((a, b) => b.y - a.y);

  if (!targets.length) {
    streak.value = 0;
    showMissIndicator.value = true;
    clearTimeout(missTimeout);
    missTimeout = setTimeout(() => {
      showMissIndicator.value = false;
    }, 500);
    return;
  }

  const target = targets[0];
  target.destroying = true;

  const comboBonus = streak.value * 5;
  score.value += 10 + comboBonus;
  streak.value++;
  destroyed.value++;

  if (streak.value > maxStreak.value) {
    maxStreak.value = streak.value;
  }

  setTimeout(() => {
    asteroids.value = asteroids.value.filter((a) => a.id !== target.id);
  }, 250);
};

const getComboText = () => {
  if (streak.value >= 20) return "LÉGENDAIRE ! 🌟";
  if (streak.value >= 15) return "INCROYABLE ! ⚡";
  if (streak.value >= 10) return "EXCELLENT ! 💫";
  if (streak.value >= 5) return "En feu ! 🔥";
  return "";
};

const loop = (t) => {
  if (!startTime) startTime = t;

  const spawnInterval = Math.max(
    MIN_SPAWN_INTERVAL,
    BASE_SPAWN_INTERVAL - difficulty.value * 150
  );

  if (t - lastSpawn > spawnInterval) {
    spawnAsteroid();
    lastSpawn = t;
  }

  asteroids.value.forEach((a) => {
    if (!a.destroying) {
      a.y += a.speed;
    }
  });

  if (t - startTime > difficulty.value * DIFFICULTY_INTERVAL) {
    difficulty.value++;
  }

  const hits = asteroids.value.filter(
    (a) => a.y > GROUND_LIMIT_Y && !a.destroying
  );

  if (hits.length) {
    lives.value -= hits.length;
    streak.value = 0;
    asteroids.value = asteroids.value.filter((a) => a.y <= GROUND_LIMIT_Y);

    if (lives.value <= 0) {
      lives.value = 0;
      gameOver.value = true;
      cancelAnimationFrame(raf);
      return;
    }
  }

  if (!gameOver.value) raf = requestAnimationFrame(loop);
};

// --- Contrôles ---

const startGame = () => {
  asteroids.value = [];
  score.value = 0;
  lives.value = maxLives;
  difficulty.value = 1;
  streak.value = 0;
  maxStreak.value = 0;
  destroyed.value = 0;
  totalKeyPresses.value = 0;
  gameStarted.value = true;
  gameOver.value = false;
  showMissIndicator.value = false;
  id = 0;
  lastSpawn = 0;
  startTime = 0;
  raf = requestAnimationFrame(loop);
};

const resetGame = () => {
  setTimeout(() => {
    startGame();
  }, 200);
};

const handleKeydown = (e) => {
  if (!gameStarted.value || gameOver.value) return;
  if (/^[a-zA-Z]$/.test(e.key)) {
    destroyAsteroid(e.key.toUpperCase());
  }
};

onMounted(() => {
  window.addEventListener("keydown", handleKeydown);
});

onUnmounted(() => {
  window.removeEventListener("keydown", handleKeydown);
  if (raf) cancelAnimationFrame(raf);
  if (missTimeout) clearTimeout(missTimeout);
});
</script>

<style scoped>
/* --- RESET & BASE --- */
ul {
  list-style: none;
  margin: 0;
  padding: 0;
}

p,
h1,
h2 {
  margin: 0;
  padding: 0;
}

/* --- LAYOUT GLOBAL --- */
.main-wrapper {
  width: 100vw;
  height: 100vh;
  background: #090a0f;
  display: flex;
  justify-content: center;
  align-items: center;
  font-family: "Arial Rounded MT Bold", "Segoe UI", sans-serif;
  color: white;
}

.game-layout {
  display: flex;
  width: 95vw;
  height: 90vh;
  max-width: 1200px;
  max-height: 900px;
  border: 4px solid #2c3e50;
  background: #1a1a2e;
  box-shadow: 0 0 50px rgba(0, 0, 0, 0.8);
}

/* --- ZONE DE JEU 4x3 --- */
.game-arena-container {
  aspect-ratio: 4 / 3;
  height: 100%;
  flex-grow: 1;
  position: relative;
  overflow: hidden;
  background: radial-gradient(ellipse at bottom, #1b2735 0%, #090a0f 100%);
  border-right: 4px solid #2c3e50;
}

.ground-limit {
  position: absolute;
  bottom: 0;
  left: 0;
  width: 100%;
  height: 12%;
  background: linear-gradient(
    to top,
    rgba(255, 50, 0, 0.6) 0%,
    transparent 100%
  );
  border-top: 3px solid #ff4500;
  box-shadow: 0 -5px 25px rgba(255, 69, 0, 0.4);
  z-index: 5;
}

/* --- PANNEAU LATÉRAL --- */
.side-panel {
  flex-basis: 300px;
  min-width: 250px;
  background: #222831;
  padding: 25px;
  display: flex;
  flex-direction: column;
  gap: 20px;
  box-shadow: inset 5px 0 20px rgba(0, 0, 0, 0.5);
}

.side-panel h2 {
  margin-bottom: 20px;
  text-align: center;
  color: #4a90e2;
  text-transform: uppercase;
  letter-spacing: 2px;
}

.panel-stat {
  background: rgba(255, 255, 255, 0.05);
  padding: 15px;
  border-radius: 10px;
  border: 1px solid rgba(255, 255, 255, 0.1);
  display: flex;
  flex-direction: column;
  align-items: center;
  transition: all 0.3s ease;
}

.label {
  font-size: 0.9rem;
  color: #aaa;
  margin-bottom: 5px;
}

.value {
  font-size: 2rem;
  font-weight: bold;
}

.score-box .value {
  color: #ffd700;
  text-shadow: 0 0 10px rgba(255, 215, 0, 0.5);
}

.combo-box {
  position: relative;
}

.combo-box.hot {
  background: rgba(255, 69, 0, 0.1);
  border-color: #ff4500;
}

.combo-box.hot .value {
  color: #ff4500;
  transform: scale(1.1);
  transition: 0.3s;
}

.combo-box.mega {
  background: rgba(255, 215, 0, 0.1);
  border-color: #ffd700;
  animation: pulse 1s infinite;
}

.combo-box.mega .value {
  color: #ffd700;
  transform: scale(1.2);
}

.combo-text {
  font-size: 0.75rem;
  color: #ff4500;
  margin-top: 5px;
  font-weight: bold;
  text-transform: uppercase;
}

@keyframes pulse {
  0%,
  100% {
    transform: scale(1);
  }
  50% {
    transform: scale(1.05);
  }
}

.lives-icons {
  font-size: 1.8rem;
  margin-top: 5px;
  display: flex;
  gap: 5px;
}

.stats-box {
  display: flex;
  flex-direction: row;
  justify-content: space-around;
  gap: 15px;
}

.mini-stat {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.mini-label {
  font-size: 0.75rem;
  color: #888;
}

.mini-value {
  font-size: 1.2rem;
  font-weight: bold;
  color: #4a90e2;
}

.controls-info {
  margin-top: auto;
  text-align: center;
  font-size: 0.8rem;
  color: #666;
}

.controls-info p {
  margin: 5px 0;
}

.tip {
  color: #4a90e2;
}

/* --- ASTÉROÏDES --- */
.game-area {
  width: 100%;
  height: 100%;
  position: relative;
}

.asteroid {
  position: absolute;
  width: 50px;
  height: 50px;
  border-radius: 50%;
  background: linear-gradient(135deg, #727a85, #434b55);
  box-shadow: inset -2px -2px 10px rgba(255, 255, 255, 0.1), 0 5px 15px black;
  color: #fff;
  font-size: 24px;
  font-weight: bold;
  display: flex;
  justify-content: center;
  align-items: center;
  border: 2px solid #888;
  z-index: 10;
  animation: rotate 4s linear infinite;
}

@keyframes rotate {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
}

.destroying {
  transform: scale(1.5) !important;
  opacity: 0;
  background: linear-gradient(135deg, #ff6b35, #f7931e);
  transition: all 0.25s ease-out;
}

/* --- MODALES --- */
.arena-modal {
  position: absolute;
  top: 40%;
  left: 50%;
  transform: translate(-50%, -50%);
  background: rgba(16, 20, 30, 0.95);
  padding: 30px;
  border-radius: 15px;
  text-align: center;
  border: 2px solid #4a90e2;
  box-shadow: 0 0 30px rgba(74, 144, 226, 0.5);
  z-index: 100;
  width: 70%;
}

.arena-modal h1 {
  margin-bottom: 15px;
}

.arena-modal p {
  margin: 10px 0;
}

.final-stats {
  color: #aaa;
}

.final-stats strong {
  color: #ffd700;
  font-size: 1.2em;
}

.btn-start,
.btn-restart {
  margin-top: 20px;
  padding: 10px 30px;
  font-size: 1.1rem;
  background: #4a90e2;
  color: white;
  border: none;
  border-radius: 50px;
  cursor: pointer;
  transition: transform 0.2s, background 0.2s;
}

.btn-start:hover,
.btn-restart:hover {
  background: #357abd;
  transform: translateY(-2px);
}

.btn-start:active,
.btn-restart:active {
  transform: scale(0.95);
}

/* --- INDICATEUR DE MISS --- */
.miss-indicator {
  position: absolute;
  top: 20%;
  left: 50%;
  transform: translateX(-50%);
  background: rgba(255, 0, 0, 0.8);
  color: white;
  padding: 10px 20px;
  border-radius: 10px;
  font-size: 1.5rem;
  font-weight: bold;
  z-index: 50;
}

.shake-enter-active {
  animation: shake 0.5s;
}

@keyframes shake {
  0%,
  100% {
    transform: translateX(-50%) translateY(0);
  }
  25% {
    transform: translateX(-50%) translateY(-10px);
  }
  75% {
    transform: translateX(-50%) translateY(10px);
  }
}

/* --- ÉTOILES --- */
.stars,
.stars2 {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  width: 100%;
  height: 100%;
  display: block;
  background: url(http://www.script-tutorials.com/demos/360/images/stars.png)
    repeat top center;
  z-index: 0;
}

.stars2 {
  background: url(http://www.script-tutorials.com/demos/360/images/stars2.png)
    repeat top center;
  z-index: 1;
  animation: stars 200s linear infinite;
  opacity: 0.3;
}

@keyframes stars {
  from {
    background-position: center 0;
  }
  to {
    background-position: center 10000px;
  }
}
</style>
