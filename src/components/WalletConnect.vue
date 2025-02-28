<script setup>
  import { ref, defineEmits } from "vue";

  const emit = defineEmits(["walletConnected"]);
  const isConnecting = ref(false);
  const errorMessage = ref("");
  const successMessage = ref("");
  const connectedAddress = ref("");

  async function connectWallet() {
    if (!window.ethereum) {
      errorMessage.value = "Please install MetaMask to save scores!";
      return;
    }

    try {
      isConnecting.value = true;
      errorMessage.value = "";
      successMessage.value = "";

      const accounts = await window.ethereum.request({
        method: "eth_requestAccounts",
      });

      if (accounts.length > 0) {
        connectedAddress.value = accounts[0];
        // Display shortened wallet address
        const shortAddress = `${accounts[0].substring(
          0,
          6
        )}...${accounts[0].substring(38)}`;
        successMessage.value = `Connected: ${shortAddress}`;
        emit("walletConnected", accounts[0]);
      }
    } catch (err) {
      errorMessage.value = "Failed to connect wallet. Please try again.";
      console.error(err);
    } finally {
      isConnecting.value = false;
    }
  }
</script>

<template>
  <div class="wallet-connect">
    <button @click="connectWallet" :disabled="isConnecting" class="connect-btn">
      {{
        isConnecting
          ? "Connecting..."
          : connectedAddress
          ? "Wallet Connected"
          : "Connect Wallet"
      }}
    </button>
    <p v-if="errorMessage" class="error-message">{{ errorMessage }}</p>
    <p v-if="successMessage" class="success-message">{{ successMessage }}</p>
  </div>
</template>

<style scoped>
  .wallet-connect {
    text-align: center;
    margin: 15px 0;
  }

  .connect-btn {
    background: rgba(255, 0, 255, 0.2);
    color: #00ffff;
    border: 2px solid #00ffff;
    padding: 8px 16px;
    border-radius: 8px;
    font-size: 16px;
    cursor: pointer;
    transition: all 0.2s ease;
  }

  .connect-btn:hover:not(:disabled) {
    background: rgba(0, 255, 255, 0.2);
    transform: scale(1.05);
    box-shadow: 0 0 10px rgba(0, 255, 255, 0.5);
  }

  .connect-btn.connected {
    background: rgba(0, 255, 255, 0.3);
    border-color: #00ff00;
  }

  .connect-btn:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }

  .error-message {
    color: #ff3366;
    margin-top: 10px;
    font-size: 14px;
    animation: shake 0.5s ease-in-out;
  }

  .success-message {
    color: #00ff66;
    margin-top: 10px;
    font-size: 14px;
    animation: fadeIn 0.5s ease-in-out;
  }

  @keyframes shake {
    0%,
    100% {
      transform: translateX(0);
    }
    20%,
    60% {
      transform: translateX(-5px);
    }
    40%,
    80% {
      transform: translateX(5px);
    }
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
</style>
