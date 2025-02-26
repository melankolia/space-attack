<template>
  <div class="about">
    <h2>About Endless Road Runner</h2>
    <p>
      This is a simple endless runner game built with Vue.js. Try to avoid the
      obstacles and survive as long as you can! The game gets progressively
      harder as your score increases.
    </p>
    <p>Use the up and down arrow keys to control your car.</p>
  </div>
</template>

<style>
  @media (min-width: 1024px) {
    .about {
      min-height: 100vh;
      display: flex;
      align-items: center;
    }
  }
</style>

<style scoped>
  .about {
    max-width: 600px;
    margin: 0 auto;
    padding: 2rem;
    text-align: center;
  }
</style>
