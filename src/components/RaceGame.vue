<script setup>
  import { ref, onMounted, onUnmounted } from "vue";
  import LeaderboardPopup from "./LeaderboardPopup.vue";
  import WalletConnect from "@/components/WalletConnect.vue";

  const carPosition = ref(400); // horizontal position
  const MOVE_SPEED = 7;
  const isMovingLeft = ref(false);
  const isMovingRight = ref(false);
  const gameSpeed = ref(10);
  const score = ref(0);
  const level = ref(1);
  const isGameOver = ref(false);
  const obstacles = ref([]);
  const gameLoop = ref(null);
  const isGameStarted = ref(false);
  const highScore = ref(parseInt(localStorage.getItem("highScore") || "0"));
  const isAttacking = ref(false);
  const attackCooldown = ref(false);
  const ATTACK_DURATION = 300; // milliseconds
  const ATTACK_COOLDOWN = 1000; // milliseconds
  const ATTACK_RANGE = 100; // pixels
  const bullets = ref([]);
  const BULLET_SPEED = 10;
  const BULLET_WIDTH = 8;
  const BULLET_HEIGHT = 16;

  const GAME_HEIGHT = 850;
  const GAME_WIDTH = 450;
  const CAR_SIZE = 40;
  const OBSTACLE_WIDTH = 30;
  const LEVEL_THRESHOLD = 250; // Changed from 400 to 100 for faster level progression
  const SPEED_INCREASE = 1; // Speed increase per level
  const MIN_GAP_DECREASE = 5; // Gap decrease per level
  const BASE_MIN_GAP = 80; // Starting minimum gap

  const powerUps = ref([]);
  const hasBurstAttack = ref(false);
  const POWER_UP_WIDTH = 30;
  const POWER_UP_DURATION = 5000; // 5 seconds
  const BURST_COOLDOWN = 200; // Faster shooting during burst mode

  // Add these new refs
  const powerUpTimeRemaining = ref(0);
  const powerUpTimer = ref(null);

  // New refs for Shield Power-Up
  const hasShield = ref(false);
  const SHIELD_DURATION = 5000; // 5 seconds
  const shieldTimeRemaining = ref(0);
  const shieldTimer = ref(null);

  const scoreMultiplier = ref(1);

  // Modify the audio constants
  const laserSound = new Audio("/sounds/laser.mp3");
  const explosionSound = new Audio("/sounds/explosion.mp3");
  const backgroundMusic = new Audio("/sounds/backsound.mp3"); // Your chosen 8-bit music

  // Configure the audio settings
  backgroundMusic.loop = true;
  backgroundMusic.volume = 0.4; // 40% volume is good for background music
  laserSound.volume = 0.6;
  explosionSound.volume = 0.6;

  // Add a mute state if you want to add a mute button later
  const isMuted = ref(false);

  // Add this function to toggle sound
  function toggleSound() {
    isMuted.value = !isMuted.value;
    backgroundMusic.muted = isMuted.value;
    laserSound.muted = isMuted.value;
    explosionSound.muted = isMuted.value;
  }

  // Add leaderboard refs
  const leaderboard = ref([]);
  const showLeaderboard = ref(false);
  const walletAddress = ref("");
  const isNewHighScore = ref(false);
  const MAX_LEADERBOARD_ENTRIES = 10;

  // Add this new ref at the top with other refs
  const isScoreSubmitted = ref(false);

  // Function to load leaderboard from localStorage
  function loadLeaderboard() {
    try {
      const savedLeaderboard = localStorage.getItem("leaderboard");
      if (savedLeaderboard) {
        leaderboard.value = JSON.parse(savedLeaderboard);
      }
    } catch (error) {
      console.error("Error loading leaderboard:", error);
      leaderboard.value = [];
    }
  }

  // Function to save leaderboard to localStorage
  function saveLeaderboard() {
    try {
      localStorage.setItem("leaderboard", JSON.stringify(leaderboard.value));
    } catch (error) {
      console.error("Error saving leaderboard:", error);
    }
  }

  // Function to add score to leaderboard
  function addScoreToLeaderboard(name, score) {
    // Create new entry
    const newEntry = {
      name: name || "Anonymous",
      score: score,
      level: level.value,
      date: new Date().toISOString(),
    };

    // Add to leaderboard
    leaderboard.value.push(newEntry);

    // Sort by score (descending)
    leaderboard.value.sort((a, b) => b.score - a.score);

    // Trim to maximum entries
    if (leaderboard.value.length > MAX_LEADERBOARD_ENTRIES) {
      leaderboard.value = leaderboard.value.slice(0, MAX_LEADERBOARD_ENTRIES);
    }

    // Save to localStorage
    saveLeaderboard();
  }

  // Function to toggle leaderboard visibility
  function toggleLeaderboard() {
    showLeaderboard.value = !showLeaderboard.value;
  }

  // Function to submit score (called when player enters name)
  function submitScore() {
    if (!walletAddress.value) {
      // Show error or prompt to connect wallet
      alert("Please connect your wallet to save your score!");
      return;
    }

    // Create new entry with wallet address
    const newEntry = {
      wallet: walletAddress.value,
      score: score.value,
      level: level.value,
      date: new Date().toISOString(),
    };

    // Add to leaderboard
    leaderboard.value.push(newEntry);

    // Sort by score (descending)
    leaderboard.value.sort((a, b) => b.score - a.score);

    // Trim to maximum entries
    if (leaderboard.value.length > MAX_LEADERBOARD_ENTRIES) {
      leaderboard.value = leaderboard.value.slice(0, MAX_LEADERBOARD_ENTRIES);
    }

    // Save to localStorage
    saveLeaderboard();

    // Reset states after submission
    isNewHighScore.value = false;
    isScoreSubmitted.value = true;
  }

  function startGame() {
    carPosition.value = GAME_WIDTH / 2;
    gameSpeed.value = 5;
    score.value = 0;
    level.value = 1;
    isGameOver.value = false;
    isGameStarted.value = true;
    obstacles.value = [];
    isAttacking.value = false;
    attackCooldown.value = false;
    bullets.value = [];
    powerUps.value = [];
    hasBurstAttack.value = false;
    scoreMultiplier.value = 1;

    // Only start background music if it's not already playing
    try {
      if (backgroundMusic.paused) {
        backgroundMusic.currentTime = 0;
        const playPromise = backgroundMusic.play();

        if (playPromise !== undefined) {
          playPromise.catch((error) => {
            console.log("Audio play failed:", error);
          });
        }
      }
    } catch (error) {
      console.log("Audio setup failed:", error);
    }

    if (gameLoop.value) clearInterval(gameLoop.value);

    gameLoop.value = setInterval(() => {
      updateGame();
    }, 1000 / 60); // 60 FPS

    if (powerUpTimer.value) {
      clearInterval(powerUpTimer.value);
      powerUpTimer.value = null;
    }
    powerUpTimeRemaining.value = 0;
    isScoreSubmitted.value = false;
  }

  function updateCarPosition() {
    if (isMovingLeft.value) {
      carPosition.value = Math.max(0, carPosition.value - MOVE_SPEED);
    }
    if (isMovingRight.value) {
      carPosition.value = Math.min(
        GAME_WIDTH - CAR_SIZE,
        carPosition.value + MOVE_SPEED
      );
    }
  }

  function updateLevel() {
    // Calculate points needed for each level using exponential growth
    const pointsNeeded = LEVEL_THRESHOLD * Math.pow(2, level.value - 1);
    const shouldLevelUp = score.value >= pointsNeeded;

    if (shouldLevelUp) {
      level.value++;
      gameSpeed.value += SPEED_INCREASE;
      scoreMultiplier.value = Math.floor(scoreMultiplier.value + 1);

      // Play level up sound or effect here if you want

      console.log(
        "Level Up! New speed:",
        gameSpeed.value,
        "New multiplier:",
        scoreMultiplier.value,
        "Points needed for next level:",
        LEVEL_THRESHOLD * Math.pow(2, level.value - 1)
      );
    }
  }

  function getCurrentMinGap() {
    return Math.max(BASE_MIN_GAP - (level.value - 1) * MIN_GAP_DECREASE, 40);
  }

  function getPowerUpSpawnChance() {
    // Base spawn rate is 0.2% for level 1
    if (level.value <= 1) {
      return 0.002; // 0.2% chance
    }

    // Increase spawn rate by 0.3% for each level after 1, up to a maximum of 2%
    const additionalChance = Math.min((level.value - 1) * 0.003, 0.018);
    return 0.002 + additionalChance;
  }

  function updateGame() {
    if (isGameOver.value) return;

    updateCarPosition();
    updateLevel();

    // Spawn power-ups randomly
    if (Math.random() < getPowerUpSpawnChance()) {
      // Adjust the probability for shield power-up to be less frequent
      const powerUpType = Math.random() < 0.1 ? "shield" : "burst"; // 10% chance for shield, 90% for burst
      powerUps.value.push({
        x: Math.random() * (GAME_WIDTH - POWER_UP_WIDTH),
        y: -POWER_UP_WIDTH,
        type: powerUpType,
      });
    }

    // Update power-ups
    powerUps.value.forEach((powerUp, index) => {
      powerUp.y += gameSpeed.value;

      // Check collection
      if (checkPowerUpCollision(powerUp)) {
        collectPowerUp(powerUp);
        powerUps.value.splice(index, 1);
      }
    });

    // Remove off-screen power-ups
    powerUps.value = powerUps.value.filter(
      (powerUp) => powerUp.y <= GAME_HEIGHT + POWER_UP_WIDTH
    );

    // Update bullet positions and check collisions
    bullets.value.forEach((bullet, bulletIndex) => {
      bullet.y -= BULLET_SPEED;

      // Check bullet collisions with obstacles
      obstacles.value = obstacles.value.filter((obstacle, obstacleIndex) => {
        if (checkBulletCollision(bullet, obstacle)) {
          bullets.value.splice(bulletIndex, 1);
          score.value += 5 * scoreMultiplier.value;

          return false; // Remove the obstacle
        }
        return true;
      });
    });

    // Remove bullets that are off screen
    bullets.value = bullets.value.filter((bullet) => bullet.y > -BULLET_HEIGHT);

    // Update obstacles
    obstacles.value.forEach((obs) => {
      obs.y += gameSpeed.value;

      // Check if obstacle has been passed
      if (!obs.passed && obs.y > GAME_HEIGHT - 60) {
        obs.passed = true;
        score.value += 1 * scoreMultiplier.value; // Change to 5 points for passing an obstacle
      }
    });

    // Remove obstacles that are out of view
    if (obstacles.value[0]?.y > GAME_HEIGHT + OBSTACLE_WIDTH) {
      obstacles.value.shift();
    }

    // Add new obstacles with level-based gap and passed flag
    if (
      obstacles.value.length === 0 ||
      obstacles.value[obstacles.value.length - 1].y > getCurrentMinGap()
    ) {
      obstacles.value.push({
        y: -OBSTACLE_WIDTH,
        x: Math.random() * (GAME_WIDTH - OBSTACLE_WIDTH * 2) + OBSTACLE_WIDTH,
        passed: false,
      });
    }

    // Check collisions
    obstacles.value.forEach((obs) => {
      if (checkCollision(obs)) {
        gameOver();
      }
    });
  }

  function checkCollision(obstacle) {
    if (hasShield.value) return false; // Ignore collisions if shield is active

    const collision =
      obstacle.y + OBSTACLE_WIDTH > GAME_HEIGHT - 100 &&
      obstacle.y < GAME_HEIGHT - 60 &&
      obstacle.x < carPosition.value + CAR_SIZE &&
      obstacle.x + OBSTACLE_WIDTH > carPosition.value;

    if (collision) {
      obstacle.hit = true;
    }

    return collision;
  }

  function gameOver() {
    isGameOver.value = true;

    // Check if current score is a high score
    if (score.value > highScore.value) {
      highScore.value = score.value;
      localStorage.setItem("highScore", score.value.toString());
    }

    // Check if score qualifies for leaderboard
    if (
      leaderboard.value.length < MAX_LEADERBOARD_ENTRIES ||
      score.value > leaderboard.value[leaderboard.value.length - 1].score
    ) {
      isNewHighScore.value = true;
    }

    // Play explosion sound
    try {
      explosionSound.currentTime = 0;
      explosionSound.play();
    } catch (error) {
      console.log("Audio play failed:", error);
    }

    clearInterval(gameLoop.value);
  }

  function handleAttack() {
    if (!isGameStarted.value || isGameOver.value) return;
    if (attackCooldown.value && !hasBurstAttack.value) return;

    isAttacking.value = true;
    attackCooldown.value = true;

    // Play laser sound
    laserSound.currentTime = 0; // Reset sound to start
    laserSound.play();

    // Create new bullet
    bullets.value.push({
      x: carPosition.value + CAR_SIZE / 2 - BULLET_WIDTH / 2,
      y: GAME_HEIGHT - 100,
    });

    // Add side bullets during burst attack
    if (hasBurstAttack.value) {
      bullets.value.push(
        {
          x: carPosition.value + CAR_SIZE / 2 - BULLET_WIDTH / 2 - 15,
          y: GAME_HEIGHT - 100,
        },
        {
          x: carPosition.value + CAR_SIZE / 2 - BULLET_WIDTH / 2 + 15,
          y: GAME_HEIGHT - 100,
        }
      );
    }

    // Reset attack state
    setTimeout(() => {
      isAttacking.value = false;
    }, ATTACK_DURATION);

    // Reset cooldown based on mode
    setTimeout(
      () => {
        attackCooldown.value = false;
      },
      hasBurstAttack.value ? BURST_COOLDOWN : ATTACK_COOLDOWN
    );
  }

  function checkBulletCollision(bullet, obstacle) {
    return (
      bullet.x < obstacle.x + OBSTACLE_WIDTH &&
      bullet.x + BULLET_WIDTH > obstacle.x &&
      bullet.y < obstacle.y + OBSTACLE_WIDTH &&
      bullet.y + BULLET_HEIGHT > obstacle.y
    );
  }

  function checkPowerUpCollision(powerUp) {
    return (
      powerUp.y + POWER_UP_WIDTH > GAME_HEIGHT - 100 &&
      powerUp.y < GAME_HEIGHT - 60 &&
      powerUp.x < carPosition.value + CAR_SIZE &&
      powerUp.x + POWER_UP_WIDTH > carPosition.value
    );
  }

  function collectPowerUp(powerUp) {
    if (powerUp.type === "burst") {
      hasBurstAttack.value = true;
      powerUpTimeRemaining.value = POWER_UP_DURATION / 1000; // Convert to seconds
    } else if (powerUp.type === "shield") {
      hasShield.value = true;
      shieldTimeRemaining.value = SHIELD_DURATION / 1000; // Convert to seconds
    }

    // Clear existing timer if there is one
    if (powerUpTimer.value) {
      clearInterval(powerUpTimer.value);
    }

    // Start countdown timer
    powerUpTimer.value = setInterval(() => {
      if (hasBurstAttack.value) {
        powerUpTimeRemaining.value--;
        if (powerUpTimeRemaining.value <= 0) {
          hasBurstAttack.value = false;
          powerUpTimeRemaining.value = 0;
        }
      }
      if (hasShield.value) {
        shieldTimeRemaining.value--;
        if (shieldTimeRemaining.value <= 0) {
          hasShield.value = false;
          shieldTimeRemaining.value = 0;
        }
      }
      if (powerUpTimeRemaining.value <= 0 && shieldTimeRemaining.value <= 0) {
        clearInterval(powerUpTimer.value);
        powerUpTimer.value = null;
      }
    }, 1000);

    // Original power-up timeout
    setTimeout(
      () => {
        if (powerUp.type === "burst") {
          hasBurstAttack.value = false;
          powerUpTimeRemaining.value = 0;
        } else if (powerUp.type === "shield") {
          hasShield.value = false;
          shieldTimeRemaining.value = 0;
        }
        if (powerUpTimer.value) {
          clearInterval(powerUpTimer.value);
          powerUpTimer.value = null;
        }
      },
      powerUp.type === "burst" ? POWER_UP_DURATION : SHIELD_DURATION
    );
  }

  function handleKeydown(e) {
    if (!isGameStarted.value && e.code === "Space") {
      startGame();
      return;
    }

    if (isGameOver.value && e.code === "Space") {
      startGame();
      return;
    }

    if (e.code === "Space" && isGameStarted.value && !isGameOver.value) {
      handleAttack();
      return;
    }

    if (e.key === "ArrowLeft") {
      isMovingLeft.value = true;
    } else if (e.key === "ArrowRight") {
      isMovingRight.value = true;
    }

    // Add escape key to close leaderboard
    if (e.key === "Escape" && showLeaderboard.value) {
      showLeaderboard.value = false;
      return;
    }
  }

  function handleKeyup(e) {
    if (e.key === "ArrowLeft") {
      isMovingLeft.value = false;
    } else if (e.key === "ArrowRight") {
      isMovingRight.value = false;
    }
  }

  onMounted(() => {
    window.addEventListener("keydown", handleKeydown);
    window.addEventListener("keyup", handleKeyup);

    // Load the leaderboard
    loadLeaderboard();
  });

  onUnmounted(() => {
    window.removeEventListener("keydown", handleKeydown);
    window.removeEventListener("keyup", handleKeyup);
    if (gameLoop.value) clearInterval(gameLoop.value);
    if (powerUpTimer.value) {
      clearInterval(powerUpTimer.value);
    }
    // Stop background music only when component is unmounted
    backgroundMusic.pause();
    backgroundMusic.currentTime = 0;
  });

  // Add a handler for wallet connection
  const handleWalletConnected = (address) => {
    walletAddress.value = address;
    // If you want to automatically submit when wallet is connected
    // submitScore();
  };
