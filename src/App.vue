<script setup>
  import { RouterLink, RouterView } from "vue-router";
</script>

<template>
  <main>
    <!-- Future leaderboard will go here -->
    <div class="game-container">
      <RouterView />
    </div>
  </main>
</template>

<style scoped>
  header {
    text-align: center;
    padding: 1rem;
    margin-bottom: 2rem;
  }

  nav {
    margin: 1rem 0;
  }

  nav a {
    margin: 0 1rem;
    text-decoration: none;
    color: #42b883;
  }

  main {
    display: flex;
    justify-content: flex-start;
    align-items: flex-start;
    min-height: calc(100vh - 150px);
    padding: 0 2rem;
  }

  .game-container {
    margin-left: 300px;
    flex: 1;
    display: flex;
    justify-content: center;
  }
</style>
