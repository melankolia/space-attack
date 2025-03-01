<script setup>
  import { defineEmits, onMounted, ref, watch } from "vue";
  import WalletConnect from "./WalletConnect.vue";
  import { useVirtualList } from "@vueuse/core";
  import axios from "axios";

  const props = defineProps({
    show: {
      type: Boolean,
      required: true,
    },
    connectedWallet: {
      type: String,
      default: "",
    },
    currentScore: {
      type: Number,
      default: 0,
    },
    currentLevel: {
      type: Number,
      default: 1,
    },
  });

  const emit = defineEmits(["close", "walletConnected"]);
  const serverLeaderboard = ref([]);
  const isLoading = ref(false);
  const error = ref(null);

  // Fetch leaderboard data when component is shown
  async function fetchLeaderboard() {
    isLoading.value = true;
    error.value = null;

    try {
      const response = await axios.get(
        "https://space-attack-be-production.up.railway.app/leaderboard"
      );
      serverLeaderboard.value = response.data.leaderboard.map((entry) => ({
        wallet: entry.address,
        score: parseInt(entry.score),
        level: parseInt(entry.level),
      }));
    } catch (err) {
      console.error("Error fetching leaderboard:", err);
      error.value = "Failed to load leaderboard data";
    } finally {
      isLoading.value = false;
    }
  }

  // Watch for show prop changes to fetch data when leaderboard is opened
  watch(
    () => props.show,
    (newValue) => {
      if (newValue) {
        fetchLeaderboard();
      }
    }
  );

  function closeLeaderboard() {
    emit("close");
  }

  function handleWalletConnected(address) {
    emit("walletConnected", address);
  }

  function formatAddress(address) {
    if (!address) return "";
    return `${address.slice(0, 6)}...${address.slice(-4)}`;
  }
</script>

<template>
  <div v-if="show" class="leaderboard-overlay">
    <div class="leaderboard-panel">
      <h2 class="retro-text">TOP SCORES</h2>

      <div v-if="isLoading" class="loading-state">Loading leaderboard...</div>

      <div v-else-if="error" class="error-state">
        {{ error }}
      </div>

      <div v-else class="table-container">
        <table class="leaderboard-table">
          <thead>
            <tr>
              <th>Rank</th>
              <th>Wallet</th>
              <th>Score</th>
              <th>Level</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="(entry, index) in serverLeaderboard" :key="index">
              <td>{{ index + 1 }}</td>
              <td>{{ formatAddress(entry.wallet) }}</td>
              <td>{{ entry.score }}</td>
              <td>{{ entry.level }}</td>
            </tr>
            <tr v-if="serverLeaderboard.length === 0">
              <td colspan="4">No scores yet. Be the first!</td>
            </tr>
          </tbody>
        </table>
      </div>

      <!-- Current score section -->
      <div class="current-score-section">
        <div class="current-score-title">YOUR SCORE</div>
        <div class="current-score-stats">
          <div>
            Score: <span class="score-value">{{ currentScore }}</span>
          </div>
          <div>
            Level: <span class="score-value">{{ currentLevel }}</span>
          </div>
        </div>
      </div>

      <button @click="closeLeaderboard" class="close-btn">Close</button>
    </div>
  </div>
</template>