</script>

<template>
  <div class="game-container">
    <!-- Add v-show to the score display to hide it when leaderboard is shown -->
    <div class="road-hud" v-show="!showLeaderboard">
      <div class="score-container">
        <div class="score-value">{{ score.toString().padStart(6, "0") }}</div>
        <div class="level-badge">LVL {{ level }}</div>
      </div>

      <div class="high-score">
        BEST: {{ highScore.toString().padStart(6, "0") }}
      </div>
    </div>

    <div
      class="game-area"
      :style="{ height: GAME_HEIGHT + 'px', width: GAME_WIDTH + 'px' }"
    >
      <!-- Background Grid -->
      <div class="grid-bg"></div>

      <!-- Guide Lines -->
      <div class="guide-lines">
        <div class="guide-line left"></div>
        <div class="guide-line right"></div>
      </div>

      <!-- Car -->
      <div
        class="car pixel-art"
        :class="{ attacking: isAttacking, shielded: hasShield }"
        :style="{
          left: carPosition + 'px',
        }"
      >
        <div class="car-flames"></div>
        <div class="car-body">
          <div class="car-window"></div>
          <div class="car-frame"></div>
          <div class="car-boost left"></div>
          <div class="car-boost right"></div>
        </div>
        <div class="car-trail"></div>
        <div class="car-shadow"></div>
        <div v-if="isAttacking" class="attack-effect"></div>
      </div>

      <!-- Obstacles -->
      <div
        v-for="(obstacle, index) in obstacles"
        :key="index"
        class="obstacle pixel-art"
        :class="{ hit: obstacle.hit }"
        :style="{
          left: obstacle.x + 'px',
          top: obstacle.y + 'px',
        }"
      />

      <!-- Bullets -->
      <div
        v-for="(bullet, index) in bullets"
        :key="`bullet-${index}`"
        class="bullet"
        :class="{ burst: hasBurstAttack }"
        :style="{
          left: bullet.x + 'px',
          top: bullet.y + 'px',
        }"
      />

      <!-- Power-ups -->
      <div
        v-for="(powerUp, index) in powerUps"
        :key="`powerup-${index}`"
        class="power-up"
        :class="{ shield: powerUp.type === 'shield' }"
        :style="{
          left: powerUp.x + 'px',
          top: powerUp.y + 'px',
        }"
      >
        <div class="power-up-icon">
          {{ powerUp.type === "shield" ? "🛡️" : "⚡" }}
        </div>
      </div>

      <!-- Start Screen -->
      <div v-if="!isGameStarted && !isGameOver" class="start-screen retro-text">
        <h2>READY TO RACE</h2>
        <p class="controls">← → to move, SPACE to attack</p>
        <p class="blink">PRESS SPACE TO START</p>
      </div>

      <!-- Add v-show to the game over score board as well -->
      <div v-if="isGameOver" class="game-over" v-show="!showLeaderboard">
        <h2 class="retro-text">GAME OVER</h2>
        <div class="score-board">
          <div class="final-score-text">SCORE: {{ score }}</div>
          <div class="final-high-score-text">HIGH: {{ highScore }}</div>
          <div class="final-level-text">LEVEL: {{ level }}</div>

          <!-- Show high score form only if it's a new high score and score hasn't been submitted -->
          <div
            v-if="isNewHighScore && !isScoreSubmitted"
            class="high-score-form"
          >
            <p class="new-record">NEW HIGH SCORE!</p>

            <!-- Show wallet connect if not connected -->
            <div v-if="!walletAddress">
              <p class="save-score-text">
                Connect your wallet to save your score:
              </p>
              <WalletConnect @walletConnected="handleWalletConnected" />
            </div>

            <!-- Show submit button if wallet is connected -->
            <div v-else>
              <p class="save-score-text">
                Ready to save your score with wallet:
              </p>
              <p class="wallet-address save-score-text">
                {{
                  walletAddress.substring(0, 6) +
                  "..." +
                  walletAddress.substring(38)
                }}
              </p>
              <button @click="submitScore" class="submit-btn">
                Submit Score
              </button>
            </div>
          </div>

          <!-- Show confirmation after submission -->
          <div v-if="isScoreSubmitted" class="score-submitted">
            <p>Score saved successfully!</p>
          </div>

          <div class="game-over-buttons">
            <button @click="toggleLeaderboard" class="leaderboard-btn">
              View Leaderboard
            </button>
          </div>
        </div>
        <div class="blink">PRESS SPACE TO RESTART</div>
      </div>

      <!-- Leaderboard overlay -->
      <LeaderboardPopup
        :show="showLeaderboard"
        :leaderboard="leaderboard"
        :current-score="score"
        :current-level="level"
        :connected-wallet="walletAddress"
        @close="toggleLeaderboard"
        @wallet-connected="handleWalletConnected"
      />
    </div>
  </div>
