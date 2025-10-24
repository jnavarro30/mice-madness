<script setup>
const emit = defineEmits(['start-1p', 'start-2p', 'how-to', 'settings'])
</script>

<template>
  <div class="start-menu-backdrop">
    <div class="start-menu">
      <h1 class="title">🐭 Rodents' Revenge</h1>
      <button class="menu-btn primary" @click="emit('start-1p')">1 Player</button>
      <button class="menu-btn" @click="emit('start-2p')" disabled>2 Players (WIP)</button>
      <button class="menu-btn" @click="emit('how-to')">How to Play</button>
      <button class="menu-btn" @click="emit('settings')">Settings</button>
    </div>
  </div>
</template>

<style scoped>
.start-menu-backdrop {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.8);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.start-menu {
  background: #111;
  color: #fff;
  padding: 32px 28px;
  border-radius: 12px;
  width: 320px;
  display: flex;
  flex-direction: column;
  gap: 12px;
  box-shadow: 0 10px 40px rgba(0,0,0,0.5);
  border: 2px solid #333;
}

.title {
  text-align: center;
  margin-bottom: 8px;
}

.menu-btn {
  background: #2b2b2b;
  color: #fff;
  border: 1px solid #444;
  padding: 10px 14px;
  font-size: 16px;
  border-radius: 8px;
  cursor: pointer;
  transition: background 0.2s, transform 0.05s;
}
.menu-btn:hover { background: #353535; }
.menu-btn:active { transform: translateY(1px); }
.menu-btn.primary { background: #667eea; border-color: #5568d3; }
.menu-btn.primary:hover { background: #5568d3; }

button[disabled] { opacity: 0.6; cursor: not-allowed; }
</style>
