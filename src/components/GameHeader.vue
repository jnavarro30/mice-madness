<template>
  <div class="game-header">
    <div class="stat">
      <div>Score</div>
      <div class="stat-value">{{ score }}</div>
    </div>
    <div class="stat">
      <div>Level</div>
      <div class="stat-value">{{ level }}</div>
    </div>
    <div class="stat">
      <div>Trapped Cats</div>
      <div class="stat-value">{{ trappedCats }} / {{ totalCats }}</div>
    </div>
  </div>
</template>

<script setup>
const props = defineProps({
  score: { type: Number, required: true },
  level: { type: Number, required: true },
  trappedCats: { type: Number, required: true },
  totalCats: { type: Number, required: true },
})
</script>

<style scoped>
.game-header {
  display: flex;
  justify-content: space-around;
  margin-bottom: 20px;
  gap: 20px;
}

.stat {
  background: #f0f0f0;
  padding: 10px 20px;
  border-radius: 8px;
  font-weight: bold;
}

.stat-value {
  color: #667eea;
  font-size: 24px;
}
</style>