<style scoped>
  .leaderboard-overlay {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background: rgba(0, 0, 0, 0.8);
    display: flex;
    justify-content: center;
    align-items: center;
    z-index: 9999;
    padding: 20px;
  }

  .leaderboard-panel {
    background: rgba(18, 4, 88, 0.95);
    border: 4px solid #ff00ff;
    border-radius: 8px;
    padding: 20px;
    width: 95%;
    max-width: 800px;
    height: 90vh;
    max-height: 700px;
    display: flex;
    flex-direction: column;
    box-shadow: 0 0 20px rgba(255, 0, 255, 0.8), 0 0 40px rgba(0, 255, 255, 0.5);
  }

  .retro-text {
    color: #00ffff;
    text-shadow: 2px 2px #ff00ff, 4px 4px #120458;
    letter-spacing: 2px;
    text-align: center;
    margin-bottom: 20px;
    font-size: clamp(20px, 4vw, 28px);
  }

  .table-container {
    flex: 1;
    overflow-y: auto;
    margin-bottom: 20px;
    position: relative;
    scrollbar-width: thin;
    scrollbar-color: #ff00ff #120458;
  }

  .table-container::-webkit-scrollbar {
    width: 8px;
  }

  .table-container::-webkit-scrollbar-track {
    background: #120458;
  }

  .table-container::-webkit-scrollbar-thumb {
    background: #ff00ff;
    border-radius: 4px;
  }

  .leaderboard-table {
    width: 100%;
    border-collapse: collapse;
    color: white;
  }

  .leaderboard-table th,
  .leaderboard-table td {
    padding: 12px 20px;
    text-align: center;
    border-bottom: 1px solid rgba(255, 0, 255, 0.3);
    font-size: clamp(14px, 2.5vw, 18px);
  }

  .leaderboard-table th {
    color: #00ffff;
    font-weight: bold;
    border-bottom: 2px solid #ff00ff;
    position: sticky;
    top: 0;
    background: rgba(18, 4, 88, 0.95);
    z-index: 10000;
  }

  .leaderboard-table th:nth-child(1),
  .leaderboard-table td:nth-child(1) {
    width: 15%;
  }

  .leaderboard-table th:nth-child(2),
  .leaderboard-table td:nth-child(2) {
    width: 40%;
  }

  .leaderboard-table th:nth-child(3),
  .leaderboard-table td:nth-child(3) {
    width: 25%;
  }

  .leaderboard-table th:nth-child(4),
  .leaderboard-table td:nth-child(4) {
    width: 20%;
  }

  .leaderboard-table tr:nth-child(odd) {
    background: rgba(255, 0, 255, 0.1);
  }

  .leaderboard-table tr:hover {
    background: rgba(0, 255, 255, 0.1);
  }

  .wallet-section {
    margin: 15px 0;
  }

  .wallet-status {
    text-align: center;
    color: #00ffff;
    font-size: clamp(12px, 2.5vw, 16px);
    margin: 10px 0;
  }

  .close-btn {
    background: rgba(255, 0, 255, 0.2);
    color: #00ffff;
    border: 2px solid #00ffff;
    padding: 8px 16px;
    border-radius: 8px;
    font-size: clamp(14px, 2.5vw, 16px);
    cursor: pointer;
    transition: all 0.2s ease;
    width: 100%;
  }

  .close-btn:hover {
    background: rgba(0, 255, 255, 0.2);
    transform: scale(1.02);
    box-shadow: 0 0 10px rgba(0, 255, 255, 0.5);
  }

  /* Media Queries */
  @media (max-width: 768px) {
    .leaderboard-panel {
      width: 95%;
      padding: 15px;
    }

    .leaderboard-table th,
    .leaderboard-table td {
      padding: 10px 8px;
    }
  }

  @media (max-width: 480px) {
    .leaderboard-panel {
      width: 98%;
      padding: 10px;
    }

    .leaderboard-table th,
    .leaderboard-table td {
      padding: 8px 4px;
      font-size: clamp(12px, 2.5vw, 14px);
    }
  }

  .current-score-section {
    margin: 20px 0;
    padding: 15px;
    background: rgba(255, 0, 255, 0.1);
    border: 2px solid #ff00ff;
    border-radius: 8px;
    text-align: center;
  }

  .current-score-title {
    color: #00ffff;
    font-size: 18px;
    margin-bottom: 10px;
    text-shadow: 1px 1px #ff00ff;
  }

  .current-score-stats {
    display: flex;
    justify-content: center;
    gap: 30px;
    color: white;
    font-size: 16px;
  }

  .score-value {
    color: #00ffff;
    font-weight: bold;
    font-size: 20px;
  }

  /* Add new styles for loading and error states */
  .loading-state,
  .error-state {
    text-align: center;
    padding: 20px;
    color: #00ffff;
    font-size: 16px;
    background: rgba(0, 0, 0, 0.3);
    border-radius: 8px;
    margin: 20px 0;
  }

  .error-state {
    color: #ff4444;
    border: 1px solid #ff4444;
  }
</style>