</template>

<style scoped>
  .game-container {
    display: flex;
    flex-direction: row;
    align-items: flex-start;
    gap: 2rem;
    font-family: "Courier New", monospace;
    perspective: 1000px;
  }

  .game-area {
    position: relative;
    background: #120458;
    border: 4px solid #ff00ff;
    overflow: hidden;
    box-shadow: 0 0 10px #ff00ff, 0 0 20px #00ffff;
    transform: rotateX(10deg);
  }

  /* Grid Background */
  .grid-bg {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background: linear-gradient(#ff00ff22 0%, transparent 100%),
      linear-gradient(90deg, #00ffff22 1px, transparent 1px) 0 0 / 20px 20px,
      linear-gradient(#00ffff22 1px, transparent 1px) 0 0 / 20px 20px;
    animation: gridMove 20s linear infinite;
  }

  @keyframes gridMove {
    from {
      background-position-y: 0;
    }
    to {
      background-position-y: 1000px;
    }
  }

  .car {
    position: absolute;
    bottom: 24px;
    width: 40px;
    height: 40px;
    z-index: 100;
    transition: transform 0.1s ease;
  }

  .car-body {
    position: relative;
    width: 100%;
    height: 100%;
    background: #00ffff;
    clip-path: polygon(50% 0%, 100% 100%, 0% 100%);
    transform: translateZ(2px);
    filter: drop-shadow(0 0 5px #00ffff);
  }

  .car-window {
    position: absolute;
    top: 30%;
    left: 50%;
    transform: translateX(-50%);
    width: 30%;
    height: 20%;
    background: #ffffff;
    clip-path: polygon(50% 0%, 100% 100%, 0% 100%);
    opacity: 0.7;
  }

  .car-frame {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    clip-path: polygon(50% 0%, 100% 100%, 0% 100%);
    background: transparent;
    border: 2px solid #00ffff;
    filter: drop-shadow(0 0 3px #00ffff);
  }

  .car-boost {
    position: absolute;
    bottom: -5px;
    width: 8px;
    height: 12px;
    background: #ff00ff;
    clip-path: polygon(50% 100%, 0% 0%, 100% 0%);
    animation: boostPulse 0.2s infinite alternate;
    transition: transform 0.1s ease;
  }

  .car-boost.left {
    left: 5px;
    transform: rotate(15deg);
  }

  .car-boost.right {
    right: 5px;
    transform: rotate(-15deg);
  }

  .car-flames {
    position: absolute;
    bottom: -15px;
    left: 50%;
    transform: translateX(-50%);
    width: 20px;
    height: 20px;
    background: linear-gradient(
      to top,
      transparent,
      #ff00ff 20%,
      #ff66ff 40%,
      #ffaaff 60%,
      transparent 100%
    );
    filter: blur(1px);
    opacity: 0.7;
    animation: flameFlicker 0.1s infinite alternate;
  }

  .car-trail {
    position: absolute;
    bottom: -40px;
    left: 50%;
    transform: translateX(-50%);
    width: 4px;
    height: 40px;
    background: linear-gradient(
      to top,
      transparent,
      rgba(0, 255, 255, 0.2) 20%,
      rgba(0, 255, 255, 0.4) 60%,
      rgba(0, 255, 255, 0.8) 100%
    );
    filter: blur(1px);
    opacity: 0;
    transition: opacity 0.1s ease;
  }

  .car-shadow {
    position: absolute;
    bottom: -5px;
    left: 50%;
    transform: translateX(-50%);
    width: 100%;
    height: 10px;
    background: rgba(0, 0, 0, 0.3);
    filter: blur(4px);
    border-radius: 50%;
  }

  .obstacle {
    position: absolute;
    width: 30px;
    height: 30px;
    background: #ff3366;
    border-radius: 5px;
    box-shadow: 0 0 10px rgba(255, 51, 102, 0.5);
  }

  /* Enemy car design */
  .obstacle::before {
    content: "";
    position: absolute;
    top: 5px;
    left: 5px;
    right: 5px;
    height: 8px;
    background: #ffcc00;
    border-radius: 2px;
  }

  .obstacle::after {
    content: "";
    position: absolute;
    bottom: 5px;
    left: 5px;
    right: 5px;
    height: 8px;
    background: #ff0000;
    border-radius: 2px;
  }

  .game-over {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    color: #ff3366;
    font-size: 32px;
    font-weight: bold;
    text-shadow: 0 0 10px rgba(255, 51, 102, 0.5);
    background: rgba(18, 4, 88, 0.9);
    padding: 2rem;
    border: 2px solid #ff00ff;
    border-radius: 8px;
    text-align: center;
    width: 80%;
    max-width: 300px;
  }

  .retro-text {
    color: #00ffff;
    text-shadow: 2px 2px #ff00ff, 4px 4px #120458;
    letter-spacing: 2px;
  }

  .score-board {
    display: flex;
    flex-direction: column;
    align-items: center;
    margin: 15px 0;
  }

  /* Smaller text for final score on game over screen */
  .final-score-text,
  .final-high-score-text,
  .final-level-text {
    font-size: 20px;
    color: white;
    margin: 5px 0;
    text-shadow: 0 0 8px rgba(0, 255, 255, 0.5);
  }

  .final-score-text {
    color: #00ffff;
    font-weight: bold;
  }

  .final-high-score-text {
    color: #ff00ff;
  }

  .blink {
    font-size: 1rem;
    margin-top: 1rem;
    animation: blink 1s infinite;
  }

  .pixel-art {
    image-rendering: pixelated;
    image-rendering: crisp-edges;
  }

  /* Scanlines effect */
  .game-area::after {
    content: "";
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background: linear-gradient(transparent 50%, rgba(0, 0, 0, 0.1) 50%);
    background-size: 100% 4px;
    pointer-events: none;
  }

  .guide-lines {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    pointer-events: none;
  }

  .guide-line {
    position: absolute;
    top: 0;
    bottom: 0;
    width: 2px;
    background: rgba(0, 255, 255, 0.1);
  }

  .guide-line.left {
    left: 40px;
  }

  .guide-line.right {
    right: 40px;
  }

  .start-screen {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    text-align: center;
    background: rgba(18, 4, 88, 0.9);
    padding: 2rem;
    border: 4px solid #ff00ff;
    box-shadow: 0 0 10px #ff00ff, 0 0 20px #00ffff;
  }

  .controls {
    margin: 1rem 0;
    font-size: 1.2rem;
    color: #00ffff;
    content: "← → to move, SPACE to attack";
  }

  .new-record {
    color: #ff00ff;
    font-size: 1.5rem;
    margin: 1rem 0;
    animation: pulse 0.5s ease-in-out infinite alternate;
  }

  .obstacle.hit {
    animation: explode 0.5s ease-out forwards;
  }

  @keyframes explode {
    0% {
      transform: scale(1);
      opacity: 1;
    }
    100% {
      transform: scale(3);
      opacity: 0;
    }
  }

  @keyframes pulse {
    from {
      transform: scale(1);
    }
    to {
      transform: scale(1.1);
    }
  }

  @keyframes boostPulse {
    from {
      height: 12px;
      opacity: 0.8;
    }
    to {
      height: 15px;
      opacity: 1;
    }
  }

  @keyframes flameFlicker {
    0% {
      transform: translateX(-50%) scaleY(1);
      opacity: 0.7;
    }
    100% {
      transform: translateX(-50%) scaleY(1.2);
      opacity: 0.9;
    }
  }

  @keyframes blink {
    0%,
    100% {
      opacity: 1;
    }
    50% {
      opacity: 0;
    }
  }

  .road-hud {
    position: absolute;
    top: 55px;
    left: 50%;
    transform: translateX(-50%);
    z-index: 50;
    width: 200px;
    text-align: center;
    perspective: 100px;
  }

  .score-container {
    background: rgba(0, 0, 0, 0.7);
    border: 2px solid #00ffff;
    border-radius: 20px;
    padding: 10px;
    margin-bottom: 10px;
    box-shadow: 0 0 10px rgba(0, 255, 255, 0.3),
      inset 0 0 20px rgba(0, 255, 255, 0.2);
    transform: rotateX(10deg);
  }

  .score-value {
    font-family: "Press Start 2P", monospace;
    font-size: 24px;
    color: #fff;
    text-shadow: 0 0 5px #00ffff, 0 0 10px #00ffff;
    letter-spacing: 2px;
  }

  .level-badge {
    position: absolute;
    top: -15px;
    left: 50%;
    transform: translateX(-50%);
    background: #ff00ff;
    padding: 5px 15px;
    border-radius: 15px;
    font-size: 12px;
    font-weight: bold;
    color: #fff;
    text-shadow: 0 0 5px rgba(255, 0, 255, 0.5);
    box-shadow: 0 0 10px rgba(255, 0, 255, 0.5),
      inset 0 0 5px rgba(255, 255, 255, 0.5);
  }

  .high-score {
    font-family: "Press Start 2P", monospace;
    font-size: 12px;
    color: rgba(255, 255, 255, 0.7);
    text-shadow: 0 0 5px rgba(0, 255, 255, 0.3);
    opacity: 0.8;
    transform: rotateX(10deg);
    margin-top: 5px;
  }

  /* Animation for score changes */
  @keyframes scoreUpdate {
    0% {
      transform: rotateX(10deg) scale(1);
    }
    50% {
      transform: rotateX(10deg) scale(1.1);
    }
    100% {
      transform: rotateX(10deg) scale(1);
    }
  }

  .score-value.updated {
    animation: scoreUpdate 0.2s ease-out;
  }

  /* Level up animation */
  @keyframes levelUp {
    0% {
      transform: translateX(-50%) scale(1);
    }
    50% {
      transform: translateX(-50%) scale(1.2);
      box-shadow: 0 0 20px rgba(255, 0, 255, 0.8),
        inset 0 0 10px rgba(255, 255, 255, 0.8);
    }
    100% {
      transform: translateX(-50%) scale(1);
    }
  }

  .level-badge.level-up {
    animation: levelUp 0.5s ease-out;
  }

  .attack-effect {
    position: absolute;
    top: -20px;
    left: 50%;
    transform: translateX(-50%);
    width: 4px;
    height: 20px;
    background: #00ffff;
    box-shadow: 0 0 10px #00ffff;
    animation: shootFlash 0.3s ease-out;
  }

  @keyframes shootFlash {
    0% {
      transform: translateX(-50%) scaleY(0.2);
      opacity: 1;
    }
    100% {
      transform: translateX(-50%) scaleY(1);
      opacity: 0;
    }
  }

  .car.attacking {
    transform: scale(1.2);
  }

  .bullet {
    position: absolute;
    width: 8px;
    height: 16px;
    background: #00ffff;
    border-radius: 4px;
    box-shadow: 0 0 10px #00ffff;
    z-index: 95;
  }

  .bullet::after {
    content: "";
    position: absolute;
    top: -8px;
    left: 50%;
    transform: translateX(-50%);
    width: 4px;
    height: 8px;
    background: rgba(0, 255, 255, 0.5);
    filter: blur(2px);
  }

  .power-up {
    position: absolute;
    width: 30px;
    height: 30px;
    background: rgba(255, 255, 0, 0.3);
    border-radius: 50%;
    box-shadow: 0 0 15px rgba(255, 255, 0, 0.5);
    animation: powerUpFloat 1s ease-in-out infinite alternate;
    z-index: 95;
  }

  .power-up-icon {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    font-size: 20px;
    color: #ffff00;
    text-shadow: 0 0 10px rgba(255, 255, 0, 0.8);
  }

  @keyframes powerUpFloat {
    from {
      transform: translateY(0) scale(1);
    }
    to {
      transform: translateY(-5px) scale(1.1);
    }
  }

  .car.attacking.burst {
    filter: drop-shadow(0 0 10px #ffff00);
  }

  /* Modify existing bullet style */
  .bullet {
    /* existing styles... */
    transition: width 0.3s, height 0.3s, background-color 0.3s;
  }

  .bullet.burst {
    background: #ffff00;
    box-shadow: 0 0 15px #ffff00;
  }

  .power-up-status {
    position: absolute;
    top: 20px;
    right: 20px;
    display: flex;
    align-items: center;
    gap: 10px;
    background: rgba(0, 0, 0, 0.5);
    padding: 10px;
    border-radius: 20px;
    border: 2px solid #ffff00;
    box-shadow: 0 0 10px rgba(255, 255, 0, 0.5);
    z-index: 1000;
  }

  .power-up-status .power-up-icon {
    font-size: 24px;
    color: #ffff00;
    text-shadow: 0 0 10px rgba(255, 255, 0, 0.8);
    animation: pulseIcon 1s ease-in-out infinite alternate;
  }

  .power-up-bar-container {
    width: 100px;
    height: 10px;
    background: rgba(255, 255, 255, 0.2);
    border-radius: 5px;
    overflow: hidden;
  }

  .power-up-bar {
    height: 100%;
    background: #ffff00;
    box-shadow: 0 0 10px rgba(255, 255, 0, 0.5);
    transition: width 0.1s linear;
  }

  .power-up-timer {
    color: #ffff00;
    font-size: 16px;
    font-weight: bold;
    min-width: 40px;
    text-align: right;
  }

  @keyframes pulseIcon {
    from {
      transform: scale(1);
      opacity: 0.8;
    }
    to {
      transform: scale(1.2);
      opacity: 1;
    }
  }

  .power-up.shield {
    background: rgba(0, 255, 255, 0.3);
    box-shadow: 0 0 15px rgba(0, 255, 255, 0.5);
  }

  .car.shielded {
    filter: drop-shadow(0 0 10px #00ffff);
  }

  .leaderboard-toggle {
    display: none; /* Hide it completely */
  }

  .leaderboard-overlay {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background: rgba(0, 0, 0, 0.8);
    display: flex;
    justify-content: center;
    align-items: center;
    z-index: 2000;
  }

  .leaderboard-panel {
    background: rgba(18, 4, 88, 0.95);
    border: 4px solid #ff00ff;
    border-radius: 8px;
    padding: 20px;
    width: 80%;
    max-width: 500px;
    max-height: 80%;
    overflow-y: auto;
    box-shadow: 0 0 20px rgba(255, 0, 255, 0.8), 0 0 40px rgba(0, 255, 255, 0.5);
  }

  .leaderboard-panel h2 {
    color: #00ffff;
    text-align: center;
    margin-bottom: 20px;
    font-size: 28px;
  }

  /* Leaderboard table styles */
  .leaderboard-table {
    width: 100%;
    border-collapse: collapse;
    color: white;
    margin-bottom: 20px;
  }

  .leaderboard-table th,
  .leaderboard-table td {
    padding: 10px;
    text-align: center;
    border-bottom: 1px solid rgba(255, 0, 255, 0.3);
  }

  .leaderboard-table th {
    color: #00ffff;
    font-size: 18px;
    font-weight: bold;
    border-bottom: 2px solid #ff00ff;
  }

  .leaderboard-table tr:nth-child(odd) {
    background: rgba(255, 0, 255, 0.1);
  }

  .leaderboard-table tr:hover {
    background: rgba(0, 255, 255, 0.1);
  }

  /* Button styles */
  .close-btn,
  .restart-btn,
  .leaderboard-btn,
  .submit-btn {
    background: rgba(255, 0, 255, 0.2);
    color: #00ffff;
    border: 2px solid #00ffff;
    padding: 8px 16px;
    margin: 5px;
    border-radius: 8px;
    font-size: 16px;
    cursor: pointer;
    transition: all 0.2s ease;
  }

  .close-btn:hover,
  .restart-btn:hover,
  .leaderboard-btn:hover,
  .submit-btn:hover {
    background: rgba(0, 255, 255, 0.2);
    transform: scale(1.05);
    box-shadow: 0 0 10px rgba(0, 255, 255, 0.5);
  }

  /* High score form styles */
  .high-score-form {
    margin: 15px 0;
    padding: 10px;
    background: rgba(0, 0, 0, 0.3);
    border-radius: 8px;
    border: 1px solid #ff00ff;
  }

  .input-container {
    display: flex;
    flex-direction: column;
    gap: 8px;
  }

  .input-container label {
    color: #00ffff;
    font-size: 14px;
  }

  .input-container input {
    background: rgba(0, 0, 0, 0.5);
    border: 2px solid #00ffff;
    border-radius: 4px;
    color: white;
    padding: 8px;
    font-size: 16px;
    outline: none;
  }

  .input-container input:focus {
    box-shadow: 0 0 8px rgba(0, 255, 255, 0.5);
  }

  /* Animation for the new record text */
  .new-record {
    color: #ff00ff;
    font-size: 18px;
    text-align: center;
    margin-bottom: 10px;
    animation: pulse 0.5s ease-in-out infinite alternate;
  }

  .game-over-buttons {
    display: flex;
    justify-content: center;
    gap: 10px;
    margin-top: 15px;
  }

  .wallet-address {
    font-family: monospace;
    background: rgba(0, 0, 0, 0.5);
    padding: 5px 10px;
    border-radius: 4px;
    display: inline-block;
    color: #00ffff;
    margin: 5px 0;
  }

  .score-submitted {
    color: #00ff66;
    margin-top: 10px;
    font-size: 16px;
    animation: fadeIn 0.5s ease-in-out;
  }

  @keyframes fadeIn {
    from {
      opacity: 0;
      transform: translateY(-10px);
    }
    to {
      opacity: 1;
      transform: translateY(0);
    }
  }

  .save-score-text {
    font-size: 14px;
    color: #00ffff;
    margin-bottom: 8px;
    opacity: 0.9;
  }
</style>
